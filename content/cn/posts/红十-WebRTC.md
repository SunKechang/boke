---
title: "记两次失败的开发经历"
date: 2023-04-21
categories: ['Application']
draft: true
---

该部分包含在编写过程中遇到的BUG，以及解决方法。
WebRTC在pc端运行流畅，但移动端无法运行。
首先使用chrome://inspect来监听移动端的网页。ios手机可以看这篇博客：https://www.hozen.site/archives/30/
安卓机就网上搜吧。

`TypeError: Member RTCIceServer.urls is required and must be an instance of (DOMString or sequence)`
报这个错误是配置`iceServers`这个参数时候的问题，应该是不同浏览器要求不同，我之前配的url参数，这里需要urls参数，那就加一个urls参数就好了。
```js
let iceServer = { // stun 服务，如果要做到 NAT 穿透，还需要 turn 服务
                "iceServers": [
                    {
                        "url": "stun:stun.l.google.com:19302",
                        "urls": "stun:stun.l.google.com:19302"
                    }
                ]
            }
```

`TypeError: a.addStream is not a function. (In 'a.addStream(this.localStream)', 'a.addStream' is undefined)`
这是由于WebRTC的API修改导致的，老版本使用addStream将本地流加入到远程流中，新版本中使用addTrack函数。像这样：
```js
this.localStream.getTracks().forEach(track => {
  peerConnection.addTrack(track, this.localStream);
});
```
就可以了。

IOS要关闭NSUrlSession WebSocket特性，这个功能会使websocket第一次连接正常，之后的连接就不正常了。

ios一直只发送两个ice_candidate消息，这其实是因为配置的stun、turn服务器不工作导致的，自己配置了一个服务器之后就能跑通了。具体配置服务器看文章：https://www.jianshu.com/p/ba0b0e8a461f

这里面还有一些报错

0: ERROR:
CONFIG ERROR: Empty cli-password, and so telnet cli interface is disabled! Please set a non empty cli-password!
0: WARNING: cannot find certificate file: turn_server_cert.pem (1)
0: WARNING: cannot start TLS and DTLS listeners because certificate file is not set properly
0: WARNING: cannot find private key file: turn_server_pkey.pem (1)
0: WARNING: cannot start TLS and DTLS listeners because private key file is not set properly

解决方法如下：
这个错误消息表明你的 coturn 配置有一些问题。具体来说，它提示以下问题：

"Empty cli-password"：你没有设置管理命令行界面（CLI）的密码，因此 CLI 界面被禁用了。你需要在配置文件中设置一个非空的 cli-password 参数。

"cannot find certificate file"：coturn 无法找到 SSL 证书文件 turn_server_cert.pem。coturn 需要该文件作为启用 TLS 和 DTLS 监听器所需的一部分。你需要检查证书文件是否存在，并确保在配置文件中正确地指定了该文件的路径。

"cannot find private key file"：coturn 同样找不到私钥文件 turn_server_pkey.pem。coturn 需要该文件与证书配对，以启用 TLS 和 DTLS 监听器。你需要检查私钥文件是否存在，并确保在配置文件中正确地指定了该文件的路径。

为了解决这些问题，请按照以下步骤：

找到并打开 coturn 的配置文件。

确认 cli-password 参数已经设置，并且不是空字符串。如果没有设置，请在配置文件中添加以下行：

cli-password=your_password
将 "your_password" 替换为你想要设置的密码。

检查 SSL 证书和私钥文件是否可用。如果缺少任何一个文件，请使用以下命令生成它们：
openssl genrsa -out turn_server_pkey.pem 2048
openssl req -new -key turn_server_pkey.pem -out turn_server_cert.csr
openssl x509 -req -in turn_server_cert.csr -signkey turn_server_pkey.pem -out turn_server_cert.pem
这些命令将生成一个新的私钥和自签名证书，并将它们存储在当前目录中。你可以根据需要将其移动到其他位置，并在 coturn 配置文件中正确地指定路径。

保存并关闭配置文件，并重新启动 coturn 服务。如果一切正常，coturn 应该能够成功启动并监听连接请求。