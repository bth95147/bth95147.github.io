---
title: 使用Termux部署Memos
publishDate: 2026-02-11 15:47:00
description: 'Memos是一款有趣的开源软件喵'
tags:
  - Termux
  - Memos
  - Shell
heroImage: { src: './thumbnail.png', color: '#B4C6DA' }
language: '中文'
---

## 脚本一键部署

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/ac227/memos-in-termux/main/zh/install.sh)"
```

其实看到这里就可以了，下面是摸索的过程。

## 部署旧版 memos

一开始是想从源码编译的，因为在 termux 里使用 docker 几乎不太可能，但是出现了很多至今未解决的问题，所以就去看了一下官方的 release ，发现 `memos v0.24.0` 提供了二进制文件，这下就好办了。部署源码的过程放在最后了。

先下载解压

```bash
wget https://github.com/usememos/memos/releases/download/v0.24.0/memos_v0.24.0_linux_arm64.tar.gz
tar zxvf memos_v0.24.0_linux_arm64.tar.gz
rm memos_v0.24.0_linux_arm64.tar.gz
```

直接运行就搞定了

```bash
./memos
```

## 连接 TGBot

发现 `memos v0.24.0` 没有 TGBot 的设置选项，经过一番资料查询，原来是 TGBot 被分出来成为一个单独的项目了（[usememos/telegram-integration](https://github.com/usememos/telegram-integration)）。这里面也有好多坑，首先不要使用最新版 release （验证 access token 时会有问题），建议使用 `memogram v0.2.0` 。

```bash
wget https://github.com/usememos/telegram-integration/releases/download/v0.2.0/memogram_v0.2.0_linux_arm64.tar.gz
tar zxvf memogram_v0.2.0_linux_arm64.tar.gz
rm memogram_v0.2.0_linux_arm64.tar.gz
```

需要在同一目录写一个 `.env`

```
SERVER_ADDR=localhost:<memos_port>
BOT_TOKEN=<your_telegram_bot_token>
BOT_PROXY_ADDR=https://api.telegram.org
ALLOWED_USERNAMES=<tg_user_name_1>,<tg_user_name_2>
```

直接运行

```bash
./memogram
```

初始报错

```
dial tcp: lookup api.telegram.org on [::1]:53: read udp [::1]:38087->[::1]:53: read: connection refused
```

解析请求指向 IPv6 本地回环地址 `::1` ？？？

```bash
go version -m $(which ./memogram) | grep -i netgo
```

空返回，既然没显示 `netgo` ，说明二进制不是用纯 `netgo` 构建的，理论上应该走 `cgo` 系统解析器。Go有两种 DNS 解析器，默认 `cgo` （不依赖 `resolv.conf` ）， `netgo` 纯 Go 的内置解析器（依赖 `/etc/resolv.conf` ）。

先尝试修复一下

```bash
export GODEBUG=netdns=cgo
echo "nameserver 8.8.8.8" > $PREFIX/etc/resolv.conf
echo "nameserver 1.1.1.1" >> $PREFIX/etc/resolv.conf
```

如果使用 HTTP/Sock5 代理的话会出现 tls 问题：

```bash
export GODEBUG=netdns=cgo
export http_proxy="http://127.0.0.1:7890"
export https_proxy="http://127.0.0.1:7890"

./memogram
panic: failed to create bot: error call getMe, error do request for method getMe, Post "https://api.telegram.org/token/getMe": tls: failed to verify certificate: x509: certificate signed by unknown authority
```

最后还是使用 `proot` 绑定 `resolv.conf`

```bash
proot -b $PREFIX/etc/resolv.conf:/etc/resolv.conf \
      -b $PREFIX/etc/tls:/etc/ssl \
      ./memogram
```

完美运行

## 从最新memos源码构建

还没成功喵，只是先记录一下，以后有时间再折腾

根据[官网的教程](https://usememos.com/docs/deploy/development)，就是先克隆，再 run

```bash
1. Clone the repo
git clone https://github.com/usememos/memos.git
cd memos

2. Run the backend
go run ./cmd/memos --port 8081

3. Open the app
Visit http://localhost:8081 in your browser.
For frontend development, see the web/ directory in the Memos repository.
```

其实坑一堆，前端没有构建，需要自行构建一下

```bash
cd memos/web
npm run build
```

Termux 一般会提示内存不足，需要增加一下 Node 堆内存

```bash
NODE_OPTIONS="--max-old-space-size=2048" npm run build
```

> 需要先清空旧的 dist ，避免奇奇怪怪的问题

到这里应该已经得到了 memos 二进制文件，直接运行没什么问题，但是网页显示

```
No embeddable frontend found
```

找了很久问题依旧没啥进展，那个，也欢迎来讨论。
