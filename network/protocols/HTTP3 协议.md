# HTTP/3 (HTTP over QUIC) 协议

目前正在落地或已经被广泛应用的新协议，我接触到的有 WebSocket、HTTP/2、gRPC、TLS1.3。

但是使用 Chrome DevTools 查看，可以发现主流网站的部分 API，已经用上了 HTTP/3。
为了跟上技术的脚步，显然我们也该了解一下 HTTP/3 了。

HTTP/3 相比 HTTP/2 有如下优势：

1. 在传输层使用 UDP 协议替换掉了 TCP 协议，好处有：
   1. 避免了 TCP 丢包重传导致的连接阻塞。因为 HTTP/2 只使用一个 TCP 连接（多路复用），这会放大连接阻塞造成的延迟。
   2. 传输层不再需要进行 TCP 三次握手。（握手在 HTTP/3 应用层进行）
2. 抛弃了 TLS 协议，使用自定义的方式进行加密握手与通信，好处有：
   1. 直接在 HTTP/3 层进行加密通讯的握手，并且恢复通信时可以通过缓存实现 0RTT 握手（0RTT 握手已经被 TLS1.3 采用了）。
   2. 原生的加密通讯。
1. 改进的拥塞控制
2. 避免队头阻塞（TCP 丢包重传）的多路复用。
3. 连接迁移
4. 前向冗余纠错

HTTP/3 综合实现了 TCP、TLS、HTTP/2 的部分功能，是一个 4/5/7 层的跨层次协议。


## 各类工具目前对 HTTP/3 的支持状况

- [envoy - Add support for terminating QUIC](https://github.com/envoyproxy/envoy/issues/2557)
- [Treafik - Add support for HTTP/3 (http-over-quic) ](https://github.com/traefik/traefik/issues/5514)
- [Caddy - How to implement QUIC protocol using caddy](https://github.com/caddyserver/caddy/issues/3461)


## 参考

- [科普：QUIC协议原理分析](https://zhuanlan.zhihu.com/p/32553477)


