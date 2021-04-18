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
