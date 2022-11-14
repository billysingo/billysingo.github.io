---
layout: post
title: Snell v4 with Shadow-tls 搭建记录
date: 2022-11-14
Author: Billysingo
tags: [snell, shadow-tls]
comments: true
---

最近Surge更新了5.20版本，添加了新协议Snellv4，添加了传输层插件Shadow-tls的支持，自己手动搭建了一下这个组合，现在做个记录。

## Snell

Snell比较简单，官网下载地址下载对应二进制文件即可运行。

https://manual.nssurge.com/others/snell.html

此处以 linux-amd64 为例：

```
# 如果系统没有预装可能需要先下载安装 wget 及 unzip
# APT
sudo apt update && sudo apt install wget unzip
# DNF
sudo dnf install unzip

# 下载 Snell Server
wget https://dl.nssurge.com/snell/snell-server-v4.0.0-linux-amd64.zip

# 解压 Snell Server 到指定目录
sudo unzip snell-server-v4.0.0-linux-amd64.zip -d /usr/local/bin
```

配置文件：

```
# 可以使用 Snell 的 wizard 生成一个配置文件
sudo snell-server --wizard -c /etc/snell-server.conf
# 或者自己编写一个
sudo vim /etc/snell-server.conf
```

配置文件这里由于我们要使用shadow-tls，所以监听本地地址即可。obfs也不需要开启。

```
[snell-server]
listen = 127.0.0.1:12333
psk = ScRQc0kMVaWKuIz5r41INDBkNG82CfR
ipv6 = true
obfs = false
```
配置systemd服务：

```
sudo vim /etc/systemd/system/snell.service
```

```
[Unit]
Description=Snell Proxy Service
After=network.target nss-lookup.target

[Service]
Type=simple
User=root
Group=root
LimitNOFILE=32768
ExecStart=/usr/local/bin/snell-server -c /etc/snell-server.conf
StandardOutput=append:/var/log/snell.log
StandardError=append:/var/log/snell.log
SyslogIdentifier=snell-server

[Install]
WantedBy=multi-user.target
```

然后加载服务：

```
# 重载服务
sudo systemctl daemon-reload

# 开机运行 Snell
sudo systemctl enable snell

# 开启 Snell
sudo systemctl start snell

# 关闭 Snell
sudo systemctl stop snell

## 查看 Snell 状态
sudo systemctl status snell
```

snell的运行日志会保存在/var/log/snell.log

查看日志：

```
tail -f /var/log/snell.log
```

## Shadow-tls

接下来配置shadow-tls

项目地址 https://github.com/ihciah/shadow-tls

可采用wiki里的docker compose方式运行，我这里还是自己下载编译好的文件运行

依旧以 linux-amd64 为例：

```
# 下载 Shadow-tls
wget https://github.com/ihciah/shadow-tls/releases/latest/download/shadow-tls-x86_64-unknown-linux-musl

# 重命名并放入bin目录
mv shadow-tls-x86_64-unknown-linux-musl shadow-tls && chmod +x shadow-tls
cp shadow-tls /usr/local/bin/
```

配置systemd服务：

```
sudo vim /etc/systemd/system/shadow-tls.service
```


```
[Unit]
Description=Shadow-TLS Proxy Service
After=network.target nss-lookup.target

[Service]
Type=simple
ExecStart=/usr/local/bin/shadow-tls server --listen 0.0.0.0:443 --server 127.0.0.1:12333 --tls cloud.tencent.com:443 --password passwd
StandardOutput=append:/var/log/shadow-tls.log
StandardError=append:/var/log/shadow-tls.log
SyslogIdentifier=shadow-tls
Environment="RUST_LOG=info"

[Install]
WantedBy=multi-user.target
```

注意上面0.0.0.0:443可修改绑定的端口，127.0.0.1:12333一定要和前面snell的端口一致。此处--tls的参数和--password参数可根据自己需求修改。

一些推荐的--tls参数：

```
cloud.tencent.com:443
icloud.apple.com:443
radio.itunes.apple.com:443
cdn-dynmedia-1.microsoft.com:443
...
```

选择域名时最好先ping一下确认延时不要太高。然后运行以下命令确认支持tls1.3


    curl -I --tlsv1.3 --tls-max 1.3 -vvv https://example.com


然后加载服务：

```
# 重载服务
sudo systemctl daemon-reload

# 开机运行 Shadow-tls
sudo systemctl enable shadow-tls

# 开启 Shadow-tls
sudo systemctl start shadow-tls

# 关闭 Shadow-tls
sudo systemctl stop shadow-tls

## 查看 Shadow-tls 状态
sudo systemctl status shadow-tls
```

Shadow-tls的运行日志会保存在/var/log/shadow-tls.log

查看日志：


    tail -f /var/log/shadow-tls.log

## Surge配置

在[Proxy]里添加

    snell-stls = snell, 你的VPS地址, 443, psk=ScRQc0kMVaWKuIz5r41INDBkNG82CfR, version=4, reuse=true, shadow-tls-password=passwd, shadow-tls-sni=cloud.tencent.com

注意端口为shadow-tls监听的端口。目前版本shadow-tls相关配置只能在文本编辑里配置。