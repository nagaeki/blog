---
title: "Blocking Direct IP Access to Nginx"
date: 2022-12-27T23:26:06+08:00
description: "For the time a friend forgetting this and got DDoSed"
tags: [self-host,web]
featured_image: "featured_image.webp"
draft: false
hidden: false
---

# Updates

## 2026/06/19

>I have mainly rewritten this post, both to reflect the changes in software solutions, as well as to provide more understandable language and clearer instructions.

## 2023/09/19

>Configuration snippets as well as automatic installer script available on [my Github](https://github.com/nagaeki/nginx-config)

> Note: This post uses "SSL handshake" and "TLS handshake" interchangeably.

# The problem

I don't remember clearly when it started, but now it seems to be common practice to use Cloudflare's services for your websites. A lot of my friends do the same thing. We would add the domain to Cloudflare's DNS, add records, and toggle the small orange cloud for CDN and protection. It even warns to when you forget to do so! ~Thanks Cloudflare!~

Jokes aside, nowadays it is getting even more important to protect your source servers from attacks such as DDoS. ~Which is why at the time of writing this post, the whole blog is hosted on Cloudflare pages so I don't have to deal with it. Thanks again!~ Even if you chose to use a static site [(ref: other post)](/posts/why-im-transitioning-to-static-websites), the probability of you needing to run a dynamic service on your servers with Nginx is not low, especially if you like to tinker with things just like me.

What you might not have realized, is that when putting a nginx server behind CDNs, although access from HTTP/DNS is routed through the CDN service, it is still possible to gain knowledge about what the server is serving via its IP address.

Try this for yourself. Open `http://SERVER_IP_ADDRESS` or `https://SERVER_IP_ADDRESS`, and see if one of the websites hosted by that reverse proxy shows up. Although this might change with time due to changes in default configuration, it did for my site at time of writing this post.

# The reason

> The following part may be inaccurate, due to my unfamiliarity with HTTP and the TLS handshake process.

The site nginx chooses to server is tied to the handshaking process between your browser and the Nginx backend, and there is some difference between acessing via HTTP and HTTPS.

It is quite simple with HTTP, as it serves the **default HTTP server**, marked with `default_server` in the configuration files. If none of the server blocks are marked as such, nginx chooses the first server block it reads from configuration files as the default.

With HTTPS, the server is provided with much more information from the client browser. A common sequence goes as follows.

- The clients initializes a connection with TLS.
- If a SNI (Server Name Indication) is provided in the handshake, nginx chooses the server that said name.
- If the above fails, the connection is made with the default server.

Therefore when connecting via the IP address from a browser, the SNI is typically sent as the IP address. If a TLS certificate with said IP is present, it will be served, however I do believe that is rare considering the cost of IP certificates. If not, the certificate of the default server will be served.

> This might change from 2026, as letsencrypt seems to be starting to sign IP certs.

Note how this might be an issue. No matter with either HTTP or HTTPS, it is possible to know one of the servers hosted on that IP address, either by the served content, or by the TLS certificate, which makes it a possible target for DDoS attacks. With HTTPS, because the TLS handshake sends the server's certificate, a leak of the server's hosted content can be inferred from the certificate, even if no HTTP requests are sent. Although not yet true for IPv6, the entirety of IPv4 is constantly being scanned by both threat actors and companies in the name of research. ~Yeah the internet is supposed to be neutral, but still nobody likes it.~ You can try to search for what each IP used to serve on some public databases these companies publish.

So the solution is simple, as we just need to block access to the default server. There are mainly three methods to do so, and all of them works for both HTTP and HTTPS. ~(Of course they do, because there's no reason for them to not to)~

1. For nginx version 1.19.4 are newer, use the native ssl_reject_handshake.
2. For older versions, self sign an SSL certificate.
3. Use an if block inside server blocks to identify and block such traffic.

> As of 2026, there have been serious CVEs found in a large amount of software by newly released security research LLMs, with multiple of them leading to gaining root and remote code excution, including both the Linux kernel and nginx. So realistically the version you should be using contains the patch to work with the first solution.

# Native ssl_reject_handshake method

## Installing an up to date version

For this method to work nginx should be above version 1.19.4, which can be confirmed by the command `nginx -v`. If it is not, consider either updating to your distrubution's latest version, or installing the mainline branch from nginx themselves, which is explained in [this official guide](https://www.nginx.com/resources/wiki/start/topics/tutorials/install/).

> As stated above, you really should be using something much higher than version 1.19.4.

## Enabling the default server

To avoid any random server being assigned to be the default server, we simply need to create a default server by ourselves. My default server is located in `/etc/nginx/nginx.conf`, as listed in my GitHub repository.

```
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    server_name _;
    return 308 https://$host$request_uri;
}

server {
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;
    listen 443 quic reuseport default_server;
    listen [::]:443 quic reuseport default_server;
    ssl_reject_handshake on;

    server_name _;
    return 444;
    }
```

This server block does a few things. 

- It firstly listens on both IPv4 and IPv6 on all addresses as the default server, maching all server names (`server_name _`), and HTTP 308 redirects them to the HTTPS version. (308 is a bit different and newer than 301 and 302, however I chose it due to it being a more sensible request, and also because the rest of my config in already very aggressive, so I'm not worried about it not being compatible with older software.)
- For HTTPS it serves both HTTP2 and HTTP3 on both IPv4 and IPv6, matches all servers, rejects the SSL handshake so the certificate is not served, and returns HTTP 444 no response, which is the most fitting response for this.
- Again, it is important to reject the SSL handshake, because it happens even before HTTP requests.

Again, there are other ways and settings to achieve the same result, but this is what I chose. Remember to reconfigure nginx to use the updated configuration files.

![HTTP 444](./images/444.webp)

> The sections below is **not maintained**, for reasons stated above.

# Self signed certificate method

If you attempt to use the above solution with a nginx instance that's too old, you would get the following error.

```
nginx: [emerg] no "ssl_certificate" is defined for the "listen ... ssl" directive
```

If such, you might consider self signing a certificate.

## Generating a blank SSL certificate

You can use the following openssl command to generate the required files for `ssl_certificate` and `ssl_certificate_key`.

```
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout default.key -out default.crt -subj '/CN='
```

This outputs a blank certificate to `default.key` and `default.crt` to the current directory.

## Updating nginx configuration

The default server block in nginx can then be edited to be something like this.

```
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
```

After reloading nginx, direct access to the server's IP address should be blocked.

# If statement method

At last, it is also possible to add the following if statement block to each server.

```
server {
    listen 443 default_server;
    listen [::]:443 default_server;

    if ($host != example.com) {
            return 444;
    }

    server_name example.com;

    ...remaining content
}
```

You can use regular expressions in the if statement to match multiple servers, however there are simply too many downsides to this solution.

1. High workload.
2. High risk of mistakes.
3. Not really scalable.
4. The other solutions are plainly much better.

> Again, as you are viewing this version of this post, use `ssl_reject_handshake`! You should not be using a nginx version that has vulnerabilities!

# Refrences used

1. https://stackoverflow.com/questions/29104943/how-to-disable-direct-access-to-a-web-site-by-ip-address
2. https://erikpoehler.com/2022/08/02/how-to-block-direct-ip-access-to-your-nginx-web-server/
3. https://www.codedodle.com/disable-direct-ip-access-nginx.html
4. https://www.nginx.com/resources/wiki/start/topics/tutorials/install/