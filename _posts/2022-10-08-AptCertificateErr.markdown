---
layout: post
title: "apt certificate error"
data: 2022-10-08 17:59:00
catagories: 纹到底
excerpt: 什么！证书过期了
published: true
---

今儿使用apt 出现下面错误：

> ```shell
> server certificate verification failed. CAfile: /etc/ssl/certs/ca-certificates.crt CRLfile: none
> ```

原因未知，遂google

[[How to solve apt error server certificate verification failed](https://www.claudiokuenzler.com/blog/1088/how-to-solve-apt-error-server-certificate-verification-failed)](https://www.claudiokuenzler.com/blog/1088/how-to-solve-apt-error-server-certificate-verification-failed)

大概原因是有一个CA的根证书在9月30号过期，导致校验失败。用户可以通过配置apt来忽略这个证书校验

> echo 'Acquire::https::xxxxxxxxxxxxVerify-Peer "false";' > /etc/apt/apt.conf.d/99influxdata-cert

apt的配置方式我们暂且忽略，这个知识点太硬了。而且忽略证书校验毕竟只是work的方法，文中还给出了该问题的正确解法： [remove DST root CA X3 这个证书](https://www.claudiokuenzler.com/blog/1135/lets-encrypt-root-ca-expired-git-server-certificate-verification-failed-x3)

## CA工作原理

### 非对称加密

非对称加密算法大家耳熟能详，属于我的知识范畴内（数学实现并不是），不做赘述

### 非对称加密通信

![asym-enc.jpg](https://s2.loli.net/2022/10/08/ImA6KVrxlayzOs9.jpg)

撇开如何交换公钥的问题，非对称加密用于通信需要4个key参与： A.pri  A.pub  B.pri  B.pub。 如果只有一方的公私钥的话，只能保证数据不被其他人看到，并不能防止有中间人制造假数据。

使用4个key即保证了数据的加密性又保证了数据的完整性

发送数据包括：   data.hash.sign(a.pri) + data.verify(b.pub)

接收方需要验证 ： data.hash.encrypt(a.pri).decrypt(a.pub) = data.encrypt(b.pub).decrypt(b.pri).hash

### CA

上面的流程遗留了一个问题：公钥如何传递。 如果公钥的传递过程被人破坏，那么通信就不是安全的了。 CA机制的出现就是一定程度上解决了这个问题。

引入一个可信第三方，并将第三方的公钥预置到操作系统/浏览器中。通信双方的公钥交换通过第三方的秘钥进行。

![ca-sign.jpg](https://s2.loli.net/2022/10/08/x9HlfEq1Du8ioPe.jpg)

CA归根结底并没有带来算法上的进步，而是一种基于共识下的方法，通过另一种手段解决了public秘钥交换的问题。

### CA证书链

证书链是一种保护根证书的方法，CA的技术手段表明了CA 根证书非常非常重要。所以人们采取了各种各样的手段保证证书的安全性，CA证书链便是其中一种手段。GoDaddy是这样描述的：

> Intermediate certificates are used as a stand-in for our rootl certificate. We use intermediate certificates as a proxy because we must keep our root certificate behind numerous layers of security, ensuring its keys are absolutely inaccessible.
> However, because the root certificate itself signed the intermediate certificate, the intermediate certificate can be used to sign the SSLs our customers install and maintain the "Chain of Trust."

PS: ca证书的认证过程是否需要联网?

# 问题定位

有了基础知识，那这个问题应该如何debug呢？

我们把现场复原，在apt update时又出现了该错误

> server certificate verification failed. CAfile: /etc/ssl/certs/ca-certificates.crt CRLfile: none

该镜像源的地址为：https://mirrors.tuna.tsinghua.edu.cn/ubuntu

```shell
->   curl https://mirrors.tuna.tsinghua.edu.cn/ubuntu

curl: (60) server certificate verification failed. CAfile: /etc/ssl/certs/ca-certificates.crt CRLfile: none
More details here: http://curl.haxx.se/docs/sslcerts.html
```

确认这个网站的证书有问题。

使用ssllabs查看证书链（这里显示的是本地证书的验证结果？）

![tsinghua-certi-paths.png](https://s2.loli.net/2022/10/08/GJqdI3VlYh6jogt.png)

果然！ 在Path #2 上， DST Root CA X3 在2021-09-30号过期了。到这里debug就结束了，只要按照上面Let's Encrypt给出的解决方案，remove掉DST Root CA X3就可以了。

那么，这个证书究竟存在机器的哪个位置，其原始内容又是什么呢？

在Ubunt 16.04下， 证书被存放在/etc/ssl/certs中

```shell
->   ll |grep DST

lrwxrwxrwx 1 root root   18 10月  9 10:38 12d55845.0 -> DST_Root_CA_X3.pem
lrwxrwxrwx 1 root root   18 10月  9 10:38 2e5ac55d.0 -> DST_Root_CA_X3.pem
lrwxrwxrwx 1 root root   53 10月  9 10:38 DST_Root_CA_X3.pem -> /usr/share/ca-certificates/mozilla/DST_Root_CA_X3.crt
```

查看一下文件内容

> -----BEGIN CERTIFICATE-----
> MIIDSjCCAjKgAwIBAgIQRK+wgNajJ7qJMDmGLvhAazANBgkqhkiG9w0BAQUFADA/
> MSQwIgYDVQQKExtEaWdpdGFsIFNpZ25hdHVyZSBUcnVzdCBDby4xFzAVBgNVBAMT
> DkRTVCBSb290IENBIFgzMB4XDTAwMDkzMDIxMTIxOVoXDTIxMDkzMDE0MDExNVow
> PzEkMCIGA1UEChMbRGlnaXRhbCBTaWduYXR1cmUgVHJ1c3QgQ28uMRcwFQYDVQQD
> Ew5EU1QgUm9vdCBDQSBYMzCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEB
> AN+v6ZdQCINXtMxiZfaQguzH0yxrMMpb7NnDfcdAwRgUi+DoM3ZJKuM/IUmTrE4O
> rz5Iy2Xu/NMhD2XSKtkyj4zl93ewEnu1lcCJo6m67XMuegwGMoOifooUMM0RoOEq
> OLl5CjH9UL2AZd+3UWODyOKIYepLYYHsUmu5ouJLGiifSKOeDNoJjj4XLh7dIN9b
> xiqKqy69cK3FCxolkHRyxXtqqzTWMIn/5WgTe1QLyNau7Fqckh49ZLOMxt+/yUFw
> 7BZy1SbsOFU5Q9D8/RhcQPGX69Wam40dutolucbY38EVAjqr2m7xPi71XAicPNaD
> aeQQmxkqtilX4+U9m5/wAl0CAwEAAaNCMEAwDwYDVR0TAQH/BAUwAwEB/zAOBgNV
> HQ8BAf8EBAMCAQYwHQYDVR0OBBYEFMSnsaR7LHH62+FLkHX/xBVghYkQMA0GCSqG
> SIb3DQEBBQUAA4IBAQCjGiybFwBcqR7uKGY3Or+Dxz9LwwmglSBd49lZRNI+DT69
> ikugdB/OEIKcdBodfpga3csTS7MgROSR6cz8faXbauX+5v3gTt23ADq1cEmv8uXr
> AvHRAosZy5Q6XkjEGB5YGV8eAlrwDPGxrancWYaLbumR9YbK+rlmM6pZW87ipxZz
> R8srzJmwN0jP41ZL9c8PDHIyh8bwRLtTcm1D9SZImlJnt1ir/md2cXjbDaJWFBM5
> JDGFoqgCWjBH4d1QB7wCCZAA62RjYJsWvIjJEubSfZGL+T0yjWW06XyxV3bqxbYo
> Ob8VZRzI9neWagqNdwvYkQsEjgfbKbYK7p2CNTUQ
> -----END CERTIFICATE-----

pem格式看不懂，需要转换一下

```shell
->    openssl x509 -noout -text -in DST_Root_CA_X3.pem

Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            44:af:b0:80:d6:a3:27:ba:89:30:39:86:2e:f8:40:6b
    Signature Algorithm: sha1WithRSAEncryption
        Issuer: O=Digital Signature Trust Co., CN=DST Root CA X3
        Validity
            Not Before: Sep 30 21:12:19 2000 GMT
            Not After : Sep 30 14:01:15 2021 GMT
        Subject: O=Digital Signature Trust Co., CN=DST Root CA X3
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:df:af:e9:97:50:08:83:57:b4:cc:62:65:f6:90:	
                    82:ec:c7:d3:2c:6b:30:ca:5b:ec:d9:c3:7d:c7:40:
                    c1:18:14:8b:e0:e8:33:76:49:2a:e3:3f:21:49:93:
                    ac:4e:0e:af:3e:48:cb:65:ee:fc:d3:21:0f:65:d2:
                    2a:d9:32:8f:8c:e5:f7:77:b0:12:7b:b5:95:c0:89:
                    a3:a9:ba:ed:73:2e:7a:0c:06:32:83:a2:7e:8a:14:
                    30:cd:11:a0:e1:2a:38:b9:79:0a:31:fd:50:bd:80:
                    65:df:b7:51:63:83:c8:e2:88:61:ea:4b:61:81:ec:
                    52:6b:b9:a2:e2:4b:1a:28:9f:48:a3:9e:0c:da:09:
                    8e:3e:17:2e:1e:dd:20:df:5b:c6:2a:8a:ab:2e:bd:
                    70:ad:c5:0b:1a:25:90:74:72:c5:7b:6a:ab:34:d6:
                    30:89:ff:e5:68:13:7b:54:0b:c8:d6:ae:ec:5a:9c:
                    92:1e:3d:64:b3:8c:c6:df:bf:c9:41:70:ec:16:72:
                    d5:26:ec:38:55:39:43:d0:fc:fd:18:5c:40:f1:97:
                    eb:d5:9a:9b:8d:1d:ba:da:25:b9:c6:d8:df:c1:15:
                    02:3a:ab:da:6e:f1:3e:2e:f5:5c:08:9c:3c:d6:83:
                    69:e4:10:9b:19:2a:b6:29:57:e3:e5:3d:9b:9f:f0:
                    02:5d
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Basic Constraints: critical
                CA:TRUE
            X509v3 Key Usage: critical
                Certificate Sign, CRL Sign
            X509v3 Subject Key Identifier: 
                C4:A7:B1:A4:7B:2C:71:FA:DB:E1:4B:90:75:FF:C4:15:60:85:89:10
    Signature Algorithm: sha1WithRSAEncryption
         a3:1a:2c:9b:17:00:5c:a9:1e:ee:28:66:37:3a:bf:83:c7:3f:
         4b:c3:09:a0:95:20:5d:e3:d9:59:44:d2:3e:0d:3e:bd:8a:4b:
         a0:74:1f:ce:10:82:9c:74:1a:1d:7e:98:1a:dd:cb:13:4b:b3:
         20:44:e4:91:e9:cc:fc:7d:a5:db:6a:e5:fe:e6:fd:e0:4e:dd:
         b7:00:3a:b5:70:49:af:f2:e5:eb:02:f1:d1:02:8b:19:cb:94:
         3a:5e:48:c4:18:1e:58:19:5f:1e:02:5a:f0:0c:f1:b1:ad:a9:
         dc:59:86:8b:6e:e9:91:f5:86:ca:fa:b9:66:33:aa:59:5b:ce:
         e2:a7:16:73:47:cb:2b:cc:99:b0:37:48:cf:e3:56:4b:f5:cf:
         0f:0c:72:32:87:c6:f0:44:bb:53:72:6d:43:f5:26:48:9a:52:
         67:b7:58:ab:fe:67:76:71:78:db:0d:a2:56:14:13:39:24:31:
         85:a2:a8:02:5a:30:47:e1:dd:50:07:bc:02:09:90:00:eb:64:
         63:60:9b:16:bc:88:c9:12:e6:d2:7d:91:8b:f9:3d:32:8d:65:
         b4:e9:7c:b1:57:76:ea:c5:b6:28:39:bf:15:65:1c:c8:f6:77:
         96:6a:0a:8d:77:0b:d8:91:0b:04:8e:07:db:29:b6:0a:ee:9d:
         82:35:35:10
```

到此，这个过期的证书就找到了，这次debug之旅也圆满结束！