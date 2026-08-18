# Lesson25_Logs

## Задание 1
Поднимаем две машины — web и log  

На web поднимаем nginx.  

Настраиваем центральный лог-сервер на любой системе по выбору:
journald;
rsyslog;
elk.
1.1 Настраиваем аудит, который будет отслеживать изменения конфигураций nginx.  

1.2 Все критичные логи с web должны собираться и локально и удаленно.  

1.3 Все логи с nginx должны уходить на удаленный сервер (локально только критичные).  

1.4 Логи аудита должны также уходить на удаленную систему



## Выполнение
Сервера:  
Alp-ansible 192.168.50.221  
Alp-nginx-web	192.168.50.219  
Alp-log 192.168.50.217  
Alp-elk 192.168.50.218 

playbook установки nginx
```
sadmin@alp-ansible:~$ cat ./projects/alp/lesson25_logs/install_nginx.yml 
---
- name: Установка Nginx на веб-сервер
  hosts: gr_alp-nginx-web
  become: yes
  tasks:
    - name: Установить Nginx
      apt:
        name: nginx
        state: present
        update_cache: yes    # обновить кэш apt перед установкой


```

Начну с пункта "1.2 Все критичные логи с web должны собираться и локально и удаленно"  
Залание несколько двусмыслено, я его расцениваю - как вообще все критичные логи с этого сервера, а не только критичные логи NGINX. Потому как в этом пункте четко сказано "Все критичные логи с web"


- на alp-nginx-web надо создать конфиг для настройки отправки критичных данных на сервер alp-log. это будет отдельный конфиг 73-to_alp-log.conf
```  
---
- name: настройка rsyslog на alp-nginx-web
  hosts: gr_alp-nginx-web
  become: yes
  tasks:
    - name: Создаем конфиг для отправки всех  критичных и выше сообщений на  alp-log
      copy:
        content: |
          *.crit 192.168.50.217
        dest: /etc/rsyslog.d/73-to_alp-log.conf
        mode: '0644'
      
# запускаем такой командой ansible-playbook config_rsyslog_alp-nginx-web.yml -k -K
```


- на alp-log надо создать конфиг для приема данных с сервера alp-nginx-web, конфиг будет называться /etc/rsyslog.d/98-alp-log.conf
```
---
- name: настройка rsyslog на alp-log
  hosts: gr_alp-log
  become: yes
  tasks:
    - name: Создаем конфиг для внешних логов
      copy:
        content: |
          $template RemoteLogs,"/var/log/rsyslog/%HOSTNAME%/%PROGRAMNAME%.log"
          # Все логи пишем по шаблону
          *.* ?RemoteLogs
          # дальше ничего не обрабатываем
          & ~
        dest: /etc/rsyslog.d/98-alp-nginx-web.conf
        mode: '0644'
      
# запускаем такой командой ansible-playbook config_98-alp-log.yml -k -K
```
- на alp-log надо поменять конфиг etc/rsyslog.conf тем самым включить модули приема (UDP/TCP) надо будет раскомментировать:  
#module(load="imudp")  
#input(type="imudp" port="514")  
#module(load="imtcp")  
#input(type="imtcp" port="514")

```
---
- name: Включить удаленный прием логов в rsyslog
  hosts: alp-log
  become: yes
  gather_facts: no

  tasks:
    - name: Раскомментировать модуль UDP
      lineinfile:
        path: /etc/rsyslog.conf
        regexp: '^#module\(load="imudp"\)'
        line: 'module(load="imudp")'
        state: present

    - name: Раскомментировать порт UDP
      lineinfile:
        path: /etc/rsyslog.conf
        regexp: '^#input\(type="imudp" port="514"\)'
        line: 'input(type="imudp" port="514")'
        state: present

    - name: Раскомментировать модуль TCP
      lineinfile:
        path: /etc/rsyslog.conf
        regexp: '^#module\(load="imtcp"\)'
        line: 'module(load="imtcp")'
        state: present

    - name: Раскомментировать порт TCP
      lineinfile:
        path: /etc/rsyslog.conf
        regexp: '^#input\(type="imtcp" port="514"\)'
        line: 'input(type="imtcp" port="514")'
        state: present

    - name: Перезапустить rsyslog
      service:
        name: rsyslog
        state: restarted

# запускаем командой запускаем такой командой ansible-playbook config_rsyslog.conf.yml -k -K

```

делаем отправку критичного сообщения sadmin@alp-nginx-web:~$ logger -p crit "критичная ошибка"
Видим что в на alp-log появился лог с сервера alp-nginx-web
```
sadmin@alp-log:~$ cat /var/log/rsyslog/alp-nginx-web/sadmin.log 
2026-08-18T12:57:43.084909+00:00 alp-nginx-web sadmin Test from alp-nginx-web
2026-08-18T13:07:00+00:00 alp-nginx-web sadmin: критичная ошибка
sadmin@alp-log:~$ 
```

Переходим к пункту "1.3 Все логи с nginx должны уходить на удаленный сервер (локально только критичные)."  
А вот здесь мы уже будем работать с логими именно nginx  
То есть вообще все логи nginx отправляем на alp-log, но критичные еще дублируются на локальном сервере alp-nginx-web
 
в конфиге nginx.conf приводим настройки к такому виду
```
error_log /var/log/nginx/error.log crit;
error_log syslog:server=192.168.50.217:514,tag=nginx_error;
# в блоке html
access_log syslog:server=192.168.50.217:514,tag=nginx_access,severity=info combined;
#   access_log /var/log/nginx/access.log;
```

делаем такой плейбук
```
- name: настройка логирования NGINX
  hosts: alp-nginx-web
  become: yes
  tasks:
    - name: копируем правильный конфиг
      copy:
        src: /projects/alp/lesson25_logs/nginx.conf
        dest: /etc/nginx/nginx.conf
        owner: root
        group: root
        mode: '0644'
      notify: restart nginx

  handlers:
    - name: restart nginx
      systemd:
        name: nginx
        state: restarted

# запускаем плейбук sudo ansible-playbook config_nginx.yml -k -k
```

видим что access логи пошли уже на сервер alp-log
```
sadmin@alp-log:~$ cat /var/log/rsyslog/alp-nginx-web/nginx_access.log 
2026-08-18T14:51:25+00:00 alp-nginx-web nginx_access: ::1 - - [18/Aug/2026:14:51:25 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/8.5.0"
2026-08-18T14:52:39+00:00 alp-nginx-web nginx_access: ::1 - - [18/Aug/2026:14:52:39 +0000] "GET /jkjk HTTP/1.1" 404 162 "-" "curl/8.5.0"
2026-08-18T14:53:18+00:00 alp-nginx-web nginx_access: ::1 - - [18/Aug/2026:14:53:18 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/8.5.0"
```

перехожу к пункту "1.1 Настраиваем аудит, который будет отслеживать изменения конфигураций nginx" 
получается нам надо отслеживать изменения в файле ``` /etc/nginx/nginx.conf ```  

делаем плейбук для установки и запуска auditd
```
---
- name: Установка auditd
  hosts: gr_alp-nginx-web
  become: yes
  tasks:
    - name: Установить auditd
      apt:
        name: 
          - auditd 
          - audispd-plugins
        state: present
        update_cache: yes    # обновить кэш apt перед установкой
    - name: Запустить и включить auditd
      systemd:
        name: auditd
        state: started
        enabled: yes
#запускаем командой ansible-playbook install_auditd.yml -k -K
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

# настраиваем конфиг
# ======================== Filebeat inputs =========================
filebeat.inputs:

- type: log
  enabled: true
  paths:
    - /var/log/nginx/access.log
    - /var/log/nginx/error.log
  tags: ["nginx"]
  fields:
    service: nginx
    server: lp-ubn1
  fields_under_root: true

# ======================== Output to Logstash ======================
output.logstash:
  hosts: ["192.168.50.226:5044"]
  ssl:
    enabled: false

# ======================== Logging ================================
logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
  permissions: 0644


#в конфиг nginx добавляем хранение логов в файлах
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

#логи пишуться
root@lp-ubn1:/home/sadmin# cat /var/log/nginx/access.log
::1 - - [21/Jul/2026:20:23:49 +0000] "GET / HTTP/1.1" 403 162 "-" "curl/8.5.0"
::1 - - [21/Jul/2026:20:24:38 +0000] "GET / HTTP/1.1" 403 162 "-" "curl/8.5.0"
::1 - - [21/Jul/2026:20:24:38 +0000] "GET / HTTP/1.1" 403 162 "-" "curl/8.5.0"

```










