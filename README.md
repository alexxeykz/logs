# logs
Настраиваем центральный сервер для сбора логов.
```
1. В вагранте поднимаем 2 машины web и log
2. На web поднимаем nginx
3. На log настраиваем центральный лог сервер на любой системе на выбор:
 journald;
 rsyslog;
 elk.
```
```
Настраиваем в Vagrantfile на дистрибутиве Ubuntu 22.04 2 гостевые машины web и log

Vagrant.configure("2") do |config|
  # Base VM OS configuration.
  config.vm.box = "ubuntu-22.04"
  config.vm.provider :virtualbox do |v|
    v.memory = 1512
    v.cpus = 2
  end

  # Define two VMs with static private IP addresses.
  boxes = [
    { :name => "web",
      :ip => "192.168.56.10",
    },
    { :name => "log",
      :ip => "192.168.56.15",
    }
  ]
  # Provision each of the VMs.
  boxes.each do |opts|
    config.vm.define opts[:name] do |config|
      config.vm.hostname = opts[:name]
      config.vm.network "private_network", ip: opts[:ip]

    end
  end
end
```

Приводим их к правильному времени и дате:
```
timedatectl set-timezone Europe/Moscow



Далее устанавливаем на Web машину NGINX

vagrant@web:~$ sudo -s
root@web:/home/vagrant# apt update && apt install -y nginx

Hit:1 http://us.archive.ubuntu.com/ubuntu jammy InRelease
Get:2 http://us.archive.ubuntu.com/ubuntu jammy-updates InRelease [128 kB]
Get:3 http://us.archive.ubuntu.com/ubuntu jammy-backports InRelease [127 kB]
Get:4 http://us.archive.ubuntu.com/ubuntu jammy-security InRelease [129 kB]
Get:5 http://us.archive.ubuntu.com/ubuntu jammy-updates/main amd64 Packages [2,058 kB]
Get:6 http://us.archive.ubuntu.com/ubuntu jammy-updates/main Translation-en [355 kB]
Get:7 http://us.archive.ubuntu.com/ubuntu jammy-updates/restricted amd64 Packages [2,495 kB]
Get:8 http://us.archive.ubuntu.com/ubuntu jammy-updates/restricted Translation-en [429 kB]
Get:9 http://us.archive.ubuntu.com/ubuntu jammy-updates/universe amd64 Packages [1,124 kB]
Get:10 http://us.archive.ubuntu.com/ubuntu jammy-updates/universe Translation-en [261 kB]
Get:11 http://us.archive.ubuntu.com/ubuntu jammy-updates/multiverse amd64 Packages [43.3 kB]
```
```
Проверяем:
```
```
root@web:/home/vagrant# systemctl status nginx

● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2024-09-18 16:31:31 MSK; 2min 57s ago
       Docs: man:nginx(8)
    Process: 2486 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
    Process: 2487 ExecStart=/usr/sbin/nginx -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
   Main PID: 2578 (nginx)
      Tasks: 3 (limit: 1586)
     Memory: 4.7M
        CPU: 25ms
     CGroup: /system.slice/nginx.service
             ├─2578 "nginx: master process /usr/sbin/nginx -g daemon on; master_process on;"
             ├─2581 "nginx: worker process" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ""
             └─2582 "nginx: worker process" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ""

Sep 18 16:31:31 web systemd[1]: Starting A high performance web server and a reverse proxy server...
Sep 18 16:31:31 web systemd[1]: Started A high performance web server and a reverse proxy server.
```
```
root@web:/home/vagrant# ss -tln | grep 80
LISTEN 0      511          0.0.0.0:80        0.0.0.0:*
LISTEN 0      511             [::]:80           [::]:*
root@web:/home/vagrant#
```
Настройка центрального сервера логов
```
rsyslog установлен в системе

root@log:/home/vagrant# apt list rsyslog
Listing... Done
rsyslog/jammy-updates,jammy-security,now 8.2112.0-2ubuntu2.2 amd64 [installed,automatic]
N: There is 1 additional version. Please use the '-a' switch to see it
root@log:/home/vagrant#
```
```
Все настройки Rsyslog хранятся в файле /etc/rsyslog.conf 
Для того, чтобы наш сервер мог принимать логи, нам необходимо внести следующие изменения в файл:
```
```
Открываем порт 514 (TCP и UDP):
Находим закомментированные строки:
```
```
# provides UDP syslog reception
#module(load="imudp")
#input(type="imudp" port="514")

# provides TCP syslog reception
#module(load="imtcp")
#input(type="imtcp" port="514")

И приводим их к виду:

# provides UDP syslog reception
module(load="imudp")
input(type="imudp" port="514")

# provides TCP syslog reception
module(load="imtcp")
input(type="imtcp" port="514")

```
В конец файла /etc/rsyslog.conf добавляем правила приёма сообщений от хостов:
```
#Add remote logs
$template RemoteLogs,"/var/log/rsyslog/%HOSTNAME%/%PROGRAMNAME%.log"
*.* ?RemoteLogs
& ~
```
```
Данные параметры будут отправлять в папку /var/log/rsyslog логи, которые будут приходить от других серверов. 
Например, Access-логи nginx от сервера web, будут идти в файл /var/log/rsyslog/web/nginx_access.log
```
```
Далее сохраняем файл и перезапускаем службу rsyslog:

root@log:/home/vagrant# systemctl restart rsyslog

Если ошибок не допущено, то у нас будут видны открытые порты TCP,UDP 514:
рис 1
```
Далее настроим отправку логов с web-сервера

vagrant ssh web

Проверим версию nginx: nginx -v:

root@web:/home/vagrant# nginx -v
nginx version: nginx/1.18.0 (Ubuntu)
```
Версия nginx должна быть 1.7 или выше. В нашем примере используется версия nginx 1.18. 
Находим в файле /etc/nginx/nginx.conf раздел с логами и приводим их к следующему виду:
```
```
root@web:/home/vagrant# vi /etc/nginx/nginx.conf

error_log  /var/log/nginx/error.log;
 error_log  syslog:server=192.168.56.15:514,tag=nginx_error;
 access_log syslog:server=192.168.56.15:514,tag=nginx_access,severity=info combined;
```
рис2
```
```
Для Access-логов указываем удаленный сервер и уровень логов, которые нужно отправлять. Для error_log добавляем удаленный сервер. 
Если требуется чтобы логи хранились локально и отправлялись на удаленный сервер, требуется указать 2 строки. 	
Tag нужен для того, чтобы логи записывались в разные файлы.
По умолчанию, error-логи отправляют логи, которые имеют severity: error, crit, alert и emerg. 
Если требуется хранить или пересылать логи с другим severity, то это также можно указать в настройках nginx. 
```
Далее проверяем, что конфигурация nginx указана правильно: nginx -t:
```
```
root@web:/home/vagrant# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Далее перезапускаем nginx: systemctl restart nginx

```
Попробуем несколько раз зайти по адресу http://192.168.56.10
Далее заходим на log-сервер и смотрим информацию об nginx:
    cat /var/log/rsyslog/web/nginx_access.log 
    cat /var/log/rsyslog/web/nginx_error.log 
```
рис 3
```
Поскольку наше приложение работает без ошибок, файл nginx_error.log не будет создан. 
Чтобы сгенерировать ошибку, можно переместить файл веб-страницы, который открывает nginx - 
```
mv /var/www/html/index.nginx-debian.html /var/www/
```
После этого мы получим 403 ошибку.
Видим, что логи отправляются корректно. 


рис4

Запаковываем в стенд стенд Vagrant + Ansible.






















