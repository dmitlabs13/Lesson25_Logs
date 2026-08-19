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

** Начну с пункта "1.2 Все критичные логи с web должны собираться и локально и удаленно"  
Задание несколько двухсмыслено, я его понял как - вообще все критичные логи с этого сервера, а не только критичные логи NGINX. Потому как в этом пункте четко сказано "Все критичные логи с web"


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
А вот здесь мы уже будем работать с логами именно nginx  
То есть вообще все логи именно nginx отправляем на alp-log, но критичные еще дублируются(фактически остаются) на локальном сервере alp-nginx-web
 
в конфиге nginx.conf приводим настройки к такому виду:
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

** перехожу к пункту "1.1 Настраиваем аудит, который будет отслеживать изменения конфигураций nginx" 
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
делаем файл nginx_changes.rules с нашим правилом 
``` -w /etc/nginx/nginx.conf -p wa -k nginx_conf_changes ```

делаем плейбук для отправки этого файла на сервер alp-nginx-web
```
---
- name: настройка аудита конфига  NGINX
  hosts: alp-nginx-web
  become: yes
  tasks:
    - name: копируем файл правила аудита  конфига nginx
      copy:
        src: /home/sadmin/projects/alp/lesson25_logs/nginx_changes.rules
        dest: /etc/audit/rules.d/nginx_changes.rules
        owner: root
        group: root
        mode: '0644'
      notify: restart auditd

  handlers:
    - name: restart auditd 
      systemd:
        name: auditd
        state: restarted

#запускаем командой ansible-playbook config_audit.yml -k -K
```
Поменяли конфиг nginx, в логе аудита видим  
```
sadmin@alp-nginx-web:~$ tail -f /var/log/audit/audit.log | grep nginx_conf
type=SYSCALL msg=audit(1787075893.628:514): arch=c000003e syscall=257 success=yes exit=3 a0=ffffff9c a1=5b1208693eb0 a2=241 a3=1b6 items=2 ppid=20696 pid=20697 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts3 ses=991 comm="nano" exe="/usr/bin/nano" subj=unconfined key="nginx_conf_changes"ARCH=x86_64 SYSCALL=openat AUID="sadmin" UID="root" GID="root" EUID="root" SUID="root" FSUID="root" EGID="root" SGID="root" FSGID="root"
```

** Остался пункт "1.4 Логи аудита должны также уходить на удаленную систему"


Сначала включим плагин отправки логов аудита в локальный rsyslog  



```
nano /etc/audit/plugins.d/syslog.conf
# This file controls the configuration of the syslog plugin.
# It simply takes events and writes them to syslog. The
# arguments provided can be the default priority that you
# want the events written with. And optionally, you can give
# a second argument indicating the facility that you want events
# logged to. Valid options are LOG_LOCAL0 through 7, LOG_AUTH,
# LOG_AUTHPRIV, LOG_DAEMON, LOG_SYSLOG, and LOG_USER.

active = yes
direction = out
path = /sbin/audisp-syslog
type = always
args = LOG_INFO
format = string

```



меняем файл /etc/nginx/nginx.conf, видим в логах на alp-log 
```
2026-08-19T09:55:03.857149+00:00 alp-nginx-web kernel: audit: type=1300 audit(1787133303.855:1931): arch=c000003e syscall=257 success=yes exit=3 a0=ffffff9c a1=5bb14b0b6f00 a2=241 a3=1b6 items=2 ppid=29899 pid=29900 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts4 ses=991 comm="nano" exe="/usr/bin/nano" subj=unconfined key="nginx_conf_changes"
```
значит аудит уже отдает логи в syslog. Теперь надо сделтаь отправку этих логов на alp-log.  
добавим в наш файл 30-auditd.conf
```
# Отправляем логи аудита на центральный сервер
if $programname == 'audisp-syslog' then {
    action(type="omfwd" target="192.168.50.217" port="514" protocol="tcp")
    stop
}
```
Снова запускаем плейбук ansible-playbook config_audit.yml -k -K  
Затем меняем nginx.conf и в итоге на alp-log видим логи
```
sadmin@alp-log:/var/log/rsyslog/alp-nginx-web$ cat audisp-syslog.log | grep nginx_conf_changes
2026-08-19T10:58:05+00:00 alp-nginx-web audisp-syslog: type=SYSCALL msg=audit(1787137085.050:2169): arch=c000003e syscall=257 success=no exit=-13 a0=ffffff9c a1=566a14e349d0 a2=441 a3=1b6 items=2 ppid=19187 pid=30386 auid=1000 uid=1000 gid=1000 euid=1000 suid=1000 fsuid=1000 egid=1000 sgid=1000 fsgid=1000 tty=pts2 ses=991 comm="bash" exe="/usr/bin/bash" subj=unconfined key="nginx_conf_changes" ARCH=x86_64 SYSCALL=openat AUID="sadmin" UID="sadmin" GID="sadmin" EUID="sadmin" SUID="sadmin" FSUID="sadmin" EGID="sadmin" SGID="sadmin" FSGID="sadmin"
2026-08-19T10:58:17+00:00 alp-nginx-web audisp-syslog: type=SYSCALL msg=audit(1787137097.551:2170): arch=c000003e syscall=257 success=no exit=-13 a0=ffffff9c a1=566a14efd540 a2=441 a3=1b6 items=2 ppid=19187 pid=30387 auid=1000 uid=1000 gid=1000 euid=1000 suid=1000 fsuid=1000 egid=1000 sgid=1000 fsgid=1000 tty=pts2 ses=991 comm="bash" exe="/usr/bin/bash" subj=unconfined key="nginx_conf_changes" ARCH=x86_64 SYSCALL=openat AUID="sadmin" UID="sadmin" GID="sadmin" EUID="sadmin" SUID="sadmin" FSUID="sadmin" EGID="sadmin" SGID="sadmin" FSGID="sadmin"
2026-08-19T10:59:28+00:00 alp-nginx-web audisp-syslog: type=SYSCALL msg=audit(1787137168.918:2171): arch=c000003e syscall=257 success=no exit=-13 a0=ffffff9c a1=566a14dca550 a2=441 a3=1b6 items=2 ppid=19187 pid=30388 auid=1000 uid=1000 gid=1000 euid=1000 suid=1000 fsuid=1000 egid=1000 sgid=1000 fsgid=1000 tty=pts2 ses=991 comm="bash" exe="/usr/bin/bash" subj=unconfined key="nginx_conf_changes" ARCH=x86_64 SYSCALL=openat AUID="sadmin" UID="sadmin" GID="sadmin" EUID="sadmin" SUID="sadmin" FSUID="sadmin" EGID="sadmin" SGID="sadmin" FSGID="sadmin"
2026-08-19T10:59:55+00:00 alp-nginx-web audisp-syslog: type=SYSCALL msg=audit(1787137195.717:2176): arch=c000003e syscall=257 success=yes exit=3 a0=ffffff9c a1=58f5566e7608 a2=441 a3=1b6 items=2 ppid=30391 pid=30392 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts4 ses=991 comm="sh" exe="/usr/bin/dash" subj=unconfined key="nginx_conf_changes" ARCH=x86_64 SYSCALL=openat AUID="sadmin" UID="root" GID="root" EUID="root" SUID="root" FSUID="root" EGID="root" SGID="root" FSGID="root"
2026-08-19T11:00:58+00:00 alp-nginx-web audisp-syslog: type=SYSCALL msg=audit(1787137258.617:2191): arch=c000003e syscall=257 success=yes exit=3 a0=ffffff9c a1=5f4a8b764608 a2=441 a3=1b6 items=2 ppid=30402 pid=30403 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts4 ses=991 comm="sh" exe="/usr/bin/dash" subj=unconfined key="nginx_conf_changes" ARCH=x86_64 SYSCALL=openat AUID="sadmin" UID="root" GID="root" EUID="root" SUID="root" FSUID="root" EGID="root" SGID="root" FSGID="root"
2026-08-19T11:01:27+00:00 alp-nginx-web audisp-syslog: type=SYSCALL msg=audit(1787137287.843:2198): arch=c000003e syscall=257 success=yes exit=3 a0=ffffff9c a1=5dbb08853cc0 a2=241 a3=1b6 items=2 ppid=30405 pid=30406 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts4 ses=991 comm="nano" exe="/usr/bin/nano" subj=unconfined key="nginx_conf_changes" ARCH=x86_64 SYSCALL=openat AUID="sadmin" UID="root" GID="root" EUID="root" SUID="root" FSUID="root" EGID="root" SGID="root" FSGID="root"
```

















