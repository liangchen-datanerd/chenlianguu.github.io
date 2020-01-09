---
title: Docker部署Nextcloud
top: false
toc: true
mathjax: true
date: 2020-01-09 11:54:09
index_img: https://i.loli.net/2020/01/09/6Wies2vuV8bdrLm.png
summary:
tags:
 - docker
 - nextcloud
 - 网盘
 - 部署
categories: 运维
---

## 准备工作

### docker的安装及配置

```shell
# 通过yum安装
yum install -y docker-ce
# 启动docker并设置开机启动
systemctl start docker
systemctl enable docker
# aws ec2 linux2另外一种安装方式
sudo amazon-linux-extras install docker
sudo service docker start
```

由于docker镜像源默认是在国外，拉取镜像速度非常慢，修改daemon配置文件/etc/docker/daemon.json来使用加速器

```shell
mkdir -p /etc/docker
touch /etc/docker/daemon.json
vim /etc/docker/daemon.json
{
  "registry-mirrors": ["https://b3sst9pc.mirror.aliyuncs.com"]
}
systemctl daemon-reload
systemctl restart docker
```

### docker-compose安装

Docker Compose是 docker 提供的一个命令行工具，用来定义和运行由多个容器组成的应用。使用 compose，我们可以通过 YAML 文件声明式的定义应用程序的各个服务，并由单个命令完成应用的创建和启动。

- docker-compose命令安装

  ```shell
  # 安装pip
  yum -y install epel-release
  yum -y install python-pip
  # 确认版本
  pip --version
  # 更新pip
  pip install --upgrade pip
  # 安装docker-compose
  pip install docker-compose 
  # 查看版本
  docker-compose version
  ```

## nextcloud的部署

nextcloud通过docker-compose命令进行构建，创建 docker-compose.yml文件

```shell
# 创建docker-compose.yml文件
touch docker-compose.yml
# 粘贴以下内容
version: '2'
services:
  db:
    image: mariadb
    restart: always
    volumes:
      - /data/mariadb:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=root
      - MYSQL_PASSWORD=nextcloud
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud

  app:
    image: nextcloud
    restart: always
    ports:
      - 8091:80
    links:
      - db
    volumes:
      - /data/nextcloud/data:/var/www/html/data    
      - /data/nextcloud/themes:/var/www/html/themes
      - /data/nextcloud/apps:/var/www/html/custom_apps
```

docker-compose.yml文件说明

通过启动两个service服务mariadb及nextcloud服务，并通过link连接到一起，其中将/var/www/html/data、/var/www/html/themes、/var/www/html/custom_apps映射到linux服务器指定目录实现数据持久化，这样下次docker重启的时候数据不会发生丢失。

nextcloud实现预览编辑office文件需要插件onlyoffice支持，通过docker创建onlyoffice服务器，并启动nextcloud配置onlyoffice

```shell
docker run -it -d -p 8061:80 onlyoffice/documentserver  -v /data/onlyoffice/logs:/var/log/onlyoffice /data/onlyoffice/data:/var/www/onlyoffice/Data /data/onlyoffice/lib:/var/lib/onlyoffice /data/onlyoffice/db:/var/lib/postgresql
```

启动nextcloud、停止nextcloud、查看nextcloud启动状态

```shell
# 启动nextcloud
docker-compose up -d
# 查看nextcloud状态
docker-compose ps
# 停止nextcloud
docker-compose stop
```

## nextcloud的数据迁移

当需要nextcloud迁移到另外一台服务的时候，需要将nextcloud持久化的数据通过scp或者其他方式复制到另外一台机器，将原来的机器mariadb的nextcloud数据库进行导出，在新的机器上面导入数据库数据。复制YAML文件到新的机器，通过docker-compose进行启动，进入容器内部修改持久化数据的权限及修改nextcloud配置文件，最后重启容器。

```shell
# 复制持久化数据到新的机器上面
scp -R /data/nextcloud root@192.168.12.1:/data
# 导出原来机器的上面的mariadb数据
docker exec -it  nextcloud_db_1【docker容器名称/ID】 mysqldump -uroot -proot【数据库密码】 nextcloud【数据库名称】 > /opt/sql_bak/nextcloud.sql【导出表格路径】
# 将sql文件复制到新的机器上面
scp  /opt/sql_bak/nextcloud.sql root@192.168.12.1:/opt/sql_bak
# 在新的机器上面执行该sql文件(需要先启动docker容器)
docker cp /opt/sql_bak/nextcloud.sql 【容器名】:/root/
docker exec -it 【容器名/ID】sh
mysql -uroot -p nextcloud【数据库名】 < /root/nextcloud.sql
# 通过docker-compose启动镜像，修改权限
 docker exec -it -u root nextcloud_app_1【容器id/容器名】 /bin/bash
 root@bafc02ce112a:/var/www/html# chown -R www-data:root /var/www/html/
 root@bafc02ce112a:exit
# 修改nextcloud配置文件
vim /data/nextcloud/config/config.php
# 修改ip
'trusted_domains' =>
  array (
    0 => '161.189.27.8:8091',
  ),
  'datadirectory' => '/var/www/html/data',
  'dbtype' => 'mysql',
  'version' => '16.0.4.1',
  'overwrite.cli.url' => 'http://161.189.27.8:8091',
# 重启
docker-compose restart
```

## trouble shooting

- 安装docker-compose命令执行pip install docker-compose的时候报以下错误

  ```
  ERROR: Cannot uninstall 'requests'. It is a distutils installed project and thus we cannot accurately determine which files belong to it which would lead to only a partial uninstall.
  ```

  解决版本，强制安装requests包

  ```shell
   pip install --ignore-installed requests
  ```

  再重新执行

  ```shell
  pip install docker-compose
  ```

- docker-compose报错：No module named ssl_match_hostname

  ```
  File "/usr/local/lib/python2.7/dist-packages/docker/transport/ssladapter.py", line 23, in <module>
  from backports.ssl_match_hostname import match_hostname
  ImportError: No module named ssl_match_hostname
  ```

  原因：

  **/usr/local/lib/python2.7/distpackages/docker/transport/ssladapter.py **
  在包路径下找不到 **backports包里的ssl_match_hostname**模块

  解决办法

  ```shell
  #进入backports模块路径
  cd /usr/lib/python2.7/site-packages
  #复制整个包到transport包路径下
  cp -r backports /usr/lib/python2.7/site-packages/docker/transport
  ```
