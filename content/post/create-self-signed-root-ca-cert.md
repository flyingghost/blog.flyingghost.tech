---
title: "使用自签名根证书来颁发SSL证书"
date: 2019-10-29T15:15:34+08:00
draft: false

tags:

- Security
- Certificate
- DevOps
---

如果你有茫茫多的内部测试网站，每个网站都部署了自签名的https证书。

如果你每天都会收到chrome的安全警告然后手动忽略。

如果你厌倦了每天添加一个不同的自签证书进入信任列表。

如果你想研究某种必须使用https证书的技术并需要部署多个服务组建测试网络环境（例如我）。

那么这份攻略值得一读。

本攻略打算自建一个rootCA，并作为根证书机构给所有内部网站颁发SSL证书。这样所有的内部用户就只需信任一份根证书，证书链体系就会自动信任由它颁发的所有中级证书和子证书。

本攻略还更新了网上一大波已过期的教程，避免大家因使用陈旧地图而掉入同样的坑。

## 生成rootCA的私钥

首先，我们为根证书机构生成私钥。

```shell
openssl genrsa -out rootCA_key.pem 2048
```

得到`rootCA_key.pem`密钥文件。

## 生成rootCA的根证书

然后使用rootCA的私钥生成rootCA的根证书。

```shell
openssl req -new -x509 -sha256 -days 3650 -key rootCA_key.pem -out rootCA_cert.pem
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) []:CN
State or Province Name (full name) []:Shanghai
Locality Name (eg, city) []:Shanghai
Organization Name (eg, company) []:flyingghost
Organizational Unit Name (eg, section) []:IT
Common Name (eg, fully qualified host name) []:flyingghost.tech
Email Address []:flyingghost@gmail.com
```

得到`rootCA_cert.pem`证书文件。

几个注意点：

- openssl默认会使用SHA-1作为证书签名算法，而SHA-1已因安全强度不足而被各大浏览器标记为“不安全”甚至拒绝使用。所以在任何签发证书环节请使用`-sha256`参数强制使用SHA-256来签名。
- openssl默认签发证书只有一个月有效期。可使用`-days 3650`参数自行指定证书有效期时长。



## 生成服务器私钥

这个操作由证书申请者来执行。

```shell
openssl genrsa -out local.flyingghost.tech.key 2048
```

## 生成证书请求CSR

接下来，证书申请者应该提供自己的公钥和各种必要信息给证书颁发机构以申请证书了，这个请求被称为 Certificate Signing Request (CSR)。

```shell
openssl req -new -key local.flyingghost.tech.key -out local.flyingghost.tech.csr
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) []:CN
State or Province Name (full name) []:Shanghai
Locality Name (eg, city) []:Shanghai
Organization Name (eg, company) []:flyingghost
Organizational Unit Name (eg, section) []:test
Common Name (eg, fully qualified host name) []:local.flyingghost.tech
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
```

接下来就可以把这个csr文件连同几百美金一起发给证书颁发者（还是我）了。

## 根证书机构签发证书

证书机构收到CSR之后，使用根证书签发出服务器最终使用的证书。

```shell
openssl x509 -req -in local.flyingghost.tech.csr -CA rootCA_cert.pem -CAkey rootCA_key.pem -CAcreateserial -extfile local.flyingghost.tech.v3.ext -sha256 -days 3650 -out local.flyingghost.tech.crt
```

其中，`v3.ext`文件是签发证书时使用的扩展信息。内容如下：

```
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = local.flyingghost.tech
```

为什么需要这个呢？等会讲。

## 查看证书详情

可以输出证书细节以查看证书是否签署妥当。

```shell
openssl x509 -in local.flyingghost.tech.crt -noout -text
```

没有问题就可以把证书文件和发票发回申请者（还是我）了。

## 服务器部署证书

私钥和证书文件部署方式请参阅各服务器文档，大同小异。以nginx为例：

```
server {
	  # mapping to local 8080
    listen 443 ssl http2;
    server_name local.flyingghost.tech;
    set $base /var/www/html;
	  root $base;

    # SSL
    ssl_certificate      certs/local.flyingghost.tech.crt;
    ssl_certificate_key  certs/local.flyingghost.tech.key;

	  index index.html;
}
```

记得重启服务。



## 将rootCA根证书加入客户端系统信任列表

这一步各系统和各浏览器有所不同。以mac为例：

双击`rootCA_cert.pem`文件，将会打开钥匙串访问应用并开始添加证书。



![](https://fg-public-1252239724.file.myqcloud.com/blog/20191029161451.png)



钥匙串可以选择“登录”，仅对当前用户登录后有效。或者也可以选择“系统”，对所有用户生效。

添加成功后可在对应钥匙串（刚才我们选的是“登录”）里找到证书，再次双击，在“信任”选项中选择“始终信任”。



![](https://fg-public-1252239724.file.myqcloud.com/blog/20191029161543.png)



完成证书添加后，由此证书签发的所有证书都将得到信任。受影响的软件有：Chrome、Safari、curl、Postman等。

Firefox是一个例外，它不使用系统的证书管理，而是有自己的证书管理。所以使用Firefox的同学还得再手动添加一次。

路径为：首选项->隐私与安全->安全->证书->查看证书->证书颁发机构->导入->选择证书文件，勾选“信任由此证书颁发机构来标识网站”，确定。

![](https://fg-public-1252239724.file.myqcloud.com/blog/20191029162543.png)

大功告成！

## 效果展示

curl一把看看。

```shell
curl -v -o /dev/null https://local.flyingghost.tech:8080/
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to local.flyingghost.tech (127.0.0.1) port 8080 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* Cipher selection: ALL:!EXPORT:!EXPORT40:!EXPORT56:!aNULL:!LOW:!RC4:@STRENGTH
* successfully set certificate verify locations:
*   CAfile: /etc/ssl/cert.pem
  CApath: none
* TLSv1.2 (OUT), TLS handshake, Client hello (1):
} [228 bytes data]
* TLSv1.2 (IN), TLS handshake, Server hello (2):
{ [94 bytes data]
* TLSv1.2 (IN), TLS handshake, Certificate (11):
{ [1178 bytes data]
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
{ [1038 bytes data]
* TLSv1.2 (IN), TLS handshake, Server finished (14):
{ [4 bytes data]
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
} [262 bytes data]
* TLSv1.2 (OUT), TLS change cipher, Client hello (1):
} [1 bytes data]
* TLSv1.2 (OUT), TLS handshake, Finished (20):
} [16 bytes data]
* TLSv1.2 (IN), TLS change cipher, Client hello (1):
{ [1 bytes data]
* TLSv1.2 (IN), TLS handshake, Finished (20):
{ [16 bytes data]
* SSL connection using TLSv1.2 / DHE-RSA-AES256-GCM-SHA384
* ALPN, server accepted to use h2
* Server certificate:
*  subject: C=CN; ST=Shanghai; L=Shanghai; O=flyingghost; OU=test; CN=local.flyingghost.tech
*  start date: Oct 29 08:47:34 2019 GMT
*  expire date: Oct 26 08:47:34 2029 GMT
*  subjectAltName: host "local.flyingghost.tech" matched cert's "local.flyingghost.tech"
*  issuer: C=CN; ST=Shanghai; L=Shanghai; O=flyingghost; OU=IT; CN=flyingghost.tech; emailAddress=flyingghost@gmail.com
*  SSL certificate verify ok.
* Using HTTP2, server supports multi-use
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* Using Stream ID: 1 (easy handle 0x7fc78300e600)
> GET / HTTP/2
> Host: local.flyingghost.tech:8080
> User-Agent: curl/7.54.0
> Accept: */*
>
* Connection state changed (MAX_CONCURRENT_STREAMS updated)!
< HTTP/2 200
< server: nginx
< date: Tue, 29 Oct 2019 08:55:26 GMT
< content-type: text/html; charset=utf-8
< content-length: 612
< last-modified: Tue, 29 Oct 2019 03:11:54 GMT
< etag: "5db7adfa-264"
< accept-ranges: bytes
<
{ [612 bytes data]
100   612  100   612    0     0   7197      0 --:--:-- --:--:-- --:--:--  7285
* Connection #0 to host local.flyingghost.tech left intact
```

Chrome表示认可。

![](https://fg-public-1252239724.file.myqcloud.com/blog/20191029165900.png)

Firefox发来贺电。

![](https://fg-public-1252239724.file.myqcloud.com/blog/20191029170034.png)



## 关键问题和解释

如果按照网上一大票过期中文教程来做，curl似乎能勉强工作，但chrome访问基本上会得到这么个结果：

![](https://fg-public-1252239724.file.myqcloud.com/blog/QQ20191029-135250@2x.png)

其中包含有如下两个问题。

### Certificate - Subject Alternative Name missing

The certificate for this site does not contain a Subject Alternative Name extension containing a domain name or IP address.

旧的证书中我们都是用Common Name(CN)字段来表达主机地址，也就是证书或者CSR生成时需要输入的`Common Name (eg, fully qualified host name) []:`这一行。但从Chrome58开始，chrome要求在SSL证书中提供Subject Alternative Name(SAN)扩展字段来标识主机。过去这个字段只是用来做多主机证书，所以有些工具默认不包含这个字段。

总之，不管你是自签证书，还是通过自签CA证书签发的证书，需要确保服务器证书里包含一个`subjectAltName`字段，内有若干个`DNS`或者`IP`项。

在例子中，我们通过`-extfile`选项传入扩展属性文件`local.flyingghost.tech.v3.ext`：

```
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = local.flyingghost.tech
DNS.2 = test.flyingghost.tech
```

这样签出的证书就可以在详情中看到SAN信息：

```shell
openssl x509 -in local.flyingghost.tech.crt -noout -text

Certificate:
    Data:
        X509v3 extensions:
            X509v3 Subject Alternative Name:
                DNS:local.flyingghost.tech
```

其实使用SAN挺好。这样可以很方便的签出多主机证书和通配符证书。

### Certificate - missing

和刚才那个问题相关。Chrome 58开始只认SAN，直接忽略CN了。按上文提供SAN后自然解决。

### Certificate - insecure (SHA-1)

openssh默认使用的签名算法`SHA-1`已经被各大浏览器标记为不安全。在所有证书签发环节使用`SHA-256`作为签名算法即可。参考前例不再赘述。



## TODO

1. 中级证书机构
2. 多域名证书、通配符证书
3. xca
4. 证书吊销和机构吊销

To Be Continued...我先去拯救艾泽拉斯了。

