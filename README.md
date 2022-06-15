# otus-linux-day07
Управление пакетами. Дистрибуция софта

# **Prerequisite**
- Host OS: Windows 10.0.19043
- Guest OS: CentOS 7.8.2003
- VirtualBox: 6.1.34
- Vagrant: 2.2.19

# **Содержание ДЗ**

* Создание rpm-пакета
* Создание репозитория и публикация ранее созданного rpm-пакета
* Создание docker-контейнера с ранее созданным rpm-пакетом и публикация

# **Выполнение**

### Создание rpm-пакета

Установка требуемых пакетов:
```
yum install -y redhat-lsb-core wget rpmdevtools rpm-build createrepo yum-utils gcc
```
Загрузка SRPM для NGINX и установка в домашний каталог:
```
wget https://nginx.org/packages/centos/7/SRPMS/nginx-1.14.1-1.el7_4.ngx.src.rpm
rpm -i nginx-1.14.1-1.el7_4.ngx.src.rpm
```
Загрузка и распаковка исходников Openssl:
```
wget --no-check-certificate https://www.openssl.org/source/openssl-1.1.1o.tar.gz
tar -xvf openssl-1.1.1o.tar.gz
```
Установка требуемых зависимостей:
```
yum-builddep -y rpmbuild/SPECS/nginx.spec
```
Правка spec для сборки с Openssl:
```
sed -i '/--with-debug/i --with-openssl=\/root\/openssl-1.1.1o \\' rpmbuild/SPECS/nginx.spec
```
Сборка rpm:
```
rpmbuild -bb rpmbuild/SPECS/nginx.spec
```
Установка полученного пакета и запуск nginx:
```
yum localinstall -y rpmbuild/RPMS/x86_64/nginx-1.14.1-1.el7_4.ngx.x86_64.rpm
systemctl start nginx
```

### Создание репозитория и публикация ранее созданного rpm-пакета

Создание каталога для репозитория и копирование туда созданного пакета и дополнительного для примера:
```
mkdir /usr/share/nginx/html/repo
cp rpmbuild/RPMS/x86_64/nginx-1.14.1-1.el7_4.ngx.x86_64.rpm /usr/share/nginx/html/repo/
wget https://repo.percona.com/yum/release/7/RPMS/noarch/percona-release-0.1-6.noarch.rpm -O /usr/share/nginx/html/repo/percona-release-0.1-6.noarch.rpm
```
Инициализация репозитория:
```
createrepo /usr/share/nginx/html/repo/
```
Разрешение вывода листинга каталога с перезапуском nginx:
```
sed -i '/index  index.html index.htm;/a autoindex on;' /etc/nginx/conf.d/default.conf
nginx -s reload
```
Проверка:
```
[root@otus ~]# curl -a http://localhost/repo/
<html>
<head><title>Index of /repo/</title></head>
<body bgcolor="white">
<h1>Index of /repo/</h1><hr><pre><a href="../">../</a>
<a href="repodata/">repodata/</a>                                          13-Jun-2022 17:37                   -
<a href="nginx-1.14.1-1.el7_4.ngx.x86_64.rpm">nginx-1.14.1-1.el7_4.ngx.x86_64.rpm</a>                13-Jun-2022 17:32             2160472
<a href="percona-release-0.1-6.noarch.rpm">percona-release-0.1-6.noarch.rpm</a>                   21-May-2018 14:19               14520
</pre><hr></body>
</html>
```
Прописывание созданного репозитория в системе:
```
cat >> /etc/yum.repos.d/otus.repo << EOF
[otus]
name=otus-linux
baseurl=http://localhost/repo
gpgcheck=0
enabled=1
EOF
```
Просмотр содержимого репозитория, без ключа `--show-duplicates` пакет nginx в вывод не попадёт т.к. уже установлен из локального пакета той же версии:
```
[root@otus ~]# yum repolist enabled | grep otus
otus                            otus-linux                                    2
[root@otus ~]#
[root@otus ~]# yum list --show-duplicates | grep otus
nginx.x86_64                                1:1.14.1-1.el7_4.ngx       otus
percona-release.noarch                      0.1-6                      otus
[root@otus ~]#
[root@otus ~]# yum list | grep nginx
nginx.x86_64                                1:1.14.1-1.el7_4.ngx       @/nginx-1.14.1-1.el7_4.ngx.x86_64
pcp-pmda-nginx.x86_64                       4.3.2-13.el7_9             updates
```


### Создание docker-контейнера с ранее созданным rpm-пакетом и публикация

Установка пакетов Docker и его запуск:
```
yum install -y docker
systemctl enable docker
systemctl start docker
```

Подготовка Dockerfile для создания контейнера на основе centos:7, созданный ранее rpm-пакет nginx устанавливается в контейнер:
```
mkdir docker
ln rpmbuild/RPMS/x86_64/nginx-1.14.1-1.el7_4.ngx.x86_64.rpm docker/nginx-1.14.1-1.el7_4.ngx.x86_64.rpm
cat >> docker/Dockerfile << EOF
FROM centos:7
MAINTAINER Jimi Dini <jimi77.gk@gmail.com>
RUN rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
ADD nginx-1.14.1-1.el7_4.ngx.x86_64.rpm ./nginx-1.14.1-1.el7_4.ngx.x86_64.rpm
RUN yum localinstall -y ./nginx-1.14.1-1.el7_4.ngx.x86_64.rpm
RUN yum clean all
RUN echo "daemon off;" >> /etc/nginx/nginx.conf
RUN sed -i "0,/nginx/s/nginx/docker-nginx/i" /usr/share/nginx/html/index.html
CMD [ "nginx" ]
EOF
```

Создание контейнера:
```
docker build -t jimidini/nginx:v1 docker/
```

Публикация созданного контейнера в DockerHub:
```
docker login --username jimidini
docker tag jimidini/nginx:v1 jimidini/otus-linux-day07:nginx
docker push jimidini/otus-linux-day07:nginx
```
Удаление локального контейнера:
```
docker image rm jimidini/nginx:v1
```
Запуск созданного контейнера:
```
docker run -d -p 8080:80 jimidini/otus-linux-day07:nginx
```
Проверка:
```
[root@otus ~]# ss -tnlp | grep '80'
LISTEN     0      128          *:80                       *:*                   users:(("nginx",pid=10057,fd=6),("nginx",pid=10048,fd=6))
LISTEN     0      128       [::]:8080                  [::]:*                   users:(("docker-proxy-cu",pid=10572,fd=4))
[root@otus ~]#
[root@otus ~]#
[root@otus ~]#
[root@otus ~]# docker ps
CONTAINER ID        IMAGE                             COMMAND             CREATED             STATUS              PORTS                  NAMES
dad93b667c94        jimidini/otus-linux-day07:nginx   "nginx"             7 minutes ago       Up 7 minutes        0.0.0.0:8080->80/tcp   angry_wiles
[root@otus ~]#
[root@otus ~]#
[root@otus ~]# curl -a http://localhost:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to docker-nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
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

# **Результаты**

Полученный в ходе работы `Vagrantfile` с включенным inline shell provisioner и Docker-контейнер помещены в публичные репозитории.
При запуске ВМ создается и устанавливается rpm-пакет nginx c openssl, создаётся локальный репозиторий, разворачивается созданный docker-контейнер из DockerHub

- **GitHub** - https://github.com/jimidini77/otus-linux-day07
- **DockerHub** - https://hub.docker.com/repository/docker/jimidini/otus-linux-day07/
