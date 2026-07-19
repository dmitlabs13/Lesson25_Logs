# Lesson25_Logs

## Задание
Поднимаем две машины — web и log  

На web поднимаем nginx.  

Настраиваем центральный лог-сервер на любой системе по выбору:
journald;
rsyslog;
elk.
Настраиваем аудит, который будет отслеживать изменения конфигураций nginx.  

Все критичные логи с web должны собираться и локально и удаленно.  

Все логи с nginx должны уходить на удаленный сервер (локально только критичные).  

Логи аудита должны также уходить на удаленную систему.  



## Выполнение
Сервера:  
Сервер log - lp-ubn7-elk 192.168.50.226  
Сервер web - lp-ubn1 192.168.50.238  

сервер web остался с какой-то прошлой лабораторной, nginx уже стоит
```
root@lp-ubn1:/home/sadmin# systemctl status nginx
WARNING: terminal is not fully functional
Press RETURN to continue
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-07-13 18:07:15 UTC; 5 days ago
       Docs: man:nginx(8)
    Process: 2696 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_process on; (cod>
    Process: 2723 ExecStart=/usr/sbin/nginx -g daemon on; master_process on; (code=exited,>
   Main PID: 2724 (nginx)
      Tasks: 2 (limit: 1055)
     Memory: 1.5M (peak: 3.2M swap: 1.4M swap peak: 1.4M)
        CPU: 19ms
```
проверили, что rsyslog установлен  
```
root@lp-ubn1:/home/sadmin# systemctl status nginx
WARNING: terminal is not fully functional
Press RETURN to continue
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-07-13 18:07:15 UTC; 5 days ago
       Docs: man:nginx(8)
    Process: 2696 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_process on; (cod>
    Process: 2723 ExecStart=/usr/sbin/nginx -g daemon on; master_process on; (code=exited,>
   Main PID: 2724 (nginx)
      Tasks: 2 (limit: 1055)
     Memory: 1.5M (peak: 3.2M swap: 1.4M swap peak: 1.4M)
        CPU: 19ms

```

Меняем конфиг /etc/rsyslog.conf

```
# provides UDP syslog reception
module(load="imudp")
input(type="imudp" port="514")

# provides TCP syslog reception
module(load="imtcp")
input(type="imtcp" port="514")

```
создаем шаблон RemoteLogs 
```
root@lp-ubn7-elk:/home/sadmin# nano /etc/rsyslog.d/99-remote.conf
# с таким содержимым
$template RemoteLogs,"/var/log/rsyslog/%HOSTNAME%/%PROGRAMNAME%.log"
# Все логи пишем по шаблону
*.* ?RemoteLogs
# дальше ничего не обрабатываем
& ~

```

# проверяем конфиши и перезагружаем rsyslog

```
root@lp-ubn7-elk:/home/sadmin# sudo rsyslogd -N1
rsyslogd: version 8.2312.0, config validation run (level 1), master config /etc/rsyslog.conf
rsyslogd: End of config validation run. Bye.
root@lp-ubn7-elk:/home/sadmin# systemctl restart rsyslog
root@lp-ubn7-elk:/home/sadmin# systemctl status rsyslog
● rsyslog.service - System Logging Service
     Loaded: loaded (/usr/lib/systemd/system/rsyslog.service; enabled; preset: enabled)
     Active: active (running) since Sun 2026-07-19 15:53:01 UTC; 9s ago
TriggeredBy: ● syslog.socket
       Docs: man:rsyslogd(8)
             man:rsyslog.conf(5)
             https://www.rsyslog.com/doc/
    Process: 33783 ExecStartPre=/usr/lib/rsyslog/reload-apparmor-profile (code=exited, status=0/SUCCESS)
   Main PID: 33788 (rsyslogd)
      Tasks: 10 (limit: 2263)
     Memory: 1.8M (peak: 5.1M)
        CPU: 66ms
     CGroup: /system.slice/rsyslog.service
             └─33788 /usr/sbin/rsyslogd -n -iNONE

Jul 19 15:53:01 lp-ubn7-elk systemd[1]: Starting rsyslog.service - System Logging Service...
Jul 19 15:53:01 lp-ubn7-elk systemd[1]: Started rsyslog.service - System Logging Service.
Jul 19 15:53:01 lp-ubn7-elk rsyslogd[33788]: warning: ~ action is deprecated, consider using the 'stop' statement instead [v8.2312.0 try https://www.rsyslog.com/>
Jul 19 15:53:01 lp-ubn7-elk rsyslogd[33788]: imuxsock: Acquired UNIX socket '/run/systemd/journal/syslog' (fd 3) from systemd.  [v8.2312.0]
Jul 19 15:53:01 lp-ubn7-elk rsyslogd[33788]: rsyslogd's groupid changed to 104
Jul 19 15:53:01 lp-ubn7-elk rsyslogd[33788]: rsyslogd's userid changed to 103
Jul 19 15:53:01 lp-ubn7-elk rsyslogd[33788]: [origin software="rsyslogd" swVersion="8.2312.0" x-pid="33788" x-info="https://www.rsyslog.com"] start

#видим открыте порты 514
root@lp-ubn7-elk:/home/sadmin# ss -tuln
Netid State  Recv-Q Send-Q          Local Address:Port Peer Address:Port Process
udp   UNCONN 0      0                  127.0.0.54:53        0.0.0.0:*
udp   UNCONN 0      0               127.0.0.53%lo:53        0.0.0.0:*
udp   UNCONN 0      0      192.168.50.226%enp6s18:68        0.0.0.0:*
udp   UNCONN 0      0                     0.0.0.0:514       0.0.0.0:*
udp   UNCONN 0      0                        [::]:514          [::]:*
tcp   LISTEN 0      4096               127.0.0.54:53        0.0.0.0:*
tcp   LISTEN 0      4096                  0.0.0.0:22        0.0.0.0:*
tcp   LISTEN 0      4096            127.0.0.53%lo:53        0.0.0.0:*
tcp   LISTEN 0      25                    0.0.0.0:514       0.0.0.0:*
tcp   LISTEN 0      4096                     [::]:22           [::]:*
tcp   LISTEN 0      25                       [::]:514          [::]:*

```

Настраиваем nginx на отдачу логов
```
nano /etc/nginx/nginx.conf
# добавляем стоки. Строки добавляем в блок html, иначе конфиг считается не верным
error_log syslog:server=192.168.50.226,tag=nginx_error;
access_log syslog:server=192.168.50.226,tag=nginx_acces,severety=info combined;
```

Перезагружаем nginx, делаем обращение к сайту, проверяем, папка появилась
```
root@lp-ubn7-elk:/home/sadmin# ls -la /var/log/rsyslog/
total 16
drwxr-xr-x  4 syslog syslog 4096 Jul 19 16:58 .
drwxrwxr-x 11 root   syslog 4096 Jul 19 15:53 ..
drwxr-xr-x  2 syslog syslog 4096 Jul 19 16:58 lp-ubn1
drwxr-xr-x  2 syslog syslog 4096 Jul 19 16:12 lp-ubn7-elk
```

Собственно видим, что access логи пришли
```
-rw-r----- 1 syslog adm     215 Jul 19 16:58 nginx_error.log
root@lp-ubn7-elk:/home/sadmin# cat  /var/log/rsyslog/lp-ubn1/nginx_acces.log
2026-07-19T16:58:30+00:00 lp-ubn1 nginx_acces: ::1 - - [19/Jul/2026:16:58:30 +0000] "GET / HTTP/1.1" 403 162 "-" "curl/8.5.0"
```

Не очень понятно, тчо имеется ввиду в пункте задания: "Все критичные логи с web должны собираться и локально и удаленно."  

Будем исходить из того, что надо вообще все логи критичные логи с сервера lp-ubn1 должны отправлятся на удаленный rsyslog. все это не только nginx но и логи системы например.

```
#создаем файл /etc/rsyslog/
*.crit @@192.168.50.226

sudo rsyslogd -N1

systemctl restart rsyslog

# на web сервере отправлем тестовый критичный лог
root@lp-ubn1:/home/sadmin# sudo logger -p syslog.crit "TEST: Critical log from client"

# на сервере log видим полученоый лог
root@lp-ubn1:/home/sadmin# sudo logger -p syslog.crit "TEST: Critical log from client"

```

аудит  
```
# ставим утилиту
sudo apt update
sudo apt install auditd audispd-plugins -y
sudo systemctl enable --now auditd

# Создаем файл правил аудита
sudo nano /etc/audit/rules.d/nginx.rules

# будем мониторить основной конфиг Nginx
-w /etc/nginx/nginx.conf -p wa -k nginx_conf

# применяем
sudo auditctl -R /etc/audit/rules.d/nginx.rules

root@lp-ubn1:/home/sadmin# sudo touch /etc/nginx/nginx.conf

# видим логи
root@lp-ubn1:/home/sadmin# sudo ausearch -k nginx_conf
----
time->Sun Jul 19 18:15:37 2026
type=PROCTITLE msg=audit(1784484937.091:271): proctitle=617564697463746C002D52002F6574632F61756469742F72756C65732E642F6E67696E782E72756C6573
type=SYSCALL msg=audit(1784484937.091:271): arch=c000003e syscall=44 success=yes exit=1088 a0=3 a1=7fffa43f4bc0 a2=440 a3=0 items=0 ppid=26341 pid=26342 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts2 ses=1024 comm="auditctl" exe="/usr/sbin/auditctl" subj=unconfined key=(null)
type=CONFIG_CHANGE msg=audit(1784484937.091:271): auid=1000 ses=1024 subj=unconfined op=add_rule key="nginx_conf" list=4 res=1
----
time->Sun Jul 19 18:17:57 2026
type=PROCTITLE msg=audit(1784485077.362:292): proctitle=746F756368002F6574632F6E67696E782F6E67696E782E636F6E66
type=PATH msg=audit(1784485077.362:292): item=1 name="/etc/nginx/nginx.conf" inode=393992 dev=fc:00 mode=0100644 ouid=0 ogid=0 rdev=00:00 nametype=NORMAL cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=PATH msg=audit(1784485077.362:292): item=0 name="/etc/nginx/" inode=393983 dev=fc:00 mode=040755 ouid=0 ogid=0 rdev=00:00 nametype=PARENT cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=CWD msg=audit(1784485077.362:292): cwd="/home/sadmin"
type=SYSCALL msg=audit(1784485077.362:292): arch=c000003e syscall=257 success=yes exit=3 a0=ffffff9c a1=7ffe0bd007ac a2=941 a3=1b6 items=2 ppid=26357 pid=26358 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts2 ses=1024 comm="touch" exe="/usr/bin/touch" subj=unconfined key="nginx_conf"
```
Настраиваем отправку адит логов на LOG сервер
```
sudo nano /etc/rsyslog.d/10-audit.conf

#Содержимое
module(load="imfile" PollingInterval="10")

input(type="imfile"
      File="/var/log/audit/audit.log"
      Tag="auditd"
      Severity="info"
      Facility="local7")

# Отправляем на удаленный сервер
if $syslogtag contains 'auditd' then {
    action(type="omfwd" target="192.168.50.226" port="514" protocol="tcp")
    stop
}

в результате на ЛОГ сервере видим
root@lp-ubn7-elk:/home/sadmin# cat  /var/log/rsyslog/lp-ubn1/auditd.log | grep nginx
2026-07-19T18:43:47+00:00 lp-ubn1 auditd type=CONFIG_CHANGE msg=audit(1784484937.091:271): auid=1000 ses=1024 subj=unconfined op=add_rule key="nginx_conf" list=4 res=1#035AUID="sadmin"
2026-07-19T18:43:47+00:00 lp-ubn1 auditd type=SYSCALL msg=audit(1784485077.362:292): arch=c000003e syscall=257 success=yes exit=3 a0=ffffff9c a1=7ffe0bd007ac a2=941 a3=1b6 items=2 ppid=26357 pid=26358 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts2 ses=1024 comm="touch" exe="/usr/bin/touch" subj=unconfined key="nginx_conf"#035ARCH=x86_64 SYSCALL=openat AUID="sadmin" UID="root" GID="root" EUID="root" SUID="root" FSUID="root" EGID="root" SGID="root" FSGID="root"
2026-07-19T18:43:47+00:00 lp-ubn1 auditd type=PATH msg=audit(1784485077.362:292): item=0 name="/etc/nginx/" inode=393983 dev=fc:00 mode=040755 ouid=0 ogid=0 rdev=00:00 nametype=PARENT cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0#035OUID="root" OGID="root"
2026-07-19T18:43:47+00:00 lp-ubn1 auditd type=PATH msg=audit(1784485077.362:292): item=1 name="/etc/nginx/nginx.conf" inode=393992 dev=fc:00 mode=0100644 ouid=0 ogid=0 rdev=00:00 nametype=NORMAL cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0#035OUID="root" OGID="root"
2026-07-19T18:43:47+00:00 lp-ubn1 auditd type=SYSCALL msg=audit(1784486174.705:331): arch=c000003e syscall=257 success=yes exit=3 a0=ffffff9c a1=7ffeb71ce7ac a2=941 a3=1b6 items=2 ppid=26388 pid=26389 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts2 ses=1024 comm="touch" exe="/usr/bin/touch" subj=unconfined key="nginx_conf"#035ARCH=x86_64 SYSCALL=openat AUID="sadmin" UID="root" GI

```










