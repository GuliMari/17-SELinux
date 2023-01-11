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






