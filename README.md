# Пользователи и группы. Авторизация и аутентификация
## Цель домашнего задания:
Научиться создавать пользователей и добавлять им ограничения.
______________________________________________________________
Создаём пользователей и пароли для них:
```
[vagrant@pam ~]$ sudo -i
[root@pam ~]# sudo useradd otusadm && sudo useradd otus
[root@pam ~]# echo "Otus2022!" | sudo passwd --stdin otusadm && echo "Otus2022!" | sudo passwd --stdin otus
Changing password for user otusadm.
passwd: all authentication tokens updated successfully.
Changing password for user otus.
passwd: all authentication tokens updated successfully.
```
Создаём группу, добавляем в неё пользователей и пробуем подключиться к VM из другого терминала:
```
[root@pam ~]# sudo groupadd -f admin
[root@pam ~]# usermod otusadm -a -G admin && usermod root -a -G admin && usermod vagrant -a -G admin
feelins@FeeLinS-PC:~$ ssh otus@192.168.57.10
otus@192.168.57.10's password: 
Last failed login: Sat Feb  3 14:56:55 UTC 2024 from 192.168.57.1 on ssh:notty
There were 2 failed login attempts since the last successful login.
[otus@pam ~]$ 
```
Далее настроим правило, по которому все пользователи кроме тех, что указаны в группе admin не смогут подключаться в выходные дни:
```
[otus@pam ~]$ cat /etc/group | grep admin
admin:x:1003:otusadm,root,vagrant
```
Создадим скрипт `/usr/local/bin/login.sh`:
```
#!/bin/bash
if [ $(date +%a) = "Sat" ] || [ $(date +%a) = "Sun" ]; then
if getent group admin | grep -qw "$PAM_USER"; then
        exit 0
      else
        exit 1
    fi
  else
    exit 0
fi
```
Добавим права на исполнение файла: `chmod +x /usr/local/bin/login.sh`\
Укажем в файле /etc/pam.d/sshd модуль pam_exec и наш скрипт:
```
#%PAM-1.0
auth       substack     password-auth
auth       include      postlogin
auth       required     pam_exec.so /usr/local/bin/login.sh
account    required     pam_sepermit.so
account    required     pam_nologin.so
account    include      password-auth
password   include      password-auth
# pam_selinux.so close should be the first session rule
session    required     pam_selinux.so close
session    required     pam_loginuid.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session    required     pam_selinux.so open env_params
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    optional     pam_motd.so
session    include      password-auth
session    include      postlogin
```
На этом настройка закончена, проверяем:
```
[root@pam ~]# date
Sat Feb  3 15:46:04 UTC 2024

feelins@FeeLinS-PC:~$ ssh otus@192.168.57.10
otus@192.168.57.10's password: 
Permission denied, please try again.

feelins@FeeLinS-PC:~$ ssh otusadm@192.168.57.10
otusadm@192.168.57.10's password: 
Last login: Sat Feb  3 15:25:03 2024 from 192.168.57.1
 
[root@pam ~]# date --set="2 FEB 2024 18:00:00"
Fri Feb  2 18:00:00 UTC 2024
[root@pam ~]# date
Fri Feb  2 18:00:04 UTC 2024

feelins@FeeLinS-PC:~$ ssh otus@192.168.57.10
otus@192.168.57.10's password: 
Last failed login: Sat Feb  3 15:44:43 UTC 2024 from 192.168.57.1 on ssh:notty
There were 1 failed login attempts since the last successful login.
Last login: Fri Feb  2 18:00:22 2024 from 192.168.57.1
```