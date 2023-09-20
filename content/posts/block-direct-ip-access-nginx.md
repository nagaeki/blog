---
title: "Nginx 禁用源站 IP 对网站的访问"
date: 2022-12-27T23:26:06+08:00
description: "关于群友忘了这件事然后机器人被找到源站被打这件事。"
draft: false
tags: [self-host,web]
featured_image: "/images/posts/block-direct-ip-access-nginx/index.webp"
---

对于这个问题有三种解决方法：
1. 对于Nginx 1.19.4与更新，使用ssl_reject_handshake；
2. 对于较老版本，使用自签SSL证书；
3. 在server块中用if语句判断。

三种方法对 HTTP, HTTPS 都适用（为什么不呢）。

# Update 2023/09/19

Configuration snippets as well as automatic installer script available on [my Github](https://github.com/nagaeki/nginx-config)

# ssl_reject_handshake 方法

## 安装可用的Nginx版本

首先需要确定安装的是Nginx 1.19.4及以上版本，可使用 `nginx -v 命令查看。

处理此问题时Debian Stable仍在发行1.18.0版本，因此使用Nginx官网源安装新版。参考 [Nginx官网安装步骤](https://www.nginx.com/resources/wiki/start/topics/tutorials/install/)。

获得 nginx 签名密钥并添加信任

    wget https://nginx.org/keys/nginx_signing.key 
    apt-key add nginx_signing.key

添加 nginx 源

    >/etc/apt/sources.list.d/nginx.conf
    deb https://nginx.org/packages/mainline/debian/ bullseye nginx
    deb-src https://nginx.org/packages/mainline/debian bullseye nginx

如果需要卸载原有的 nginx 安装

    apt purge nginx nginx-common

随后安装并重启 nginx 服务即可。

## 启用default配置

在 nginx 的配置文件夹中写一个default文件，我选择位于 `/etc/nginx/conf.d/default.conf` ，根据习惯即可。

    server {
        listen 80 default_server;
        listen [::]:80 default_server;
 
        server_name _;
        return 444;
    }

解释：server_name _ 针对一切没有在本机匹配的域名，444响应让Nginx直接不响应内容。

记得启用此配置并刷新Nginx。

## 针对HTTPS

以上内容只针对HTTP请求有效（也只监听了80端口）。为了对HTTPS请求也生效，增加文件至以下内容：

    server {
        listen 80 default_server;
        listen [::]:80 default_server;

        listen 443 default_server;
        listen [::]:443 default_server;
        ssl_reject_handshake on;

        server_name _;
        return 444;
    }

这样处理后，HTTPS请求到源站后不会露出证书。 **这是我选择的方法。** 效果如图，不同浏览器报错不同。

![](/images/posts/block-direct-ip-access-nginx/ssl_error.webp)

# 自签SSL证书方法

如果你在较老版本的Nginx上使用了上面的方法，你很有可能会遇到以下报错：

     nginx: [emerg] no "ssl_certificate" is defined for the "listen ... ssl" directive

如果你又不想更新Nginx，则可以使用自签SSL证书方法。

## 获得空白SSL证书

可以使用以下命令获得 `ssl_certificate` 和 `ssl_certificate_key`

     openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout default.key -out default.crt -subj '/CN='

这样会直接把 default.key 和 default.crt 输出在当前目录下。

## 更新Nginx配置

随后可以把 Nginx 的 default 配置改为如下：

    server {
        listen 80 default_server;
        listen [::]:80 default_server;
        
        listen 443 default_server;
        listen [::]:443 default_server;
 
        ssl_certificate /etc/nginx/ssl/default.crt;
        ssl_certificate_key /etc/nginx/ssl/default.key;
 
        server_name _;
        return 444;
    }

重载配置后再次通过IP访问应该不会得到结果。

# IF语句判断方法

对于某个域名的配置，写入以下内容：

    server {
        listen 443 default_server;
        listen [::]:443 default_server;
 
        if ($host != eki.moe) {
                return 444;
        }
 
        server_name eki.moe;
 
    ...剩余内容

判断语句可以搭配正则表达式。但这种方法缺点太多了：

1. 工作量大；
2. 容易出错；
3. 难以扩展；
4. 上面两种方法好太多了。

# 本文参考

1. https://stackoverflow.com/questions/29104943/how-to-disable-direct-access-to-a-web-site-by-ip-address
2. https://erikpoehler.com/2022/08/02/how-to-block-direct-ip-access-to-your-nginx-web-server/
3. https://www.codedodle.com/disable-direct-ip-access-nginx.html
4. https://www.nginx.com/resources/wiki/start/topics/tutorials/install/