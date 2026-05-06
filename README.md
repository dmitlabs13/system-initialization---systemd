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



 tail -f /var/log/auth.log | grep --line-buffered "Failed" >> monitoring.txt


```


### Установить spawn-fcgi и создать unit-файл (spawn-fcgi.sevice) с помощью переделки init-скрипта (https://gist.github.com/cea2k/1318020).

```

```


### Установить spawn-fcgi и создать unit-файл (spawn-fcgi.sevice) с помощью переделки init-скрипта (https://gist.github.com/cea2k/1318020).

```

```
