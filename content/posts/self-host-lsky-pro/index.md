---
title: "部署 Lsky-Pro 自建图床搭配 nsfwjs 自动审核"
date: 2022-12-14T19:54:08+08:00
description: "为朋友的女装找一个新家。"
tags: [self-host,web]
featured_image: "featured_image.webp"
draft: false
hidden: false
---

为什么要自建图床？~~因为我机器在吃灰（逃~~ 因为要给朋友一起用。

选用的图床平台为 [Lsky-Pro](https://www.lsky.pro/)（GitHub仓库：[lsky-org/lsky-pro](https://github.com/lsky-org/lsky-pro)），对我而言功能足够多，且符合我对美观和实用的要求。  
特性（From [官方文档](https://docs.lsky.pro/docs/free/v2/#%E7%89%B9%E6%80%A7)）：
- 支持本地等多种第三方云储存 AWS S3、阿里云 OSS、腾讯云 COS、七牛云、又拍云、SFTP、FTP、WebDav、Minio
- 多种数据库驱动支持，MySQL 5.7+、PostgreSQL 9.6+、SQLite 3.8.8+、SQL Server 2017+
- 支持配置使用多种缓存驱动，Memcached、Redis、DynamoDB、等其他关系型数据库，默认以文件的方式缓存
- 多图上传、拖拽上传、粘贴上传、动态设置策略上传、复制、一键复制链接
- 强大的图片管理功能，瀑布流展示，支持鼠标右键、单选多选、重命名等操作
- 自由度极高的角色组配置，可以为每个组配置多个储存策略，同时储存策略可以配置多个角色组
- 可针对角色组设置上传文件、文件夹路径命名规则、上传频率限制、图片审核等功能
- 支持图片水印、文字水印、水印平铺、设置水印位置、X/y 轴偏移量设置、旋转角度等
- 支持通过接口上传、管理图片、管理相册
- 支持在线增量更新、跨版本更新
- 图片广场

可以在[官方文档](https://docs.lsky.pro/)中了解到更多信息。

# 安装前准备

Lsky-Pro需要的组件为（From [官方文档](https://docs.lsky.pro/docs/free/v2/#%E5%AE%89%E8%A3%85%E8%A6%81%E6%B1%82)）：
- PHP >= 8.0.2
- BCMath PHP 扩展
- Ctype PHP 扩展
- DOM PHP 拓展
- Fileinfo PHP 扩展
- JSON PHP 扩展
- Mbstring PHP 扩展
- OpenSSL PHP 扩展
- PDO PHP 扩展
- Tokenizer PHP 扩展
- XML PHP 扩展
- Imagick 拓展
- exec、shell_exec 函数
- readlink、symlink 函数
- putenv、getenv 函数

支持的数据库（From [官方文档](https://docs.lsky.pro/docs/free/v2/#%E6%94%AF%E6%8C%81%E7%9A%84%E6%95%B0%E6%8D%AE%E5%BA%93)）：
- Mysql 5.7+
- PostgreSQL 9.6+
- SQLite 3.8.8+
- SQL Server 2017+

## 配置PHP

需要注意的是：在使用PHP8.2测试时会报HTTP 500错误，使用8.1版本正常，故使用8.1版继续。

获取PHP源（如果当前系统不存在）：  
[Frequently Asked Questions · oerdnj/deb.sury.org Wiki (github.com)](https://github.com/oerdnj/deb.sury.org/wiki/Frequently-Asked-Questions#debian)  

    curl -sSL https://packages.sury.org/php/README.txt | bash -x
    sudo apt update

安装Lsky-Pro需求组件：

    apt install php8.1 php8.1-cli php8.1-common php8.1-fpm php8.1-xml php8.1-curl php8.1-mysql php8.1-sqlite3 php8.1-mbstring php8.1-gd php8.1-fileinfo php8.1-exif php8.1-bcmath php8.1-imagick

可以创建index.php并包含

    <?php
    phpinfo();

来获得当前PHP可用的信息，并与Lsky-Pro的需求进行对照。

随后对PHP-FPM设置进行一些调整；我的配置文件位于 /etc/php/8.1/fpm/php.ini，以您的安装为准：
- upload_max_filesize = 5M 单个文件上传的最大大小。
- post_max_size = 10M 一次POST队列（即一次上传）的最大大小。
- max_execution_time = 300 最大执行时间。当上传超过此时间时，上传会被终止。以秒为单位。
- memory_limit = 128M 使用的最大内存，取决于你的用途和机器配置。
- 注释掉open_basedir（即对本行添加分号）。

保存后重启php8.1-fpm服务刷新配置文件。

## 配置mysql

虽然sqlite可能更lite ~~（废话）~~ ，但我现有服务均使用了mysql/mariadb，故直接使用mysql作为数据库。

    MariaDB [(none)]> create database lsky_pro;
    Query OK, 1 row affected (0.001 sec)
 
    MariaDB [(none)]> grant all on lsky_pro.* to lsky_pro_user@localhost identified by 'yourpasswordhere';
    Query OK, 0 rows affected (0.004 sec)
 
 
    MariaDB [(none)]> flush privileges;
    Query OK, 0 rows affected (0.002 sec)
 
    MariaDB [(none)]> exit;
    Bye

# 安装Lsky-Pro本体

首先从GitHub下载最新的release版本，本文写作时为2.1，并解压至一个喜欢的位置。我放在了`/var/www/img.example.com/`。  
解压并修改文件权限：

    wget https://github.com/lsky-org/lsky-pro/releases/download/2.1/lsky-pro-2.1.zip
    unzip lsky-pro-2.1.zip
    chmod -R 755 /var/www/example.com
    chown -R www-data /var/www/example.com

设置nginx反代：

    server {
        listen 80;
        listen [::]:80;
        server_name example.com;
        return 301 https://$server_name$request_uri;
    }
 
    server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        server_name example.com;
        ssl_certificate /dir/to/your/fullchain.pem;
        ssl_certificate_key /dir/to/your/privkey.pem;
        client_max_body_size 100M; #最大数据大小
        root /var/www/example.com/public/; #注意指定至public文件夹
 
        index index.php;
        location / {
                try_files $uri $uri/ /index.php?$query_string;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
                proxy_set_header REMOTE-HOST $remote_addr;
        }
        location ~ \.php$ {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/run/php/php8.1-fpm.sock;
        }
        location ~ /\.ht {
                deny all;
        }
    }

随后浏览器访问域名即可安装。

# 通过nsfwjs实现图片自动审核

原仓库：[infinitered/nsfwjs](https://github.com/infinitered/nsfwjs)  
此处利用Docker：[penndu/nsfw-api](https://dusays.com/495/)

可以通过作者原文直接docker run运行，但我喜欢docker-compose。docker-compose.yml如下：

    version: '3'
    services:
      nsfw-api:
        image: penndu/nsfw-api:latest
        restart: unless-stopped
        hostname: nsfw-api
        container_name: nsfw-api
        ports:
          - "3000:3000"

切换左侧数字为宿主机上空闲的端口号即可使用，此时请求地址为 `http://ip:3000/classify`。

也可继续搭配nginx反代端口：

    server {
        listen 80;
        listen [::]:80;
        server_name api.example.com;
        return 301 https://$server_name$request_uri;
    }
 
    server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        server_name api.example.com;
        ssl_certificate /etc/ssl/certs/api.example.com/fullchain.pem;
        ssl_certificate_key /etc/ssl/certs/api.example.com/privkey.pem;
        client_max_body_size 100M;
        location / {
                proxy_pass http://127.0.0.1:3000/classify;
                proxy_redirect     off;
                proxy_set_header   Host             $host;
                proxy_set_header   X-Real-IP        $remote_addr;
                proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        }
    }

此时直接使用根地址即可使用。