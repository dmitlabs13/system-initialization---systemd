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

#в отличие от методички, помимо start пришлость еще сделдать
sudo systemctl enable monitoring_log.timer
sudo systemctl restart monitoring_log.timer #как-то только после этого заработал таймер 

#проверяем
sadmin@lp-ubn1:~$ sudo systemctl status monitoring_log.service
○ monitoring_log.service - Monitoring log service
     Loaded: loaded (/etc/systemd/system/monitoring_log.service; static)
     Active: inactive (dead) since Wed 2026-05-13 11:48:19 UTC; 16s ago
TriggeredBy: ● monitoring_log.timer
    Process: 45492 ExecStart=/opt/monitoring_log.sh (code=exited, status=0/SUCCESS)
   Main PID: 45492 (code=exited, status=0/SUCCESS)
        CPU: 4ms

May 13 11:48:19 lp-ubn1 systemd[1]: Starting monitoring_log.service - Monitoring log service...
May 13 11:48:19 lp-ubn1 systemd[1]: monitoring_log.service: Deactivated successfully.
May 13 11:48:19 lp-ubn1 systemd[1]: Finished monitoring_log.service - Monitoring log service.
sadmin@lp-ubn1:~$ sudo systemctl status monitoring_log.timer
● monitoring_log.timer - Run monitoring_log.service every 30sec
     Loaded: loaded (/etc/systemd/system/monitoring_log.timer; enabled; preset: enabled)
     Active: active (waiting) since Wed 2026-05-13 11:43:25 UTC; 5min ago
    Trigger: Wed 2026-05-13 11:49:29 UTC; 28s left
   Triggers: ● monitoring_log.service

May 13 11:43:25 lp-ubn1 systemd[1]: monitoring_log.timer: Deactivated successfully.
May 13 11:43:25 lp-ubn1 systemd[1]: Stopped monitoring_log.timer - Run monitoring_log.service ever>
May 13 11:43:25 lp-ubn1 systemd[1]: Stopping monitoring_log.timer - Run monitoring_log.service eve>
May 13 11:43:25 lp-ubn1 systemd[1]: Started monitoring_log.timer - Run monitoring_log.service ever>

#в нашем файле есть записи о попытке подбора пароля
sadmin@lp-ubn1:~$ cat /opt/monitoring.txt 
2026-05-13T11:16:28.836339+00:00 lp-ubn1 sshd[44860]: Failed password for invalid user user from 192.168.20.78 port 60086 ssh2
2026-05-13T11:45:07.617781+00:00 lp-ubn1 sshd[45462]: Failed password for invalid user user from 192.168.20.78 port 60152 ssh2
2026-05-13T11:45:07.617781+00:00 lp-ubn1 sshd[45462]: Failed password for invalid user user from 192.168.20.78 port 60152 ssh2


```


### Установить spawn-fcgi и создать unit-файл (spawn-fcgi.sevice) с помощью переделки init-скрипта (https://gist.github.com/cea2k/1318020).

```
#Устанавливаем spawn-fcgi и необходимые для него пакеты:
sudo apt install spawn-fcgi php php-cgi php-cli apache2 libapache2-mod-fcgid -y
#создаем fcgi.conf
sadmin@lp-ubn1:~$ sudo mkdir /etc/spawn-fcgi
sadmin@lp-ubn1:~$ sudo nano /etc/spawn-fcgi/fcgi.conf
#содержимое файла
# You must set some working options before the "spawn-fcgi" service
will work.
# If SOCKET points to a file, then this file is cleaned up by the
init script.
#
# See spawn-fcgi(1) for all possible options.
#
# Example :
SOCKET=/var/run/php-fcgi.sock
OPTIONS=
"
-u www-data -g www-data -s $SOCKET -S -M 0600 -C 32 -F 1 --
/usr/bin/php-cgi
#####

#Создаем файл юнита
sadmin@lp-ubn1:~$ sudo nano /etc/systemd/system/spawn-fcgi.service
#содержимое файла
[Unit]
Description=Spawn-fcgi startup service by Otus
After=network.target
[Service]
Type=simple
PIDFile=/var/run/spawn-fcgi.pid
EnvironmentFile=/etc/spawn-fcgi/fcgi.conf
ExecStart=/usr/bin/spawn-fcgi -n $OPTIONS
KillMode=process
[Install]
WantedBy=multi-user.target
##############################
sadmin@lp-ubn1:~$ sudo systemctl status spawn-fcgi.service 
● spawn-fcgi.service - Spawn-fcgi startup service by Otus
     Loaded: loaded (/etc/systemd/system/spawn-fcgi.service; disabled; preset: enabled)
     Active: active (running) since Wed 2026-05-13 12:22:28 UTC; 12s ago
   Main PID: 54852 (php-cgi)
      Tasks: 33 (limit: 1055)
     Memory: 14.3M (peak: 14.5M)
        CPU: 24ms
     CGroup: /system.slice/spawn-fcgi.service
             ├─54852 /usr/bin/php-cgi
             ├─54853 /usr/bin/php-cgi
             ├─54854 /usr/bin/php-cgi
             ├─54855 /usr/bin/php-cgi
             ├─54856 /usr/bin/php-cgi
             ├─54857 /usr/bin/php-cgi
             ├─54858 /usr/bin/php-cgi
             ├─54859 /usr/bin/php-cgi
             ├─54860 /usr/bin/php-cgi
             ├─54861 /usr/bin/php-cgi
             ├─54862 /usr/bin/php-cgi
             ├─54863 /usr/bin/php-cgi
             ├─54864 /usr/bin/php-cgi
             ├─54865 /usr/bin/php-cgi
             ├─54866 /usr/bin/php-cgi
             ├─54867 /usr/bin/php-cgi
             ├─54868 /usr/bin/php-cgi
             ├─54869 /usr/bin/php-cgi
             ├─54870 /usr/bin/php-cgi
             ├─54871 /usr/bin/php-cgi
             ├─54872 /usr/bin/php-cgi
             ├─54873 /usr/bin/php-cgi
             ├─54874 /usr/bin/php-cgi
             ├─54875 /usr/bin/php-cgi
             ├─54876 /usr/bin/php-cgi
             ├─54877 /usr/bin/php-cgi
             ├─54878 /usr/bin/php-cgi
             ├─54879 /usr/bin/php-cgi
             ├─54880 /usr/bin/php-cgi
             ├─54881 /usr/bin/php-cgi
             ├─54882 /usr/bin/php-cgi
sadmin@lp-ubn1:~$ 
```

###Доработать unit-файл Nginx (nginx.service) для запуска нескольких инстансов сервера с разными конфигурационными файлами одновременно.

```
sudo apt install nginx -y
#создаем новый юнит
sudo nano /etc/systemd/system/nginx@.service
# Stop dance for nginx
# =======================
#
# ExecStop sends SIGSTOP (graceful stop) to the nginx process.
# If, after 5s (--retry QUIT/5) nginx is still running, systemd
takes control
# and sends SIGTERM (fast shutdown) to the main process.
# After another 5s (TimeoutStopSec=5), and if nginx is alive,
systemd sends
# SIGKILL to all the remaining processes in the process group
(KillMode=mixed).
#
# nginx signals reference doc:
# http://nginx.org/en/docs/control.html
#
[Unit]
Description=A high performance web server and a reverse proxy server
Documentation=man:nginx(8)
After=network.target nss-lookup.target
[Service]
Type=forking
PIDFile=/run/nginx-%I.pid
ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx-%I.conf -q -g
'daemon on; master_process on;'
ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx-%I.conf -g 'daemon on;
master_process on;'
ExecReload=/usr/sbin/nginx -c /etc/nginx/nginx-%I.conf -g 'daemon
on; master_process on;'
-s reload
ExecStop=-/sbin/start-stop-daemon --quiet --stop --retry QUIT/5
--pidfile /run/nginx-%I.pid
TimeoutStopSec=5
KillMode=mixed
[Install]
WantedBy=multi-user.target


#создаем два файла###########################################################
sudo nano /etc/nginx/nginx-first.conf
# содержимое конфига
user www-data;
worker_processes auto;
pid /run/nginx-first.pid;                 
error_log /var/log/nginx/error.log;
include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections 768;
}

http {
    sendfile on;
    tcp_nopush on;
    types_hash_max_size 2048;
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    access_log /var/log/nginx/access.log;
    gzip on;

  
    server {
        listen 9001;                          
        server_name _;                        
        root /var/www/first_site;              
        index index.html index.htm index.php;

        location / {
            try_files $uri $uri/ =404;
        }

        # Подключаем PHP через наш spawn-fcgi
        location ~ \.php$ {
            include fastcgi_params;
            fastcgi_pass unix:/var/run/php-fcgi.sock;
            fastcgi_index index.php;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        }
    }
    # include /etc/nginx/conf.d/*.conf;
    # include /etc/nginx/sites-enabled/*;   
}



sudo nano /etc/nginx/nginx-second.conf
user www-data;
worker_processes auto;
pid /run/nginx-second.pid;                 
error_log /var/log/nginx/error.log;
include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections 768;
}

http {
    sendfile on;
    tcp_nopush on;
    types_hash_max_size 2048;
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
   ssl_prefer_server_ciphers on;

    access_log /var/log/nginx/access.log;
    gzip on;

  
    server {
        listen 9002;                          
        server_name _;                        
        root /var/www/second_site;              
        index index.html index.htm index.php;

        location / {
            try_files $uri $uri/ =404;
        }


        location ~ \.php$ {
            include fastcgi_params;
            fastcgi_pass unix:/var/run/php-fcgi.sock;
            fastcgi_index index.php;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        }
    }



    # include /etc/nginx/conf.d/*.conf;
    # include /etc/nginx/sites-enabled/*;   
}



#результат
                                                                      
sadmin@lp-ubn1:~$ sudo ss -tnulp | grep nginx
tcp   LISTEN 0      511                   0.0.0.0:9002       0.0.0.0:*    users:(("nginx",pid=55867,fd=5),("nginx",pid=55866,fd=5))                                                                                                                                           
tcp   LISTEN 0      511                   0.0.0.0:9001       0.0.0.0:*    users:(("nginx",pid=55836,fd=5),("nginx",pid=55834,fd=5))                                                                                                                                           
sadmin@lp-ubn1:~$ ps afx | grep nginx
  55898 pts/2    S+     0:00              \_ grep --color=auto nginx
  55834 ?        Ss     0:00 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx-first.conf -g daemon on; master_process on;
  55836 ?        S      0:00  \_ nginx: worker process
  55866 ?        Ss     0:00 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx-second.conf -g daemon on; master_process on;
  55867 ?        S      0:00  \_ nginx: worker process


```
