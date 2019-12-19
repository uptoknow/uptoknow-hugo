+++
title = "Setup Http Proxy"
date = 2019-12-19T08:16:19+08:00
draft = false
tags = ["blog", "uptoknow", "shadowsockes", "gost"]
categories = []
+++

Recently, I spend some time working on setup a http proxy in synology nas server, it's a little bit difficult firstly when I setup this first time, here I gonna write down how I setup a http proxy server.

* First, you need get a virtual host that running the shadowsockes server, and start the shadownsocks server using docker:

```
  docker run  -e METHOD=aes-256-cfb -e PASSWORD=<password> -p 8388:8388 -p 8388:8388/udp -d shadowsocks/shadowsocks-libev
```
* Then, on your laptop, run this to start up the proxy client:

```
  docker run -u root -d -p 1080:1080 shadowsocks/shadowsocks-libev:latest ss-local -s <server_ip> -p 8388 -b 0.0.0.0 -l 1080 -k <password> -m aes-256-cfb
```
* Now we already startup a proxy client using socks, if you need an http proxy, you need:

```
  docker run -u root -d -p 8080:8080 ginuerzh/gost:latest -L=:8080 -F=socks5://<local_ip>:1080
```

Now you can use local http proxy http://127.0.0.1:8080 on synology
