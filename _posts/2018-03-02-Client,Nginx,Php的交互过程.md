---
title: Client,Nginx,Php的交互过程
description: 梳理关于web应用到Nginx到PHP上的整个过程
categories:
 - others
tags:
 - Php
 - Nignx
---
> 梳理关于web应用到Nginx到PHP上的整个过程

## Client到Nginx之间的交互
1. client发起http请求。
2. dns服务器解析域名得到主机ip。
3. 默认端口为80，通过ip:port建立tcp/ip链接。
4. 建立连接,tcp/ip三次握手，建立成功发送数据包。
5. nginx匹配规则进行请求分发。
    * .html: 静态内容，分发静态内容响应
    * .php: php脚本，转发请求内容到php-fpm进程，分发php-fpm返回的内容响应
6. 断开连接,tcp/ip四次握手，断开成功。


## Nginx与PHP
Nginx 不能直接处理 PHP 文件，而是将请求通过 FastCGI 交给 PHP-FPM 处理。
FastCGI 是一个协议，它是应用程序和 WEB 服务器连接的桥梁。

### 简述CGI、FastCGI、PHP-CGI、PHP-FPM
1. 什么是 cgi  
>早期的 webserver 只处理 html 等静态文件，但是随着技术的发展，出现了像php等动态语言。
为了解决不同的语言解释器(如php、python解释器)与 webserver 的通信，于是出现了 cgi 协议。只要按照 cgi 协议去编写程序，就能实现语言解释器与 webwerver 的通信,如 php-cgi 程序。
2. FastCGI
>webserver每收到一个请求，都会去fork一个cgi进程，请求结束再kill掉这个进程。如果有大量的请求就会造成资源的浪费,于是fastcgi使得请求结束后保留进程,可以一次处理多个请求,避免了每次请求都fork一个进程,很好的提高了效率。
3. php-fpm
>php-fpm(Php-Fastcgi Process Manager)是 FastCGI 的实现，并提供了进程管理的功能,包含 master 主进程和 worker 子进程。master 进程负责解析配置文件，初始化执行环境，然后再启动多个worker进程，这个子进程就是php-cgi.php-fpm 可以根据配置文件动态的控制worker数量,既提高了性能又节约了资源。

### Nginx与PHP的结合
```conf
location ~ \.php(.*)$ {
    fastcgi_pass   127.0.0.1:9000;
    fastcgi_index  index.php;
    include        fastcgi.conf;
}
```
这里可以看到 Nginx 接受请求后 将匹配到的 PHP 文件通过 fastcgi_pass 转发到了9000端口,我们通过`netstat -lnp | grep 9000` 发现 php.fpm 监听了此端口,打开 php.fpm 的配置文件找到了这一段。

```conf
; The address on which to accept FastCGI requests.
; Valid syntaxes are:
;   'ip.add.re.ss:port'    - to listen on a TCP socket to a specific IPv4 address on
;                            a specific port;
;   '[ip:6:addr:ess]:port' - to listen on a TCP socket to a specific IPv6 address on
;                            a specific port;
;   'port'                 - to listen on a TCP socket to all addresses
;                            (IPv6 and IPv4-mapped) on a specific port;
;   '/path/to/unix/socket' - to listen on a unix socket.
; Note: This value is mandatory.
listen = 127.0.0.1:9000
```
通过配置文件我们可以发现 php.fpm 还可以通过 unix socket 连接套接字的方式接收 Nginx 的转发,具体操作如下:
1. 新建/dev/shm/php-cgi.sock文件，然后将所有者改为与 nginx 的用户一致。
```sh
touch /dev/shm/php-cgi.sock
chown nginx:nginx /dev/shm/php-cgi.sock
```
2. 编辑 Nginx 中的配置文件。
将`fastcgi_pass 127.0.0.1:9000;`修改为`fastcgi_pass unix:/dev/shm/php-cgi.sock;`
3. 编辑 php-fpm.conf 文件。
将`listen = 127.0.0.1:9000` 修改为 `listen = /dev/shm/php-cgi.sock`
4. 重启php-fpm与nginx,ls -al查看/dev/shm/php-cgi.sock由普通文件变成s开头的unix套接字。

php-fpm的master进程接收到请求后，会把请求分发到php-fpm的子进程，每个php-fpm 子进程都包含一个php解析器。  
最后php-fpm进程处理完请求后返回给nginx。