---
title: "在 internet.nl 上拿到满分"
date: 2023-02-13T17:05:14+08:00
description: "但是真的那么有用吗？"
tags: [web,self-host]
featured_image: "featured_image.webp"
draft: false
hidden: false
---

[Internet.nl](https://internet.nl) 是一个由网络社区和荷兰政府共同创立的项目。这个项目提供对网站，电子邮件和网络连接的测试，以便查看他们是否遵循了现代而可靠的技术/协议。 [Source](https://internet.nl/about/)

测试分为三个部分：网站，邮件和网络连接。网站，邮局测试中达到满分的域名会被加入他们的“名人堂”列表，双满分则更为稀少。网络连接测试不会进行研究，原因会在后文解释。


# 网站测试

此部分使用 nginx 为例，版本号 1.22.1。

对于一些项目，可能需要更改的配置文件有
- /etc/nginx/nginx.conf
- /etc/nginx/sites-available/example.com (or /default)

如果你在使用 Certbot，则还可能需要更改
- /etc/letsencrypt/options-ssl-nginx.conf

## Modern address (IPv6)

分为三部分：
- Nameserver 是否有 IPv6 地址，且是否可达（正确）。
- 网站是否有 IPv6 地址，且是否可达（正确）。
- IPv6 连接下的网站是否和 IPv4 连接下的网站相同。

一般来说是能很容易满足的条件。

## Signed domain name (DNSSEC)

[DNSSEC](https://zh.wikipedia.org/zh-cn/%E5%9F%9F%E5%90%8D%E7%B3%BB%E7%BB%9F%E5%AE%89%E5%85%A8%E6%89%A9%E5%B1%95) 提供了对 DNS 数据来源的验证，在自己的域名注册商处设置即可。

## Secure connection (HTTPS)

简单设置了 HTTPS 并不能在这里得到满分。

一个很有用的网站，由 mozilla 提供，可生成众多配置文件：[ssl-config.mozilla.org](https://ssl-config.mozilla.org)

### HTTPS available & HTTPS redirect

网站启用 HTTPS 连接与 HTTP 重定向即可。Nginx 模板如下

    server {
        listen 80;
        listen [::]:80;
        server_name example.com www.example.com;
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl http2; 
        listen [::]:443 ssl http2;
        server_name example.com www.example.com

        ssl_certificate /etc/ssl/certs/example.com/fullchain.pem;
        ssl_certificate_key /etc/ssl/certs/example.com/privkey.pem;
    }

### HTTP compression

HTTP 压缩可能使服务器遭受 BREACH 攻击，启用与否是对流量/带宽和安全性的取舍。

    gzip off;

### HSTS

HSTS 将在浏览器再次访问网站时强制使用 HTTPS 连接，这有助于预防中间人攻击。Internel.nl 认为 HSTS 策略缓存时间为***至少***一年时，策略足够安全。

在 nginx 配置中添加以下语句

    add_header Strict-Transport-Security "max-age=31536000" always;


参照 internet.nl 的提示，只有当确认**全部**子域名都被 HTTPS 覆盖到时，才应该添加 ```includeSubDomains``` 语句，即

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

### TLS version

TLS 版本安全性表格：
- 优秀：TLS 1.3
- 足够：TLS 1.2
- 换代：TLS 1.1,1.0
- 不足：SSL 3.0,2.0,1.0

在 server 块中加入以下内容：

    ssl_protocols TLSv1.2 TLSv1.3;

同时，如果使用了 Cloudflare CDN，我还需要在 Cloudflare 的面板中调整以下设置

![](images/cloudflare_tls_version.webp)

### Ciphers (Algorithm selections)

很多算法已经到了 phase-out 阶段，可以在 nginx 配置文件中禁用。

    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;

### Cipher order

在 internet.nl 的列表中列出了 good, sufficient 和 phase out 列表。服务器应该：
- 若只支持 good 列表中的内容，则不参加此测试
- 若同时支持 good 列表外的内容，应优先使用 good cipher

#### ssl_prefer_server_ciphers

Nginx 推荐此项设置为 on，而 mozilla 推荐设置为 off。

相关讨论可参考此处：[https://serverfault.com/questions/997614/setting-ssl-prefer-server-ciphers-directive-in-nginx-config](https://serverfault.com/questions/997614/setting-ssl-prefer-server-ciphers-directive-in-nginx-config)

我将其设置为了 off。

### Key exchange parameters

[Diffie-Hellman key exchange](https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange) 是一种安全协议。服务端应支持足够安全的交换参数。

Github 上的 [internetstandards/dhe_roups](https://github.com/internetstandards/dhe_groups) 仓库给出了足够安全的 ffdhe4096 参数。下载后在 nginx 中指定：
    ssl_dhparam /etc/nginx/ffdhe4096.pem;

### OCSP stapling

OCSP stapling 将本应客户端发起的 OCSP 请求交给服务端发起，服务端将 OCSP 结果随证书一同发送给客户端，跳过了客户自己请求 OCSP 的过程，提高了效率。

    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /path/to/fullchain;

### Certificate 相关

证书相关，请求了证书并启用即可。

### DANE

此处不强求，故不启用。若有兴趣，方法于邮件处的 DANE 设置相同。

## Security options

### X-Frame-Options

X-Frame-Options 用于避免 [clickjacking 攻击](https://owasp.org/www-community/attacks/Clickjacking)。

相关选项语法为：

    add_header X-Frame-Options "OPTION";

其中 OPTION 可选择 SAMEORIGIN, DENY 和 ALLOW-FROM URI。前两项被认为足够安全。  
此处设置为

    add_header X-Frame-Options "SAMEORIGIN"

### X-Content-Type-Options

X-Content-Type-Options 用于指明 MIME 类型，同时避免 [MIME 类型嗅探](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/MIME_types#mime_sniffing)。

    add_header X-Content-Type-Options nosniff;

### Content-Security-Policy

按照 internet.nl 推荐，应至少有以下内容：
- default-src: none/self/https:(不推荐)
- frame-src: none/self/url
- frame-ancestors: hone/self/url

其他项目可在 internet.nl 的测试项中查看细节，或在 [content-security-policy.com](https://content-security-policy.com) 查看规范。

    add_header Content-Security-Policy "default-src 'self';frame-src 'self';frame-ancestors 'self';" always;

### Referrer-Policy existence

    add_header Referrer-Policy 'strict-origin-when-cross-origin';

推荐值：
- no-referrer/same-origin 如果不发送敏感信息到第三方。
- strict-origin/strict-origin-when-cross-origin 如果只通过 HTTPS 发送敏感信息到第三方。


### Security.txt

可以通过 [securitytxt.org](https://securitytxt.org/) 生成 security.txt，并放置在 domain.com/security.txt 和/或 domain.com/.well-known/security.txt

## Route authorisation (RPKI)

BGP 相关。

## Nginx配置总和

不推荐直接抄，放这里只是为了方便我自己看。

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
        root /var/www/example.com;

        ssl_certificate /etc/ssl/certs/example.com/fullchain.pem;
        ssl_certificate_key /etc/ssl/certs/example.com/privkey.pem;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
        ssl_prefer_server_ciphers off;
        ssl_stapling on;
        ssl_stapling_verify on;
        ssl_trusted_certificate /etc/ssl/certs/example.com/fullchain.pem;
        ssl_dhparam /etc/nginx/ffdhe4096.pem;
        add_header Strict-Transport-Security "max-age=31536000" always;
        add_header X-Frame-Options "SAMEORIGIN";
        add_header X-Content-Type-Options nosniff;
        add_header Referrer-Policy 'strict-origin-when-cross-origin';

        index index.html index.xml;

        location / {
                index index.html;
        }
    }

# 邮局测试

此处使用 [mailcow](https://mailcow.email) 为基础进行设置。

## Modern address (IPv6) & Signed domain names (DNSSEC)

与上文相同。

## Authenticity marks against phishing (DMARC, DKIM and SPF)

### DMARC

DMARC 策略用于指明当一封邮件不能被 DKIM 和 SPF 认证时的处理方式。

参考： [dmarc.org](https://dmarc.org/)。

### DKIM

不同平台配置方法不同，得到数据后添加至 DNS 记录即可。

### SPF

SPF 用于验证邮件发送人的真实性。参考 Cloudflare 就此的文章：[什么是 DNS SPF 记录？](https://www.cloudflare.com/zh-cn/learning/dns/dns-records/dns-spf-record/)。

## Secure mail server connection (STARTTLS and DANE)

### TLS

简而言之就是对 postfix 进行的设置。`/opt/mailcow-dockerized/data/conf/postfix/extra.cf` 内容如下

    tls_ssl_options = NO_COMPRESSION, 0x40000000
    tls_preempt_cipherlist = yes
    smtpd_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
    smtpd_tls_mandatory_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
    smtpd_tls_ciphers = high
    smtpd_tls_mandatory_ciphers = high
    smtpd_tls_exclude_ciphers = EXP, LOW, MEDIUM, aNULL, eNULL, SRP, PSK, kDH, DH, kRSA, DHE, DSS, RC4, DES, IDEA, SEED, ARIA, CAMELLIA, AESCCM8, 3DES, ECDHE-ECDSA-AES256-SHA>
    smtpd_tls_eecdh_grade = ultra
    smtpd_tls_dh1024_param_file = /opt/mailcow-dockerized/data/assets/ssl/ffdhe4096.pem
    smtp_host_lookup = dns
    smtp_tls_note_starttls_offer = yes
    smtp_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
    smtp_tls_mandatory_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
    smtp_tls_ciphers = high
    smtp_tls_mandatory_ciphers = high
    smtp_tls_exclude_ciphers = EXP, LOW, MEDIUM, aNULL, eNULL, SRP, PSK, kDH, DH, kRSA, DHE, DSS, RC4, DES, IDEA, SEED, ARIA, CAMELLIA, AESCCM8, 3DES, ECDHE-ECDSA-AES256-SHA3>
    lmtp_tls_note_starttls_offer = yes
    lmtp_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
    lmtp_tls_mandatory_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
    lmtp_tls_ciphers = high
    lmtp_tls_mandatory_ciphers = high
    lmtp_tls_exclude_ciphers = EXP, LOW, MEDIUM, aNULL, eNULL, SRP, PSK, kDH, DH, kRSA, DHE, DSS, RC4, DES, IDEA, SEED, ARIA, CAMELLIA, AESCCM8, 3DES, ECDHE-ECDSA-AES256-SHA3>
    lmtp_tls_loglevel = 1
    lmtp_tls_session_cache_database = btree:/var/lib/postfix/lmtp_tls_session_cache

对每一项具体内容感兴趣可自行研究。

### Certificate

证书相关，mailcow 帮我生成了自签名证书，此处自动解决。

### DANE

提示：DNSSEC 为 DANE 设置的前提。

DANE 用于尽可能减少邮件传输时可能的内容窥探。可参考 [Hands-on: implementing DANE in Postfix](https://www.sidn.nl/en/news-and-blogs/hands-on-implementing-dane-in-postfix)。

启用过程请参考 [DANE for SMTP how-to](https://github.com/internetstandards/toolbox-wiki/blob/main/DANE-for-SMTP-how-to.md)。

懒人版：

需两个数值，自签证书的 DANE SHA-256 hash，和 root certificate 的 DANE SHA-256 hash。

对于 mailcow，自签证书位于 `/mailcow-folder/data/assets/ssl/cert.pem`；  
根因使用 Let's Encrypt，根证书为 ISRG Root X1，位于 `/etc/ssl/certs/ISRG_Root_X1.pem`

    # openssl x509 -in /opt/mailcow-dockerized/data/assets/ssl/cert.pem -noout -pubkey | openssl pkey -pubin -outform DER | openssl sha256
    (stdin)= example_output_1
    # openssl x509 -in /etc/ssl/certs/ISRG_Root_X1.pem -noout -pubkey | openssl pkey -pubin -outform DER | openssl sha256
    (stdin)= example_output_2

随后需要在 DNS 中声明这两项结果

    _25._tcp.mail.example.com. IN TLSA 3 1 1 example_output_1
    _25._tcp.mail.example.com. IN TLSA 2 1 1 example_output_2

数值的具体意义可查看上述参考内容。

## Route authorisation (RPKI)

同上，BGP 大佬受我一拜。

# 网络连接测试

基本就是测试本地是否能连接 IPv6 网络。但从国内连接的效果基本没有意义，如果真的在意不如使用 [testipv6.cn](https://testipv6.cn/)。

这么多内容调整完了。问题是：有多少会产生实际影响？

~~当然在于玩的开心~~