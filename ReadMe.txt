Домашнее задание

Написать сервис, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова.
Файл и слово должны задаваться в /etc/sysconfig

1.Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова 
(файл лога и ключевое слово должны задаваться в /etc/sysconfig или в /etc/default).

2.Установить spawn-fcgi и переписать init-скрипт на unit-файл (имя service должно называться так же: spawn-fcgi).

3.Дополнить unit-файл httpd (он же apache2) возможностью запустить несколько инстансов сервера с разными конфигурационными файлами.

##################################################################################################################################
1.Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова 


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




