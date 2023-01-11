# Практика с SELinux
## Описание/Пошаговая инструкция выполнения домашнего задания:
1.Запустить nginx на нестандартном порту 3-мя разными способами:
1.1.    переключатели setsebool;
1.2.    добавление нестандартного порта в имеющийся тип;
1.3.    формирование и установка модуля SELinux.
К сдаче:
README с описанием каждого решения (скриншоты и демонстрация приветствуются).
2. Обеспечить работоспособность приложения при включенном selinux.
-    развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems;
-    выяснить причину неработоспособности механизма обновления зоны (см. README);
-    предложить решение (или решения) для данной проблемы;
-    выбрать одно из решений для реализации, предварительно обосновав выбор;
-    реализовать выбранное решение и продемонстрировать его работоспособность.
К сдаче:
README с анализом причины неработоспособности, возможными способами решения и обоснованием выбора одного из них;
исправленный стенд или демонстрация работоспособной системы скриншотами и описанием.

## Выполнение ДЗ
### 1.Запустить nginx на нестандартном порту 3-мя разными способами

Не сразу заметила, что к ДЗ была приложена методичка с инструкцией, поэтому немного отличается от предложенного варианта исполнения.

Для выполнения ДЗ была развернута ВМ на CentOS7 с  уcтановленным `Nginx` и необходимыми для работы пакетами `policycoreutils-python`, `setroubleshoot`.

Поменяем стандартный порт в `/etc/nginx/conf.d/default.conf` на порт `4080` и перезапустим сервис.
Получаем следующую ошибку:

```bash
[root@localhost ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```

Воспользовавшись `journalctl -xe` или `sealert -a /var/log/audit/audit.log` находим варианты решения проблемы:

```
SELinux is preventing /usr/sbin/nginx from name_bind access on the tcp_socket port 4080.

*****  Plugin bind_ports (92.2 confidence) suggests   ************************

If you want to allow /usr/sbin/nginx to bind to network port 4080
Then you need to modify the port type.
Do
# semanage port -a -t PORT_TYPE -p tcp 4080
    where PORT_TYPE is one of the following: http_cache_port_t, http_port_t, jboss_management_port_t, jboss_messaging_port_t, ntop_port_t, puppet_port_t.

*****  Plugin catchall_boolean (7.83 confidence) suggests   ******************

If you want to allow nis to enabled
Then you must tell SELinux about this by enabling the 'nis_enabled' boolean.

Do
setsebool -P nis_enabled 1

*****  Plugin catchall (1.41 confidence) suggests   **************************

If you believe that nginx should be allowed name_bind access on the port 4080 tcp_socket by default.
Then you should report this as a bug.
You can generate a local policy module to allow this access.
Do
allow this access for now by executing:
# ausearch -c 'nginx' --raw | audit2allow -M my-nginx
# semodule -i my-nginx.pp
```

1. Используем первый вариант - разрешим Nginx подключаться по нестандартному порту:

```bash
[root@localhost ~]# semanage port -a -t http_port_t -p tcp 4080
[root@localhost ~]# systemctl restart nginx
```
Проверим, работает ли сервис:

```bash
[root@localhost ~]# ss -ntlp | grep nginx
LISTEN     0      128          *:4080                     *:*                   users:(("nginx",pid=2532,fd=6),("nginx",pid=2531,fd=6))
[root@localhost ~]# curl localhost:4080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

2. Второй вариант решения проблемы - использовать переключатели:

```bash
[root@localhost ~]# setsebool -P nis_enabled 1
[root@localhost ~]# systemctl restart nginx
[root@localhost ~]# ss -ntlp | grep nginx
LISTEN     0      128          *:5080                     *:*                   users:(("nginx",pid=21662,fd=6),("nginx",pid=21661,fd=6))
[root@localhost ~]# curl localhost:5080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```

3. Еще один вариант - формирование и установка модуля:

```bash
[root@localhost ~]# ausearch -c 'nginx' --raw | audit2allow -M my-nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i my-nginx.pp

[root@localhost ~]# semodule -i my-nginx.pp
[root@localhost ~]# systemctl restart nginx
[root@localhost ~]# ss -ntlp | grep nginx
LISTEN     0      128          *:6080                     *:*                   users:(("nginx",pid=2544,fd=6),("nginx",pid=2543,fd=6))
[root@localhost ~]# curl localhost:6080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
[root@localhost ~]# 
```
### 2. Обеспечить работоспособность приложения при включенном selinux.

Разворачиваем 2 ВМ с помощью `vagrant up` и подключаемся к ним по SSH.
При попытке внести изменения в зону на клиенте получаем ошибку:

```bash
[vagrant@client ~]$  nsupdate -k /etc/named.zonetransfer.key
> server 192.168.88.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.88.15
> send
update failed: SERVFAIL
> quit
```

В логах SELinux отсутвуют ошибки:

```bash
[root@client ~]# cat /var/log/audit/audit.log | audit2why
```
Делаем вывод, что проблема не связана с клиентом и смотрим сервер:

```bash
[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1673453132.114:1969): avc:  denied  { create } for  pid=5327 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.
[root@ns01 ~]# sealert -a /var/log/audit/audit.log
100% done
found 1 alerts in /var/log/audit/audit.log
--------------------------------------------------------------------------------

SELinux is preventing /usr/sbin/named from create access on the file named.ddns.lab.view1.jnl.
...
```
Мы видим, что SELinux запретил доступ на чтение и запись к файлу `named.ddns.lab.view1.jnl`, т.к. его контекст безопасности имеет неверный тип.

```bash
[root@ns01 ~]# ls -lZ /etc/named
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   dynamic
-rw-rw----. root named system_u:object_r:etc_t:s0       named.88.168.192.rev
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab.view1
-rw-rw----. root named system_u:object_r:etc_t:s0       named.newdns.lab
```
Видим, что `named` не может получить доступ к файлам, имеющих тип `etc_t`. Для решения проблемы необходимо изменить тип. Чтобы понять, какой тип нам нужен, sыведем список всех контекстов, доступных для `named` и выберем нужный:

```bash
[root@ns01 ~]# semanage fcontext -l | grep named
/etc/rndc.*                                        regular file       system_u:object_r:named_conf_t:s0 
/var/named(/.*)?                                   all files          system_u:object_r:named_zone_t:s0 
/etc/unbound(/.*)?                                 all files          system_u:object_r:named_conf_t:s0 
/var/run/bind(/.*)?                                all files          system_u:object_r:named_var_run_t:s0 
/var/log/named.*                                   regular file       system_u:object_r:named_log_t:s0 
/var/run/named(/.*)?                               all files          system_u:object_r:named_var_run_t:s0 
/var/named/data(/.*)?                              all files          system_u:object_r:named_cache_t:s0 
/dev/xen/tapctrl.*                                 named pipe         system_u:object_r:xenctl_t:s0 
/var/run/unbound(/.*)?                             all files          system_u:object_r:named_var_run_t:s0 
/var/lib/softhsm(/.*)?                             all files          system_u:object_r:named_cache_t:s0 
/var/lib/unbound(/.*)?                             all files          system_u:object_r:named_cache_t:s0 
/var/named/slaves(/.*)?                            all files          system_u:object_r:named_cache_t:s0 
/var/named/chroot(/.*)?                            all files          system_u:object_r:named_conf_t:s0 
/etc/named\.rfc1912.zones                          regular file       system_u:object_r:named_conf_t:s0 
/var/named/dynamic(/.*)?                           all files          system_u:object_r:named_cache_t:s0 
/var/named/chroot/etc(/.*)?                        all files          system_u:object_r:etc_t:s0 
/var/named/chroot/lib(/.*)?                        all files          system_u:object_r:lib_t:s0 
...
```
Изменим тип контекста безопасности для каталога /etc/named: 

```bash
[root@ns01 ~]# chcon -R -t named_zone_t /etc/named
[root@ns01 ~]# ls -lZ /etc/named
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.88.168.192.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab

```
Теперь попробуем внести изменения на клиенте:

```bash
[vagrant@client ~]$  nsupdate -k /etc/named.zonetransfer.key
> server 192.168.88.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.88.15
> send
> quit
[vagrant@client ~]$ dig www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.10 <<>> @192.168.88.10 www.ddns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 46179
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.			IN	A

;; ANSWER SECTION:
www.ddns.lab.		60	IN	A	192.168.88.15

;; AUTHORITY SECTION:
ddns.lab.		3600	IN	NS	ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.		3600	IN	A	192.168.88.10

;; Query time: 1 msec
;; SERVER: 192.168.88.10#53(192.168.88.10)
;; WHEN: Wed Jan 11 16:24:46 UTC 2023
;; MSG SIZE  rcvd: 96

```

Второй более быстрый вариант решения проблемы - это воспользоваться подсказкой `sealert`:

```bash
[root@ns01 ~]# sealert -a /var/log/audit/audit.log
100% done
found 1 alerts in /var/log/audit/audit.log
--------------------------------------------------------------------------------

SELinux is preventing /usr/sbin/named from create access on the file named.ddns.lab.view1.jnl.

*****  Plugin catchall_labels (83.8 confidence) suggests   *******************

If you want to allow named to have create access on the named.ddns.lab.view1.jnl file
Then you need to change the label on named.ddns.lab.view1.jnl
Do
# semanage fcontext -a -t FILE_TYPE 'named.ddns.lab.view1.jnl'
where FILE_TYPE is one of the following: dnssec_trigger_var_run_t, ipa_var_lib_t, krb5_host_rcache_t, krb5_keytab_t, named_cache_t, named_log_t, named_tmp_t, named_var_run_t, named_zone_t.
Then execute:
restorecon -v 'named.ddns.lab.view1.jnl'
...
[root@ns01 ~]# semanage fcontext -a -t named_zone_t 'named.ddns.lab.view1.jnl'
[root@ns01 ~]# restorecon -v /etc/named/dynamic/named.ddns.lab.view1










