Домашнее задание "Инициализация системы. Systemd"

1. Создаем виртуальную машину

- Создаем Vagrantfile с учетом требований;
- Включаем вм - vagrant up;
- Подключаемся к вм - vagrant ssh.

2. Написать сервис, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова.
Файл и слово должны задаваться в /etc/sysconfig:

- Создадим файл с конфигурацией для сервиса в директории /etc/sysconfig,
  затем создаем /var/log/watchlog.log и пишем туда строки на своё усмотрение, плюс ключевое слово 'ALERT':
~~~
[root@SystemD ~]# cd /etc/sysconfig
[root@SystemD sysconfig]# touch watchlog
[root@SystemD sysconfig]# vi watchlog
[root@SystemD sysconfig]# cat watchlog
# Configuration file for my watchlog service
# Place it to /etc/sysconfig

# File and word in that file that we will be monitoring
WORD="ALERT"
LOG=/var/log/watchlog.log

[root@SystemD sysconfig]# touch /var/log/watchlog.log
~~~

- Создадим скрипт, назначим права:
~~~
[root@SystemD sysconfig]# touch /opt/watchlog.sh
[root@SystemD /]# vi /opt/watchlog.sh
[root@SystemD /]# cat /opt/watchlog.sh
#!/bin/bash

WORD=$1
LOG=$2
DATE=`date`

if grep $WORD $LOG &> /dev/null
then
logger "$DATE: I found word, Master!"
else
exit 0
fi

[root@SystemD /]# chmod +x /opt/watchlog.sh
~~~

- Создадим юнит для сервиса:
~~~
[root@SystemD /]# touch /etc/systemd/system/watchlog.service
[root@SystemD /]# vi /etc/systemd/system/watchlog.service
[root@SystemD /]# cat /etc/systemd/system/watchlog.service
[Unit]
Description=My watchlog service

[Service]
Type=oneshot
EnvironmentFile=/etc/sysconfig/watchlog
ExecStart=/opt/watchlog.sh $WORD $LOG
~~~

- Создадим юнит для таймера:
~~~
[root@SystemD /]# cat /etc/systemd/system/watchlog.timer
[Unit]
Description=Run watchlog script every 30 second

[Timer]
# Run every 30 second

OnActiveSec=1sec
#OnUnitActiveSec=30
OnCalendar=*:*:0/30
AccuracySec=1us

Unit=watchlog.service

[Install]
WantedBy=multi-user.target
~~~

- Включим и запустим таймер:
~~~
[root@SystemD /]# systemctl enable watchlog.timer
Created symlink /etc/systemd/system/multi-user.target.wants/watchlog.timer → /etc/systemd/system/watchlog.timer.
[root@SystemD /]# systemctl start watchlog.timer
~~~

- Проверим результат:
~~~
[root@SystemD /]# tail -f /var/log/messages
Jun  1 09:07:00 localhost systemd[1]: watchlog.service: Succeeded.
Jun  1 09:07:00 localhost systemd[1]: Started My watchlog service.
Jun  1 09:07:30 localhost systemd[1]: Starting My watchlog service...
Jun  1 09:07:30 localhost root[2993]: Thu Jun  1 09:07:30 UTC 2023: I found word, Master!
Jun  1 09:07:30 localhost systemd[1]: watchlog.service: Succeeded.
Jun  1 09:07:30 localhost systemd[1]: Started My watchlog service.
Jun  1 09:08:00 localhost systemd[1]: Starting My watchlog service...
Jun  1 09:08:00 localhost root[2998]: Thu Jun  1 09:08:00 UTC 2023: I found word, Master!
Jun  1 09:08:00 localhost systemd[1]: watchlog.service: Succeeded.
Jun  1 09:08:00 localhost systemd[1]: Started My watchlog service.
Jun  1 09:08:30 localhost systemd[1]: Starting My watchlog service...
Jun  1 09:08:30 localhost root[3004]: Thu Jun  1 09:08:30 UTC 2023: I found word, Master!
Jun  1 09:08:30 localhost systemd[1]: watchlog.service: Succeeded.
~~~

2. Из epel установить spawn-fcgi и переписать init-скрипт на unit-файл. Имя сервиса должно также называться.

- Устанавливаем spawn-fcgi и необходимые для него пакеты:
~~~
[root@SystemD /]# yum install epel-release -y && yum install spawn-fcgi php php-cli mod_fcgid httpd -y
Failed to set locale, defaulting to C.UTF-8
CentOS Linux 8 - AppStream                                                                                                                                                                                   4.8 MB/s | 8.4 MB     00:01
CentOS Linux 8 - BaseOS                                                                                                                                                                                      4.0 MB/s | 4.6 MB     00:01
CentOS Linux 8 - Extras                                                                                                                                                                                       25 kB/s |  10 kB     00:00
Dependencies resolved.
=============================================================================================================================================================================================================================================
 Package                                                      Architecture                                           Version                                                    Repository                                              Size
=============================================================================================================================================================================================================================================
Installing:
 epel-release                                                 noarch                                                 8-11.el8                                                   extras                                                  24 k

Transaction Summary
=============================================================================================================================================================================================================================================
Install  1 Package
...
Installed:
  apr-1.6.3-12.el8.x86_64                           apr-util-1.6.1-6.el8.x86_64                           apr-util-bdb-1.6.1-6.el8.x86_64                                  apr-util-openssl-1.6.1-6.el8.x86_64
  centos-logos-httpd-85.8-2.el8.noarch              httpd-2.4.37-43.module_el8.5.0+1022+b541f3b1.x86_64   httpd-filesystem-2.4.37-43.module_el8.5.0+1022+b541f3b1.noarch   httpd-tools-2.4.37-43.module_el8.5.0+1022+b541f3b1.x86_64
  mailcap-2.1.48-3.el8.noarch                       mod_fcgid-2.3.9-17.el8.x86_64                         mod_http2-1.15.7-3.module_el8.4.0+778+c970deab.x86_64            nginx-filesystem-1:1.14.1-9.module_el8.0.0+184+e34fea82.noarch
  php-7.2.24-1.module_el8.2.0+313+b04d0a66.x86_64   php-cli-7.2.24-1.module_el8.2.0+313+b04d0a66.x86_64   php-common-7.2.24-1.module_el8.2.0+313+b04d0a66.x86_64           php-fpm-7.2.24-1.module_el8.2.0+313+b04d0a66.x86_64
  spawn-fcgi-1.6.3-17.el8.x86_64

Complete!
~~~

- Раскомментируем и отредактируем строки с переменными в /etc/sysconfig/spawn-fcgi:
~~~
[root@SystemD /]# cat /etc/sysconfig/spawn-fcgi
# You must set some working options before the "spawn-fcgi" service will work.
# If SOCKET points to a file, then this file is cleaned up by the init script.
#
# See spawn-fcgi(1) for all possible options.
#
# Example :
SOCKET=/var/run/php-fcgi.sock
OPTIONS="-u apache -g apache -s $SOCKET -S -M 0600 -C 32 -F 1 -- /usr/bin/php-cgi"
~~~

- Содержимое юнит файла:
~~~
[root@SystemD /]# touch /etc/systemd/system/spawn-fcgi.service
[root@SystemD /]# vi /etc/systemd/system/spawn-fcgi.service
[root@SystemD /]# cat /etc/systemd/system/spawn-fcgi.service
[Unit]
Description=Spawn-fcgi startup service by Bye-Bye
After=network.target

[Service]
Type=simple
PIDFile=/var/run/spawn-fcgi.pid
EnvironmentFile=/etc/sysconfig/spawn-fcgi
ExecStart=/usr/bin/spawn-fcgi -n $OPTIONS
KillMode=process

[Install]
WantedBy=multi-user.target
~~~

- Проверяем работу:
~~~
[root@SystemD /]# systemctl start spawn-fcgi
[root@SystemD /]# systemctl status spawn-fcgi
● spawn-fcgi.service - Spawn-fcgi startup service by Bye-Bye
   Loaded: loaded (/etc/systemd/system/spawn-fcgi.service; disabled; vendor preset: disabled)
   Active: active (running) since Thu 2023-06-01 09:39:05 UTC; 6s ago
 Main PID: 22522 (php-cgi)
    Tasks: 33 (limit: 12421)
   Memory: 18.2M
   CGroup: /system.slice/spawn-fcgi.service
           ├─22522 /usr/bin/php-cgi
           ├─22523 /usr/bin/php-cgi
           ├─22524 /usr/bin/php-cgi
           ├─22525 /usr/bin/php-cgi
           ├─22526 /usr/bin/php-cgi
           ├─22527 /usr/bin/php-cgi
           ├─22528 /usr/bin/php-cgi
           ├─22529 /usr/bin/php-cgi
           ├─22530 /usr/bin/php-cgi
           ├─22531 /usr/bin/php-cgi
           ├─22532 /usr/bin/php-cgi
           ├─22533 /usr/bin/php-cgi
           ├─22534 /usr/bin/php-cgi
           ├─22535 /usr/bin/php-cgi
           ├─22536 /usr/bin/php-cgi
           ├─22537 /usr/bin/php-cgi
           ├─22538 /usr/bin/php-cgi
           ├─22539 /usr/bin/php-cgi
           ├─22540 /usr/bin/php-cgi
           ├─22541 /usr/bin/php-cgi
           ├─22542 /usr/bin/php-cgi
           ├─22543 /usr/bin/php-cgi
           ├─22544 /usr/bin/php-cgi
           ├─22545 /usr/bin/php-cgi
           ├─22546 /usr/bin/php-cgi
           ├─22547 /usr/bin/php-cgi
           ├─22548 /usr/bin/php-cgi
           ├─22549 /usr/bin/php-cgi
           ├─22550 /usr/bin/php-cgi
           ├─22551 /usr/bin/php-cgi
           ├─22552 /usr/bin/php-cgi
           ├─22553 /usr/bin/php-cgi
           └─22554 /usr/bin/php-cgi

Jun 01 09:39:05 SystemD systemd[1]: Started Spawn-fcgi startup service by Bye-Bye.
~~~

4. Дополнить юнит-файл apache httpd возможностью запустить несколько инстансов сервера с разными конфигами.

- Скопировать шаблон конфигурации файла окружения из /usr/lib/systemd/system/httpd.service:
~~~
[root@SystemD conf]# cp /usr/lib/systemd/system/httpd.service /etc/systemd/system/httpd@.service

[root@SystemD conf]# cat /etc/systemd/system/httpd@.service
# See httpd.service(8) for more information on using the httpd service.

# Modifying this file in-place is not recommended, because changes
# will be overwritten during package upgrades.  To customize the
# behaviour, run "systemctl edit httpd" to create an override unit.

# For example, to pass additional options (such as -D definitions) to
# the httpd binary at startup, create an override unit (as is done by
# systemctl edit) and enter the following:

#       [Service]
#       Environment=OPTIONS=-DMY_DEFINE

[Unit]
Description=The Apache HTTP Server
Wants=httpd-init.service
After=network.target remote-fs.target nss-lookup.target httpd-init.service
Documentation=man:httpd.service(8)

[Service]
Type=notify
Environment=LANG=C
EnvironmentFile=/etc/sysconfig/httpd-%I
ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND
ExecReload=/usr/sbin/httpd $OPTIONS -k graceful
# Send SIGWINCH for graceful stop
KillSignal=SIGWINCH
KillMode=mixed
PrivateTmp=true


[Install]
WantedBy=multi-user.target
~~~

- Копирую файлы конфигурации:
~~~
[root@SystemD conf]# cp httpd.conf first.conf
[root@SystemD conf]# cp httpd.conf second.conf
~~~

- В файле second.conf добавляю:
~~~
PidFile /var/run/httpd-second.pid
Listen 8080
~~~

- Создаем файлы с опциями для запуска веб сервера:
~~~
[root@SystemD conf]# touch /etc/sysconfig/httpd-first
[root@SystemD conf]# vi /etc/sysconfig/httpd-first
[root@SystemD conf]# cat /etc/sysconfig/httpd-first
OPTIONS=-f conf/first.conf

[root@SystemD conf]# touch /etc/sysconfig/httpd-second
[root@SystemD conf]# vi /etc/sysconfig/httpd-second
[root@SystemD conf]# cat /etc/sysconfig/httpd-second
OPTIONS=-f conf/second.conf
~~~

- Запускаем:
~~~
[root@SystemD conf]# systemctl enable  httpd@first
Created symlink /etc/systemd/system/multi-user.target.wants/httpd@first.service → /etc/systemd/system/httpd@.service.
[root@SystemD conf]# systemctl enable  httpd@second
Created symlink /etc/systemd/system/multi-user.target.wants/httpd@second.service → /etc/systemd/system/httpd@.service.

[root@SystemD conf]# systemctl start  httpd@first
[root@SystemD conf]# systemctl start  httpd@second
~~~

- Проверяем:
~~~
[root@SystemD conf]# ss -tnulp | grep httpd
tcp     LISTEN   0        128                    *:8080                *:*       users:(("httpd",pid=25249,fd=4),("httpd",pid=25248,fd=4),("httpd",pid=25247,fd=4),("httpd",pid=25246,fd=4),("httpd",pid=25244,fd=4))
tcp     LISTEN   0        128                    *:80                  *:*       users:(("httpd",pid=25030,fd=4),("httpd",pid=25029,fd=4),("httpd",pid=25028,fd=4),("httpd",pid=25027,fd=4),("httpd",pid=25025,fd=4))
~~~









