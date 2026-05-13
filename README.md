# system-initialization---systemd

## Задание
- Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова (файл лога и ключевое слово должны задаваться в /etc/default).
- Установить spawn-fcgi и создать unit-файл (spawn-fcgi.sevice) с помощью переделки init-скрипта (https://gist.github.com/cea2k/1318020).
- Доработать unit-файл Nginx (nginx.service) для запуска нескольких инстансов сервера с разными конфигурационными файлами одновременно.

## Решение

### Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова (файл лога и ключевое слово должны задаваться в /etc/default).

```
# будем мониторить лог /var/log/auth.log
# ключевое слово будет "Failed password"

# создаем файл /etc/default/parametry.env

logpath="/var/log/auth.log"
keyword="Failed password"

#скрипт при запуске будет использовать параметры из файла /etc/default/parametry.env
#скрипт будет с помощью команды tail заглядывать в файл /var/log/auth.log
# текст скрипта:
#! /bin/bash/
#импорт переменных из файла /etc/default/parametry.env
source /etc/default/parametry.env
#команда мониторинга, берет логи за последние 30 секунд и если есть слова "Failed password" записывает строку в #файл monitoring.txt
tail $logpath |awk '$0 > strftime("%Y-%m-%d %H:%M:%S", systime()-30)'| grep --line-buffered "$keyword" >> /opt/monitoring.txt
exit 0

#создаем файл юнита
sudo bash -c 'cat > /etc/systemd/system/monitoring_log.service << EOF
[Unit]
Description=Monitoring log service

[Service]
Type=oneshot
ExecStart=/opt/monitoring_log.sh
EOF'



#создаем файл таймера
sudo nano /etc/systemd/system/monitoring_log.timer
#с таким содержимым
[Unit]
Description=Run monitoring_log.service every 30sec

[Timer]
OnUnitActiveSec=30
Unit=monitoring_log.service

[Install]
WantedBy=multi-user.target





```


### Установить spawn-fcgi и создать unit-файл (spawn-fcgi.sevice) с помощью переделки init-скрипта (https://gist.github.com/cea2k/1318020).

```

```
