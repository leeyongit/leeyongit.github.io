---
title: MacOS下自己创建根证书和域SSL证书实现https调试
categories: [技术, HTTP]
tags: [https, ssl, openssl]
---

这篇文章是讲关于如何使用 OpenSSL 在本地创建一个 HTTPS 保护的开发环境，本文基于 MacOS。

<!--more-->

## 创建根SSL证书

第一步是创建一个安全套接层（CA SSL）根证书。然后可以使用此根证书为可能为单个域生成的任意数量的证书签名。

>  CA，Catificate Authority，它的作用就是提供证书（即服务器证书，由域名、公司信息、序列号和签名信息组成）加强服务端和客户端之间信息交互的安全性，以及证书运维相关服务。

创建 root key，
```bash
$ openssl genrsa -des3 -out rootCA.key 2048
```

这一步系统将提示您输入密码，每次使用此特定密钥生成证书时都需要输入该密码。

使用生成的密钥来创建新的根SSL证书。并将其保存为rootCA.pem。证书有效期为10年。在这一过程中，还将被提示输入其他可选信息。

```bash
$ openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 3650 -out rootCA.pem
```

这是我输入的信息，仅供参考。

> Country Name (2 letter code) []:CN
> State or Province Name (full name) []:Province
> Locality Name (eg, city) []:City
> Organization Name (eg, company) []:WIZ Technology Co. Ltd.
> Organizational Unit Name (eg, section) []:WIZ Technology Co. Ltd.
> Common Name (eg, fully qualified host name) []:WIZ Technology Root CA

## 信任根SSL证书

打开【钥匙串访问】，左侧【钥匙串】选择【系统】，【种类】选择【证书】，然后把刚才生成的根证书导入进来（根证书是rootCA.pem）。

![1](/assets/img/posts/adtuo.png)

双击此证书，在【信任】设置中，SSL和X.509基本策略两项选择【始终信任】。

![2](/assets/img/posts/gku87.png)

## 生成域SSL证书

由于我们公司解析一个域名 *.l.wizmacau.com 指向127.0.0.1，所以我们准备生成一个 *.l.wizmacau.com 的通配域名证书。

创建一个v3.ext文件，以创建一个X509 v3证书。注意我们指定了subjectAltName选项。

```
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage=digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName=@alt_names

[alt_names]
DNS.1 = *.l.wizmacau.com
```

创建证书密钥，

```bash
$ openssl req -new -sha256 -nodes -out server.csr -newkey rsa:2048 -keyout server.key
```

需要输入的一个可选信息，注意 Common Name 的填写。

> Country Name (2 letter code) []:CN
> State or Province Name (full name) []:Province
> Locality Name (eg, city) []:City
> Organization Name (eg, company) []:WIZ Technology Co. Ltd.
> Organizational Unit Name (eg, section) []:WIZ Technology Co. Ltd.
> Common Name (eg, fully qualified host name) []:*.l.wizmacau.com

证书签名请求通过我们之前创建的根SSL证书颁发，创建出一个 *.l.wizmcau.com 的域名证书。输出是一个名为的证书文件server.crt，

```bash
$ openssl x509 -req -in server.csr -CA [rootCA.pem路径] -CAkey [rootCA.key路径] -CAcreateserial -out server.crt -days 500 -sha256 -extfile v3.ext
```

注意 CA 和 CA Key 根据实际情况填写路径，我的 rootCA 证书根域SSL证书是分开的。

现在可以开始使用这张证书了，我服务器使用的是 nginx，这里贴下部分配置。

```nginx
server {
    listen 80;
    listen 443 ssl;

    ssl_certificate     star.l.wizmacau.com/server.crt;
    ssl_certificate_key star.l.wizmacau.com/server.key;
    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    ...
}
```

![3](/assets/img/posts/ddur9.png)