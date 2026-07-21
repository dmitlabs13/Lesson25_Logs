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


## ELK

установка
```
sadmin@lp-ubn7-elk:~/elk_dst$ sudo dpkg -i elasticsearch_8.9.1_amd64_224190_300799-466156-a5c8b3.deb
[sudo] password for sadmin:
Selecting previously unselected package elasticsearch.
(Reading database ... 126552 files and directories currently installed.)
Preparing to unpack elasticsearch_8.9.1_amd64_224190_300799-466156-a5c8b3.deb ...
Creating elasticsearch group... OK
Creating elasticsearch user... OK
Unpacking elasticsearch (8.9.1) ...
Setting up elasticsearch (8.9.1) ...
--------------------------- Security autoconfiguration information ------------------------------

Authentication and authorization are enabled.
TLS for the transport and HTTP layers is enabled and configured.

The generated password for the elastic built-in superuser is : iwUafYeOcibIr3JVqLQ5

If this node should join an existing cluster, you can reconfigure this with
'/usr/share/elasticsearch/bin/elasticsearch-reconfigure-node --enrollment-token <token-here>'
after creating an enrollment token on your existing cluster.

You can complete the following actions at any time:

Reset the password of the elastic built-in superuser with
'/usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic'.

Generate an enrollment token for Kibana instances with
 '/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana'.

Generate an enrollment token for Elasticsearch nodes with
'/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s node'.

-------------------------------------------------------------------------------------------------
### NOT starting on installation, please execute the following statements to configure elasticsearch service to start automatically using systemd
 sudo systemctl daemon-reload
 sudo systemctl enable elasticsearch.service
### You can start elasticsearch service by executing
 sudo systemctl start elasticsearch.service

root@lp-ubn7-elk:/home/sadmin/elk_dst# systemctl daemon-reload
root@lp-ubn7-elk:/home/sadmin/elk_dst# systemctl status elasticsearch.service
○ elasticsearch.service - Elasticsearch
     Loaded: loaded (/usr/lib/systemd/system/elasticsearch.service; disabled; prese>
     Active: inactive (dead)
       Docs: https://www.elastic.co

root@lp-ubn7-elk:/home/sadmin/elk_dst# systemctl start elasticsearch.service
root@lp-ubn7-elk:/home/sadmin/elk_dst# systemctl enabled elasticsearch.service
Unknown command verb 'enabled', did you mean 'enable'?
root@lp-ubn7-elk:/home/sadmin/elk_dst# systemctl enable elasticsearch.service
Created symlink /etc/systemd/system/multi-user.target.wants/elasticsearch.service → /usr/lib/systemd/system/elasticsearch.service.
root@lp-ubn7-elk:/home/sadmin/elk_dst#


#проверяем
root@lp-ubn7-elk:/home/sadmin/elk_dst# curl -k -u elastic:iwUafYeOcibIr3JVqLQ5 https://localhost:9200
{
  "name" : "lp-ubn7-elk",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "fXTE1RcARJuY2nU_o_iQcA",
  "version" : {
    "number" : "8.9.1",
    "build_flavor" : "default",
    "build_type" : "deb",
    "build_hash" : "a813d015ef1826148d9d389bd1c0d781c6e349f0",
    "build_date" : "2023-08-10T05:02:32.517455352Z",
    "build_snapshot" : false,
    "lucene_version" : "9.7.0",
    "minimum_wire_compatibility_version" : "7.17.0",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "You Know, for Search"
}

# ставим kibana
root@lp-ubn7-elk:/home/sadmin/elk_dst# apt install ./kibana_8.9.1_amd64_224190_68eb0f__1-466156-1c2952.deb
root@lp-ubn7-elk:/home/sadmin/elk_dst# systemctl daemon-reload
root@lp-ubn7-elk:/home/sadmin/elk_dst# systemctl start kibana
root@lp-ubn7-elk:/home/sadmin/elk_dst# systemctl enabl kibana
Unknown command verb 'enabl', did you mean 'enable'?
root@lp-ubn7-elk:/home/sadmin/elk_dst# systemctl enable kibana
Created symlink /etc/systemd/system/multi-user.target.wants/kibana.service → /usr/lib/systemd/system/kibana.service.
root@lp-ubn7-elk:/home/sadmin/elk_dst# systemctl status kibana
● kibana.service - Kibana
     Loaded: loaded (/usr/lib/systemd/system/kibana.service; enabled; preset: enabl>
     Active: active (running) since Mon 2026-07-20 19:13:26 UTC; 21s ago
       Docs: https://www.elastic.co
   Main PID: 39229 (node)
      Tasks: 11 (limit: 2263)
     Memory: 409.2M (peak: 409.5M)
        CPU: 12.239s
     CGroup: /system.slice/kibana.service
             └─39229 /usr/share/kibana/bin/../node/bin/node /usr/share/kibana/bin/.>

Jul 20 19:13:40 lp-ubn7-elk kibana[39229]: [2026-07-20T19:13:40.046+00:00][INFO ][p>
Jul 20 19:13:40 lp-ubn7-elk kibana[39229]: [2026-07-20T19:13:40.046+00:00][INFO ][p>
Jul 20 19:13:40 lp-ubn7-elk kibana[39229]: [2026-07-20T19:13:40.046+00:00][INFO ][p>
Jul 20 19:13:40 lp-ubn7-elk kibana[39229]: [2026-07-20T19:13:40.046+00:00][INFO ][p>
Jul 20 19:13:40 lp-ubn7-elk kibana[39229]: [2026-07-20T19:13:40.182+00:00][INFO ][h>
Jul 20 19:13:40 lp-ubn7-elk kibana[39229]: [2026-07-20T19:13:40.340+00:00][INFO ][p>
Jul 20 19:13:40 lp-ubn7-elk kibana[39229]: [2026-07-20T19:13:40.342+00:00][INFO ][p>
Jul 20 19:13:40 lp-ubn7-elk kibana[39229]: [2026-07-20T19:13:40.368+00:00][INFO ][r>
Jul 20 19:13:40 lp-ubn7-elk kibana[39229]: i Kibana has not been configured.
Jul 20 19:13:40 lp-ubn7-elk kibana[39229]: Go to http://localhost:5601/?code=434512>


# в конфиге kibana указываем
server.host: 192.168.50.226
systemctl restart kibana

#кибана открылась
<img width="609" height="584" alt="image" src="https://github.com/user-attachments/assets/ff2cb5d0-7743-43b5-94c6-c57ccd6c2a90" />
# получаем токен, код, настраивается elastic

# ставим logstash
root@lp-ubn7-elk:/home/sadmin/elk_dst# apt install ./logstash_8.9.1_amd64_224190_e8ea1a__1-466156-c17c37.deb

nano /etc/logstash/conf.d/input.conf

# содержимое
input {
  beats {
    port => 5044
  }
}

nano /etc/logstash/conf.d/output.conf
#содержимое
output {
  elasticsearch {
    hosts => ["https://localhost:9200"]
    ssl => true
    ssl_certificate_verification => false
    manage_template => false
    index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
    user => elastic
    password => "iwUafYeOcibIr3JVqLQ5"
  }
}



```

Ставим filebeat на сервер c nginx
```
sadmin@lp-ubn1:~$ sudo apt install ./filebeat_8.9.1_amd64_224190_e0af99__1-466156-b6f621.deb


```










