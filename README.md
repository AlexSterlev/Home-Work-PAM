## Home-Work-PAM
Home Work PAM

 ### Запретить всем пользователям, кроме группы admin логин в выходные и праздничные днидоступ с помощью SSH, определенные пользователи будни (понедельник – пятница) с 8 00 до 17 00.

 -  Добавляем pam_time.so в /etc/pam.d/sshd

  ````
account required /usr/lib64/security/pam_time.so
auth	   required     pam_sepermit.so
auth	   substack     password-auth
auth	   include	postlogin
# Used with polkit to reauthorize users in remote sessions
-auth	   optional     pam_reauthorize.so prepare
account    required     pam_nologin.so
account    required     pam_time.so
account    include	password-auth
password   include	password-auth
# pam_selinux.so close should be the first session rule
session    required     pam_selinux.so close
session    required     pam_loginuid.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session    required     pam_selinux.so open env_params
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    include	password-auth
session    include	postlogin
# Used with polkit to reauthorize users in remote sessions
-session   optional     pam_reauthorize.so prepare
  ````

 - Редактируем /etc/security/time.conf

  ````
  *;*;user1|user2|user3;Wk0800-1700
  ````
- Первое поле сервисы - login, ssh, gdb
- Второе поле косоли - tty, pst/X и т.д.  
- Третье поле имена, группы 
- Четвертое поле, время ограничения. В данный момент - понедельник - пятница (Wk) с восьми утра до пяти вечера (0800-1700).


### Дать пользователю права работать с докером и возможность перезапуска докер сервиса

- Установим и запустим docker:

 ````
  [root@pam ~]# yum install -y yum-utils
[root@pam ~]# yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
[root@pam ~]# yum install -y docker-ce docker-ce-cli containerd.io
[root@pam ~]# systemctl enable --now docker
[root@pam ~]# systemctl status docker
● docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; vendor preset: disabled)
   Active: active (running) since Пн 2020-05-18 05:45:22 UTC; 48s ago
     Docs: https://docs.docker.com
 Main PID: 5300 (dockerd)
   CGroup: /system.slice/docker.service
           └─5300 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/contain...
  ````
  
