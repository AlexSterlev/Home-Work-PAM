## Homework-PAM
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
   Active: active (running) since Пн 2021-04-17 09:45:22 UTC; 48s ago
     Docs: https://docs.docker.com
 Main PID: 5300 (dockerd)
   CGroup: /system.slice/docker.service
           └─5300 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/contain...
  ````
  
- Дадим права для работы с docker пользователю user1.

 ````
 [root@pam ~]# usermod -aG docker user1
[root@pam ~]# newgrp docker
[user1@pam ~]$ docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
0e03bdcc26d7: Pull complete 
Digest: sha256:6a65f928fb91fcfbc963f7aa6d57c8eeb426ad9a20c7ee045538ef34847f44f1
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

...
  ````
  
  - PolKit. Включим логирование. Для этого создадим файл /etc/polkit-1/rules.d/00-access.rules со следующим содержимым:

 ````
[root@pam ~]# cat /etc/polkit-1/rules.d/00-access.rules
polkit.addRule(function(action, subject) {
    polkit.log("action=" + action);
    polkit.log("subject=" + subject);
});
 ````
- Теперь попробуем рестартануть docker service и увидим запрос аутентифицироваться под рутом:


 ````
[user1@pam ~]$ systemctl restart docker
==== AUTHENTICATING FOR org.freedesktop.systemd1.manage-units ===
Authentication is required to manage system services or units.
Authenticating as: root
Password: 
 ````
 
- В логе /var/log/secure увидим следующие строки:

 ````
Apr 17 11:04:46 pam polkitd[1653]: Registered Authentication Agent for unix-process:5805:364125 (system bus name :1.57 [/usr/bin/pkttyagent --notify-fd 5 --fallback], object path /org/freedesktop/PolicyKit1/AuthenticationAgent, locale en_US.UTF-8)
Apr 17 11:04:46 pam polkitd[1653]: /etc/polkit-1/rules.d/00-access.rules:2: action=[Action id='org.freedesktop.systemd1.manage-units']
Apr 17 11:04:46 pam polkitd[1653]: /etc/polkit-1/rules.d/00-access.rules:3: subject=[Subject pid=5805 user='user1' groups=user1,docker seat='' session='5' local=false active=true]
Apr 17 11:04:48 pam polkitd[1653]: Unregistered Authentication Agent for unix-process:5805:364125 (system bus name :1.57, object path /org/freedesktop/PolicyKit1/AuthenticationAgent, locale en_US.UTF-8) (disconnected from bus)
Apr 17 11:04:48 pam polkitd[1653]: Operator of unix-process:5805:364125 FAILED to authenticate to gain authorization for action org.freedesktop.systemd1.manage-units for system-bus-name::1.58 [<unknown>] (owned by unix-user:user1)
 ````
 
- Пользователю user1 не удалось пройти проверку подлинности для получения разрешения на действие org.freedesktop.systemd1.manage-units.



- Предоставим пользователю user1 право рестартить docker service. Для этого создадим правило /etc/polkit-1/rules.d/01-dockerrestart.rules со следующим содержанием:

 ````
[root@localhost ~]# cat /etc/polkit-1/rules.d/01-dockerrestart.rules
polkit.addRule(function(action, subject) {
    if (action.id == "org.freedesktop.systemd1.manage-units" &&
        action.lookup("unit") == "docker.service" &&
        action.lookup("verb") == "restart" &&
	subject.user == "user1") {
        return polkit.Result.YES;
    }
});
 ````
 
- Чтобы данная политика отработала, необходимо обновить systemd, так как в CentOS 7 используется systemd v219, а для функционирования правил action.lookup("unit") и action.lookup("verb") необходима systemd v226. В CentOS 8 версия systemd удовлетворяет нашим требованиям.


- Обновим systemd из репозитория с бэкпортами (страница руководства):


 ````
[root@pam ~]# setenforce 0
[root@pam ~]# yum install -y wget
[root@pam ~]#  wget https://copr.fedorainfracloud.org/coprs/jsynacek/systemd-backports-for-centos-7/repo/epel-7/jsynacek-systemd-backports-for-centos-7-epel-7.repo -O /etc/yum.repos.d/jsynacek-systemd-centos-7.repo
[root@pam ~]# yum update systemd -y
[root@pam ~]# setenforce 1
[root@pam ~]# systemctl --version
systemd 234
+PAM +AUDIT +SELINUX +IMA -APPARMOR +SMACK +SYSVINIT +UTMP +LIBCRYPTSETUP +GCRYPT +GNUTLS +ACL +XZ +LZ4 +SECCOMP +BLKID +ELFUTILS +KMOD -IDN2 +IDN default-hierarchy=hybrid
Теперь можно проверить, имеет ли право user1 рестартить docker.service. Также проверим, может ли он его остановить:
[user1@pam ~]$ systemctl restart docker
[user1@pam ~]$ systemctl status docker
● docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; vendor preset: 
   Active: active (running) since Wed 2021-04-17 13:01:22 UTC; 11s ago
     Docs: https://docs.docker.com
 Main PID: 4297 (dockerd)
    Tasks: 10
   Memory: 39.6M
      CPU: 205ms
   CGroup: /system.slice/docker.service
           └─4297 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd
[user1@pam ~]$ systemctl stop docker
==== AUTHENTICATING FOR org.freedesktop.systemd1.manage-units ===
Authentication is required to stop 'docker.service'.
Authenticating as: root
Password:
 ````
 
- Как видим, docker.service успешно перезапустился, но при попытке его остановки появилось требование аутентифицироваться в качестве рута.
