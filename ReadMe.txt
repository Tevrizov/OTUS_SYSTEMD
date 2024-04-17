Домашнее задание

Написать сервис, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова.
Файл и слово должны задаваться в /etc/sysconfig

1.Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова 
(файл лога и ключевое слово должны задаваться в /etc/sysconfig или в /etc/default).

2.Установить spawn-fcgi и переписать init-скрипт на unit-файл (имя service должно называться так же: spawn-fcgi).

3.Дополнить unit-файл httpd (он же apache2) возможностью запустить несколько инстансов сервера с разными конфигурационными файлами.


Если описать systemd в общих чертах, то это система управления юнитами. 
Юнитами могут быть много вещей. Одним из наиболее важных типов юнитов является сервис.
Как правило, сервисы - это процессы, которые предоставляют определенные функциональные возможности и позволяют подключаться внешним клиентам.
Помимо сервисов, существуют другие типы юнитов, такие как сокеты, монтирование и другие.

Чтобы отобразить список всех доступных юнитов, введите systemctl -t help

Основное преимущество работы с systemd по сравнению с предыдущими методами, используемыми для управления сервисами, 
заключается в том, что он обеспечивает единый интерфейс для запуска юнитов. Этот интерфейс определен в файле юнита.

Сокет создает метод для приложений, чтобы общаться друг с другом. Некоторые сервисы создают свои собственные сокеты при запуске, 
в то время как другим сервисам нужен файл-юнит сокетов для создания сокетов для них. И наоборот: каждому сокету нужен соответствующий служебный файл.


##################################################################################################################################
1.Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова 


Исрользую тестовую ВМ на базе Ubuntu 20.04.6 LTS

1. Создаём файл с конфигурацией для сервиса в директории /etc/sysconfig - из неё сервис будет брать необходимые переменные.
    root@tep:/etc/systemd# cat watchlog.conf 
    # Configuration file for my watchlog service
    # Place it to /etc/systemd

    # File and word in that file that we will be monit
    WORD="ALERT"
    LOG=/var/log/watchlog.log
    root@tep:/etc/systemd# 

2.Создадим скрипт:

    root@tep:/opt# cat watchlog.sh 
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

root@tep:/opt# 

3. root@tep:/opt# chmod +x watchlog.sh 

4. Создаю юнит для сервиса
   
   cd /etc/systemd/system && touch watchlog.daemon.timer && touch watchlog.daemon.service

    root@tep:/etc/systemd/system# cat watchlog.daemon.service 
        [Unit]
        Description=My watchlog service

        [Service]
        Type=oneshot
        EnvironmentFile=/etc/systemd/watchlog.conf
        ExecStart=/opt/watchlog.sh $WORD $LOG
    root@tep:/etc/systemd/system# 

    
    
    root@tep:/etc/systemd/system# cat watchlog.daemon.timer 
        [Unit]
        Description=Run watchlog script every 30 second

        [Timer]
        # Run every 30 second
        OnUnitActiveSec=30
        Unit=watchlog.daemon.service

        [Install]
        WantedBy=multi-user.target
    root@tep:/etc/systemd/system# 



5. Запускаем сервис 
    root@tep:/etc/systemd/system# systemctl daemon-reload
    root@tep:/etc/systemd/system# systemctl start watchlog.daemon.timer


6. Проверяем работу
    root@tep:/etc/systemd/system# systemctl status watchlog.daemon.timer
    ● watchlog.daemon.timer - Run watchlog script every 30 second
     Loaded: loaded (/etc/systemd/system/watchlog.daemon.timer; disabled; vendor preset: enabled)
     Active: active (running) since Tue 2024-04-16 09:38:00 UTC; 10min ago
    Trigger: n/a
   Triggers: ● watchlog.daemon.service

        апр 16 09:38:00 tep systemd[1]: watchlog.daemon.timer: Succeeded.
        апр 16 09:38:00 tep systemd[1]: Stopped Run watchlog script every 30 second.
        апр 16 09:38:00 tep systemd[1]: Stopping Run watchlog script every 30 second.
        апр 16 09:38:00 tep systemd[1]: Started Run watchlog script every 30 second.



    root@tep:/etc/systemd/system# systemctl status watchlog.daemon
    ● watchlog.daemon.service - My watchlog service
     Loaded: loaded (/etc/systemd/system/watchlog.daemon.service; static; vendor preset: enabled)
     Active: inactive (dead) since Tue 2024-04-16 09:49:50 UTC; 47ms ago
    TriggeredBy: ● watchlog.daemon.timer
    Process: 3560 ExecStart=/opt/watchlog.sh $WORD $LOG (code=exited, status=0/SUCCESS)
    Main PID: 3560 (code=exited, status=0/SUCCESS)

    апр 16 09:49:50 tep systemd[1]: Starting My watchlog service...
    апр 16 09:49:50 tep watchlog.sh[3560]: 123
    апр 16 09:49:50 tep root[3565]: Вт 16 апр 2024 09:49:50 UTC: I found word, Master!
    апр 16 09:49:50 tep systemd[1]: watchlog.daemon.service: Succeeded.
    апр 16 09:49:50 tep systemd[1]: Finished My watchlog service.
    root@tep:/etc/systemd/system# 


root@tep:~# systemctl list-timers
NEXT                        LEFT        LAST                        PASSED        UNIT                         ACTIVATES                     
Tue 2024-04-16 12:42:02 UTC 8s left     Tue 2024-04-16 12:41:32 UTC 21s ago       watchlog.daemon.timer        watchlog.daemon.service       
Tue 2024-04-16 18:45:59 UTC 6h left     Tue 2024-04-16 11:26:41 UTC 1h 15min ago  motd-news.timer              motd-news.service             
Tue 2024-04-16 23:59:55 UTC 11h left    Tue 2024-04-16 11:26:41 UTC 1h 15min ago  apt-daily.timer              apt-daily.service             
Wed 2024-04-17 00:00:00 UTC 11h left    Tue 2024-04-16 07:55:56 UTC 4h 45min ago  logrotate.timer              logrotate.service             
Wed 2024-04-17 00:00:00 UTC 11h left    Tue 2024-04-16 07:55:56 UTC 4h 45min ago  man-db.timer                 man-db.service                
Wed 2024-04-17 04:15:05 UTC 15h left    Tue 2024-04-16 11:26:41 UTC 1h 15min ago  fwupd-refresh.timer          fwupd-refresh.service         
Wed 2024-04-17 06:56:36 UTC 18h left    Tue 2024-04-16 08:07:50 UTC 4h 34min ago  apt-daily-upgrade.timer      apt-daily-upgrade.service     
Wed 2024-04-17 09:27:13 UTC 20h left    Tue 2024-04-16 08:11:31 UTC 4h 30min ago  systemd-tmpfiles-clean.timer systemd-tmpfiles-clean.service
Sun 2024-04-21 03:10:36 UTC 4 days left Sun 2024-04-14 15:20:45 UTC 1 day 21h ago e2scrub_all.timer            e2scrub_all.service           
Mon 2024-04-22 00:00:00 UTC 5 days left Mon 2024-04-15 05:06:11 UTC 1 day 7h ago  fstrim.timer                 fstrim.service                

10 timers listed.
Pass --all to see loaded but inactive timers, too.
root@tep:~# 


################################################################################################################################################3

2.Установить spawn-fcgi и переписать init-скрипт на unit-файл (имя service должно называться так же: spawn-fcgi).

Исрользую тестовую ВМ на базе CentOS-Stream-8

1.Устанавливаем spawn-fcgi и необходимые для него пакеты:
    [root@localhost ~]# yum install epel-release -y && yum install spawn-fcgi php php-cli


2. Приводим файл /etc/sysconfig/spawn-fcgi к следующему виду

    [root@localhost ~]# cat /etc/sysconfig/spawn-fcgi
    # You must set some working options before the "spawn-fcgi" service will work.
    # If SOCKET points to a file, then this file is cleaned up by the init script.
    #
    # See spawn-fcgi(1) for all possible options.
    #
    # Example :
    SOCKET=/var/run/php-fcgi.sock
    OPTIONS="-u apache -g apache -s $SOCKET -S -M 0600 -C 32 -F 1 -P /var/run/spawn-fcgi.pid -- /usr/bin/php-cgi"

    [root@localhost ~]#


3. Приводим файл /etc/systemd/system/spawn-fcgi.service к следующему виду

    [root@localhost ~]#  cat /etc/systemd/system/spawn-fcgi.service
    [Unit]
    Description=Spawn-fcgi startup service by Otus
    After=network.target

    [Service]
    Type=simple
    PIDFile=/var/run/spawn-fcgi.pid
    EnvironmentFile=/etc/sysconfig/spawn-fcgi
    ExecStart=/usr/bin/spawn-fcgi -n $OPTIONS
    KillMode=process

    [Install]
    WantedBy=multi-user.target
    [root@localhost ~]#

4. Проверяю
[root@localhost ~]# systemctl status spawn-fcgi
● spawn-fcgi.service - Spawn-fcgi startup service by Otus
   Loaded: loaded (/etc/systemd/system/spawn-fcgi.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2024-04-17 11:07:53 EDT; 14min ago
 Main PID: 27063 (php-cgi)
    Tasks: 33 (limit: 4678)
   Memory: 29.8M
   CGroup: /system.slice/spawn-fcgi.service
           ├─27063 /usr/bin/php-cgi
           ├─27064 /usr/bin/php-cgi
           ├─27065 /usr/bin/php-cgi
           ├─27066 /usr/bin/php-cgi
           ├─27067 /usr/bin/php-cgi
           ├─27068 /usr/bin/php-cgi
           ├─27069 /usr/bin/php-cgi
           ├─27070 /usr/bin/php-cgi
           ├─27071 /usr/bin/php-cgi
           ├─27072 /usr/bin/php-cgi
           ├─27073 /usr/bin/php-cgi
           ├─27074 /usr/bin/php-cgi
           ├─27075 /usr/bin/php-cgi
           ├─27076 /usr/bin/php-cgi
           ├─27077 /usr/bin/php-cgi
           ├─27078 /usr/bin/php-cgi
           ├─27079 /usr/bin/php-cgi
           ├─27080 /usr/bin/php-cgi
           ├─27081 /usr/bin/php-cgi
           ├─27082 /usr/bin/php-cgi
           ├─27083 /usr/bin/php-cgi
           ├─27084 /usr/bin/php-cgi
           ├─27085 /usr/bin/php-cgi
           ├─27086 /usr/bin/php-cgi
           ├─27087 /usr/bin/php-cgi
           ├─27088 /usr/bin/php-cgi
           ├─27089 /usr/bin/php-cgi
           ├─27090 /usr/bin/php-cgi
           ├─27091 /usr/bin/php-cgi
           ├─27092 /usr/bin/php-cgi
           ├─27093 /usr/bin/php-cgi
           ├─27094 /usr/bin/php-cgi
           └─27095 /usr/bin/php-cgi

апр 17 11:07:53 localhost.localdomain systemd[1]: Started Spawn-fcgi startup service by Otus.
[root@localhost ~]#




###############################################################################################################################################

Дополнить юнит-файл apache httpd возможностью запустить несколько инстансов сервера с разными конфигами

1. Создаю два файла в /etc/sysconfig/
    touch httpd-first && touch httpd-second

2. Привожу эти два файла к виду:
    [root@localhost sysconfig]# cat httpd-first
    OPTIONS=-f conf/first.conf
    [root@localhost sysconfig]# cat httpd-second
    OPTIONS=-f conf/second.conf
    [root@localhost sysconfig]#

3. Создаю два файла конфигурационными в /etc/httpd/conf
    cat httpd.conf >  first.conf
    cat httpd.conf >  second.conf

4. Правлю second.conf
    Listen 8080
    в конец конфига добавляю PidFile /var/run/httpd-second.pid


5. Запускаю

[root@localhost ~]# systemctl status httpd@second
● httpd@second.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd@.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2024-04-17 11:38:31 EDT; 1min 1s ago
     Docs: man:httpd@.service(8)
  Process: 27485 ExecStartPre=/bin/chown root.apache /run/httpd/instance-second (code=exited, status=0/SUCCESS)
  Process: 27483 ExecStartPre=/bin/mkdir -m 710 -p /run/httpd/instance-second (code=exited, status=0/SUCCESS)
 Main PID: 27487 (httpd)
   Status: "Running, listening on: port 8080"
    Tasks: 213 (limit: 4678)
   Memory: 25.1M
   CGroup: /system.slice/system-httpd.slice/httpd@second.service
           ├─27487 /usr/sbin/httpd -DFOREGROUND -f conf/second.conf
           ├─27488 /usr/sbin/httpd -DFOREGROUND -f conf/second.conf
           ├─27489 /usr/sbin/httpd -DFOREGROUND -f conf/second.conf
           ├─27490 /usr/sbin/httpd -DFOREGROUND -f conf/second.conf
           └─27491 /usr/sbin/httpd -DFOREGROUND -f conf/second.conf

апр 17 11:38:30 localhost.localdomain systemd[1]: Starting The Apache HTTP Server...
апр 17 11:38:31 localhost.localdomain httpd[27487]: AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using localhost.localdomain. Set the 'ServerName' dir>
апр 17 11:38:31 localhost.localdomain systemd[1]: Started The Apache HTTP Server.
апр 17 11:38:31 localhost.localdomain httpd[27487]: Server configured, listening on: port 8080
lines 1-21/21 (END)

[root@localhost ~]# systemctl status httpd@firs
● httpd@firs.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd@.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2024-04-17 11:38:01 EDT; 1min 59s ago
     Docs: man:httpd@.service(8)
  Process: 27250 ExecStartPre=/bin/chown root.apache /run/httpd/instance-firs (code=exited, status=0/SUCCESS)
  Process: 27248 ExecStartPre=/bin/mkdir -m 710 -p /run/httpd/instance-firs (code=exited, status=0/SUCCESS)
 Main PID: 27251 (httpd)
   Status: "Running, listening on: port 80"
    Tasks: 213 (limit: 4678)
   Memory: 40.4M
   CGroup: /system.slice/system-httpd.slice/httpd@firs.service
           ├─27251 /usr/sbin/httpd -DFOREGROUND -f conf/firs.conf
           ├─27253 /usr/sbin/httpd -DFOREGROUND -f conf/firs.conf
           ├─27254 /usr/sbin/httpd -DFOREGROUND -f conf/firs.conf
           ├─27255 /usr/sbin/httpd -DFOREGROUND -f conf/firs.conf
           └─27256 /usr/sbin/httpd -DFOREGROUND -f conf/firs.conf

апр 17 11:38:01 localhost.localdomain systemd[1]: Starting The Apache HTTP Server...
апр 17 11:38:01 localhost.localdomain httpd[27251]: AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using localhost.localdomain. Set the 'ServerName' dir>
апр 17 11:38:01 localhost.localdomain systemd[1]: Started The Apache HTTP Server.
апр 17 11:38:01 localhost.localdomain httpd[27251]: Server configured, listening on: port 80



############################################################################################################################################################



