## HTTPS

HTTPS 是 HyperText Transfer Protocol Secure 的缩写。但 HTTPS 协议本身不能加密数据，它需要借助 SSL 或 TLS 协议层进行加密。

HTTP 协议和 TLS 协议都位于 application layer。TCP 三次握手建立连接后，使用 TLS 握手建立安全连接，后续使用协商的加密算法先对数据进行加密，再通过 HTTP 传输。

## SSL

TLS 协议已经替代了 SSL 协议，当我们说 SSL 时，事实上说的是 TLS。SSL 证书实质上时 TLS 证书，SSL v3.1、SSL v4 是 TLS 1.0+ 的别称。

## TLS

TLS 1.3 是对 TLS 1.2 的全面修订，在性能和安全性方面都有很大提升，并且减少了安全连接所需的握手次数。
TLS 1.3 只支持 Diffie-Hellman 非对称加密算法，移除了 RSA 算法。
TLS 1.3 除了减少常规握手下的的网络开销，还引入了 0-RTT：60% 的网络连接都是用户在第一次访问网站或者间隔一段时间后建立的，剩下的 40% 通过 0-RTT 策略解决。
0-RTT 策略与 TFO 的实现原理比较相似，都是通过重用会话和缓存来实现的，所以存在一定的安全风险，使用时也应该结合业务的具体场景。

## TLS 1.2 握手

1. 客户端向服务端发送 Client Hello 消息，消息携带客户端支持的协议版本、加密算法、压缩算法，以及客户端生成的随机数。
2. 服务端收到客户端消息后：
  - 向客户端发送 Server Hello 消息，并携带选取的协议版本、加密方法、session id、以及服务端生成的随机数。
  - 向客户端发送 Certificate 消息，即服务端的证书链，其中包含证书支持的域名、发行方和有效期等信息。
  - 向客户端发送 Server Key Exchange 消息，传递公钥、签名等信息。
  - 向客户端发送可选的 Certificate Request 消息，验证客户端证书。
  - 向客户端发送 Server Hello Done 消息，通知已经发送了全部的信息。
3. 客户端收到服务端的协议版本、加密方法、session id 和证书等信息后，验证服务端证书。
  - 向服务端发送 Client Key Exchange 消息，包含使用服务端公钥加密的随机字符串，即预主密钥（Pre Master Secret）。
  - 向服务端发送 Change Cipher Spec 消息，通知服务端后续数据会加密传输。
  - 向服务端发送 Finished 消息，其中包含加密后的握手信息。
4. 服务端收到 Change Cipher Spec 和 Finished 消息后：
  - 向客户端发送 Change Cipher Spec 消息，通知客户端后续数据会加密传输。
  - 向客户端发送 Finished 消息，验证客户端的 Finished 消息并完成 TLS 握手。

## RSA 握手

1. 浏览器向服务器发送随机数 client_random，TLS 版本和供筛选的加密套件列表。
2. 服务器收到，立即返回 server_random，确认双方都支持的加密套件，以及数字证书（证书中附带公钥）。
3. 浏览器接收，先验证数字证书，若通过，接着使用加密套件的密钥协商算法 RSA 算法生成另一个随机数 pre_random，并且用证书里的公钥加密，传给服务器。
4. 服务器用私钥解密这个被加密后的 pre_random。

## DH 握手（Diffie-Hellman）

1. 浏览器向服务器发送随机数 client_random，TLS 版本和供筛选的加密套件列表。
2. 服务器收到，立即返回 server_random，确认双方都支持的加密套件，以及数字证书（证书中附带公钥）。同时服务器利用私钥将 client_random, server_random, server_params 签名，生成服务器签名。然后将签名和 server_params 也发送给客户端。这里的 server_params 为 DH 算法所需参数。
3. 浏览器接收，先验证数字证书和签名。若通过，将 client_params 传递给服务器。这里的 client_params 为 DH 算法所需参数。
4. 现在客户端和服务器都有 client_params、server_params 两个参数，因 ECDHE 计算基于 “椭圆曲线离散对数”，通过这两个 DH 参数就能计算出 pre_random。

## TLS 1.3 握手

1. 浏览器向服务器发送 client_params、client_random、TLS 版本和供筛选的加密套件列表。
2. 服务器返回 server_params、server_random、TLS 版本、确定的加密套件以及证书。浏览器接收，先验证数字证书和签名。现在双方都有 client_params、server_params，可以根据 ECDHE 计算出 pre_random 了。

## HTTPS 中的 TLS 握手过程可以同时进行三次握手

需要在特定的条件下才可能发生：

- 客户端和服务端都开启了 TCP Fast Open 功能，且 TLS 版本是 1.3。
- 客户端和服务端已经完成过一次通信。

如果基于 TCP Fast Open 场景下的 TLS v1.3-RTT 会话恢复过程，不仅 TLS 和 TCP 的握手过程可以同时进行，HTTP 请求也可以在这期间内一同完成。
