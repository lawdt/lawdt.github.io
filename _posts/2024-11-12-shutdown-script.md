---
title: Shutdown script as systemd service
classes: wide
tags: cloud gcp
---
Недавно у меня возникла задача запускать один скрипт при выключении сервера. А именно нужно было удалять сервер из Zabbix при Rollout Instance group в GCP. Само по себе решение такого мониторинга Instance группы с помоью стандартного zabbix linux шаблона очень спорное, однако в этой статье я хочу рассказать про запуск скрипта при выключении сервера.
Первой мыслью было сделать скрипт, например /root/script.sh, добавить на него права исполнения (что поначалу я забыл сделать и словил на этом неприятный косяк, не понимая почему сервис не запускается).
Важно заметить, что в начале скрипта нужно коректно указать shebang, иначе systemd откажется исполнять ваш скрипт.
#### скрипт /root/script.sh
```bash
#!/bin/bash
echo $(date) >> /var/log/script.log
```
дальше создаем файл с описанием systemd service, который будет запускаться при выключении, перезагрузке сервера.
#### /etc/systemd/system/my-shutdown-service.service
```bash
# my-shutdown-service.service

[Unit]
Description=Run some task at shutdown
After=syslog.service network.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStop=/root/script.sh
Restart=on-failure
RestartSec=1s

[Install]
WantedBy=multi-user.target
```
Пояснение:
	•	Type=oneshot означает, что команда выполняется один раз. Обычно для сервиса с типом oneshot он завершает свою работу после выполнения команды ExecStart, но так как я не хочу ничего делать при старте сервиса, команды ExecStart здесь нет.
	•	Я использую RemainAfterExit=yes, что позволяет сервису оставаться запущенным, даже если команда ExecStart отсутствует.
	•	ExecStop используется для выполнения команды при завершении работы.

Директива After=syslog.service network.target гарантирует, что сервис не будет запускаться до того, как заработает сервис syslog и сеть. Более того, так как systemd останавливает сервисы в обратном порядке их запуска, это также гарантирует, что syslog и сеть будут работать, когда будет выполняться команда ExecStop.

Хотя существует множество различных сервисов для работы с логами (syslog), большинство из них используют псевдоним syslog, поэтому After=syslog.service будет работать независимо от используемого сервиса (например, если используется rsyslog, это всё равно работает, так как rsyslog объявляет syslog как псевдоним).
#### Установка
Сперва я добавил startup-script, это [штатная возможность](https://cloud.google.com/compute/docs/instances/startup-scripts/linux) в GCP:
```bash
gsutil cp gs://my-project/my-bucket/script.sh /root/script.sh
chmod +x /root/script.sh
cat <<EOF > /etc/systemd/system/my-shutdown-service.service

# my-shutdown-service.service
[Unit]
Description=Run some task at shutdown
After=syslog.service network.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStop=/root/script.sh
Restart=on-failure
RestartSec=1s

[Install]
WantedBy=multi-user.target

# EOF
# systemctl daemon-reload
# systemctl enable --now remove_zabbix_host.service
```
И это замечательно работало. Однако потом я обнаружил что в GCP есть штатная возможность добавлять [shutdown-script](https://cloud.google.com/compute/docs/shutdownscript) прямо на этапе конфигурирования ВМ. Это тоже работает нормально, поскольку основано на acpi-сигналах.

#### P.S.
Я всячески не приветствую использование shutdown-scripts. Поскольку это очень тонкий момент. При выключении сервера время на выполнение скрипта ограничено. И еще, если при выполнении скрипта вам нужна сеть, даже если вы указали зависимость от network, не во всех дистрибутивах это будет работать. Но Если надо временно подложить соломку, то вполне сойдет.
