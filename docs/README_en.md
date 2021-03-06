
![Docker Pulls](https://img.shields.io/docker/pulls/tangramor/docker-tars.svg) ![Docker Automated build](https://img.shields.io/docker/automated/tangramor/docker-tars.svg) ![Docker Build Status](https://img.shields.io/docker/build/tangramor/docker-tars.svg)


TOC
-----

* [Stipulation](#stipulation)
* [MySQL](#mysql)
* [Image](#image)
   * [Notice](#notice)
* [Environment Parameters](#environment-parameters)
   * [TZ](#tz)
   * [DBIP, DBPort, DBUser, DBPassword](#dbip-dbport-dbuser-dbpassword)
   * [MOUNT_DATA](#mount_data)
   * [INET_NAME](#inet_name)
   * [MASTER](#master)
   * [General basic service for framework](#general-basic-service-for-framework)
* [Build Images](#build-images)
* [Use The Image for Development](#use-the-image-for-development)
   * [For Example:](#for-example)
* [Trouble Shooting](#trouble-shooting)
* [Thanks](#thanks)


Stipulation
------------
In this doc, we assume that you are working in **Windows**. Because the command line environment of docker in Windows will map disk driver C:, D: to `/c/` and `/d/`, just like under *nix, we will use `/c/Users/` to describe the User folder in driver C:.


MySQL
-----
This image does **NOT** have MySQL, you can use a docker official image(5.6):
```
docker run --name mysql -e MYSQL_ROOT_PASSWORD=password -d -p 3306:3306 -v /c/Users/<ACCOUNT>/mysql_data:/var/lib/mysql mysql:5.6 --innodb_use_native_aio=0
```

Please be aware of option `--innodb_use_native_aio=0` appended in the command above. Because MySQL aio does not support Windows file system.


If you use a **5.7** MySQL, you may need to add option `--sql_mode=NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION`. Because after 5.6 MySQL does not support zero date field ( https://dev.mysql.com/doc/refman/5.7/en/sql-mode.html#sqlmode_no_zero_date ).
```
docker run --name mysql -e MYSQL_ROOT_PASSWORD=password -d -p 3306:3306 -v /c/Users/<ACCOUNT>/mysql_data:/var/lib/mysql mysql:5.7 --sql_mode=NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION --innodb_use_native_aio=0
```


If use **8.0** MySQL, you need to set `--sql_mode=''`, that will disable the default strict mode ( https://dev.mysql.com/doc/refman/8.0/en/sql-mode.html )

```
docker run --name mysql -e MYSQL_ROOT_PASSWORD=password -d -p 3306:3306 -v /c/Users/<ACCOUNT>/mysql_data:/var/lib/mysql mysql:8 --sql_mode='' --innodb_use_native_aio=0
```


You can also use a customized my.cnf to add those options.


Image
------
The docker image is built automatically by docker hub: https://hub.docker.com/r/tangramor/docker-tars/ . You can pull it by following command:
```
docker pull tangramor/docker-tars
```

The image with **php7** tag supports PHP server development, and it includes php7.2 and phptars extension, as well with MySQL C++ connector for development:
```
docker pull tangramor/docker-tars:php7
```

The image with **php7mysql8** tag supports PHP server development, and it includes php7.2, JDK 10 and mysql8 related support:
```
docker pull tangramor/docker-tars:php7mysql8
```

The image with **minideb** tag is based on minideb which is "a small image based on Debian designed for use in containers":
```
docker pull tangramor/docker-tars:minideb
```

The image with **php7deb** tag is based on minideb, supports PHP server development, and it includes php7.2 and phptars extension, as well with MySQL C++ connector for development:
```
docker pull tangramor/docker-tars:php7deb
```

The image **tars-master** removed Tars source code from the docker-tars image, has same tags as **docker-tars**:
```
docker pull tangramor/tars-master
```

The image **tars-node** has only tarsnode service deployed, and does not have Tars source code either:
```
docker pull tangramor/tars-node
```

### Notice

The docker images are built based on Tars official source code, after the container started, it will launch an automatical installation process because the it need to modify the configurations in the official build according to the container's IP and environment parameters. That will need some minutes, and you may check the resin log `_log4j.log` under `/data/log/tars` to see if resin has started, or you can run `ps -ef` in container to check if all the processes have started.


Environment Parameters
----------------------
### TZ
The timezone definition, default: `Asia/Shanghai`.


### DBIP, DBPort, DBUser, DBPassword
When running the container, you need to set the environment parameters:
```
DBIP mysql
DBPort 3306
DBUser root
DBPassword password
```

### MOUNT_DATA
If you are runing container under **Linux** or **Mac**, you can set the **environment parameter** `MOUNT_DATA` to `true`. This option is used to link the data folders of Tars sub systems to the folers under /data, which we often mount to a external volumn. So even we removed old container and started a new one, with the old data in /data folder and mysql database, our deployments will not lose. That meets the principle "container should be stateless". **BUT** We **CANNOT** use this option under **Windows** because of the [problem of Windows file system and virtualbox](https://discuss.elastic.co/t/filebeat-docker-running-on-windows-not-allowing-application-to-rotate-the-log/89616/11).

### INET_NAME
If you want to expose all the Tars services to the host OS, you can use `--net=host` option when execute docker (the default mode that docker uses is bridge). Here we need to know the ethernet interface name, and if it is not `eth0`, we need to set the **environment parameter** `INET_NAME` to the one that host OS uses, such as `--env INET_NAME=ens160`. Once you started container with this network mode, you can execute `netstat -anop |grep '8080\|10000\|10001' |grep LISTEN` unser host OS to check if these ports are listened correctly.

### MASTER
The tar node server should register itself to the master node. This **environment parameter** `MASTER` is only for **tars-node** docker image, and you should set it to the IP or hostname of the master node.

The command in run_docker_tars.sh is like following, you should modify it accordingly:
```
docker run -d -it --name tars --link mysql --env DBIP=mysql --env DBPort=3306 --env DBUser=root --env DBPassword=PASS -p 8080:8080 -v /c/Users/<ACCOUNT>/tars_data:/data tangramor/docker-tars
```

### General basic service for framework
In the Dockerfile I put the successfully built service packages tarslog.tgz, tarsnotify.tgz, tarsproperty.tgz, tarsqueryproperty.tgz, tarsquerystat.tgz and tarsstat.tgz to /data, which should be mounted from the host machine like `/c/Users/<ACCOUNT>/tars_data/`. These services have been automatically installed in the docker image. You can refer to [Install general basic service for framework](https://github.com/Tencent/Tars/blob/master/Install.en.md#44-install-general-basic-service-for-framework) to understand those services.


Build Images
-------------
Build command: `docker build -t tars .`

Build command for tars-master: `docker build -t tars-master -f tars-master/Dockerfile .`

Build command for tars-node: `docker build -t tars-node -f tars-node/Dockerfile .`

To build image of [tars-master](https://github.com/tangramor/tars-master) , you need to checkout tars-master and run docker build command:

```
git clone https://github.com/tangramor/tars-master.git
cd tars-master
docker build -t tars-master -f Dockerfile .
```


To build image of [tars-node](https://github.com/tangramor/tars-node) , you need to checkout tars-node and run docker build command:

```
git clone https://github.com/tangramor/tars-node.git
cd tars-node
docker build -t tars-node -f Dockerfile .
```


Use The Image for Development
------------------------------
It should be easyer to do Tars related development with the docker image. My way is put the project files under the local folder which will be mounted as /data in the container, such as `/c/Users/<ACCOUNT>/tars_data`. And once you did and works in the project, you can use command `docker exec -it tars bash` to enter Tars environment and execute the compiling or testing works.

### For Example:

**[TARS C++ Server & Client Development](https://github.com/tangramor/docker-tars/wiki/TARS-CPP-Server-&-Client-Development)**

**[TARS PHP TCP Server & Client Development](https://github.com/tangramor/docker-tars/wiki/TARS-PHP-TCP-Server-&-Client-Development)**

**[TARS PHP HTTP Server & Client Development](https://github.com/tangramor/docker-tars/wiki/TARS-PHP-HTTP-Server-&-Client-Development)**



Trouble Shooting
----------------
Once you started up the container, you can enter it by command `docker exec -it tars bash` and then you can execute linux commands to check the status. If you see _log4j.log file under `/c/Users/<ACCOUNT>/tars_data/log/tars`, that means resin is up to work and the installation is done.



Thanks
---------------

The scripts of this image are based on project https://github.com/panjen/docker-tars, which is from https://github.com/luocheng812/docker_tars.
