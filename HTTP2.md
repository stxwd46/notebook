# HTTP/2

<a name="5316c90f"></a>
## HTTP/2 是什么
HTTP/2 很好的解决了当下最常用的 HTTP/1 所存在的一些性能问题，只需要升级到该协议就可以减少很多之前需要做的性能优化工作，当然兼容问题以及如何优雅降级应该是国内还不普遍使用的原因之一。

<a name="c666ad11"></a>
## 特性
<a name="75a7af0d"></a>
#### 二进制传输
HTTP/2 中所有加强性能的核心点在于此。在之前的 HTTP 版本中，我们是通过文本的方式传输数据。在 HTTP/2 中引入了新的编码机制，所有传输的数据都会被分割，并采用二进制格式编码。

![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/20513/1550543731234-553f0bdd-ee11-4464-a8e0-3ed2af2068a2.png#align=left&display=inline&height=314&name=image.png&originHeight=459&originWidth=874&size=127377&width=597)
<a name="b3946b88"></a>
#### 多路复用
在 HTTP/2 中，有两个非常重要的概念，分别是帧（frame）和流（stream）。<br />帧代表着最小的数据单位，每个帧会标识出该帧属于哪个流，流也就是多个帧组成的数据流。<br />多路复用，就是在一个 TCP 连接中可以存在多条流。换句话说，也就是可以发送多个请求，对端可以通过帧中的标识知道属于哪个请求。通过这个技术，可以避免 HTTP 旧版本中的[队头阻塞](https://www.jianshu.com/p/450cc7320e30)问题，极大的提高传输性能。

![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/20513/1550543824554-7c46fafe-e153-4ded-a2e3-c1ced6eccdb6.png#align=left&display=inline&height=172&name=image.png&originHeight=138&originWidth=494&size=16673&width=616)

<a name="0936972c"></a>
#### Header 压缩
在 HTTP/1 中，我们使用文本的形式传输 header，在 header 携带 cookie 的情况下，可能每次都需要重复传输几百到几千的字节。<br />在 HTTP /2 中，使用了 HPACK 压缩格式对传输的 header 进行编码，减少了 header 的大小。并在两端维护了索引表，用于记录出现过的 header ，后面在传输过程中就可以传输已经记录过的 header 的键名，对端收到数据后就可以通过键名找到对应的值。

<a name="3bda4bd8"></a>
## 弊端
虽然 HTTP/2 解决了很多之前旧版本的问题，但是它还是存在一个巨大的问题，虽然这个问题并不是它本身造成的，而是底层支撑的 TCP 协议的问题。<br />因为 HTTP/2 使用了多路复用，一般来说同一域名下只需要使用一个 TCP 连接。当这个连接中出现了丢包的情况，那就会导致 HTTP/2 的表现情况反倒不如 HTTP/1 了。<br />因为在出现丢包的情况下，整个 TCP 都要开始等待重传，也就导致了后面的所有数据都被阻塞了。但是对于 HTTP/1 来说，可以开启多个 TCP 连接，出现这种情况反到只会影响其中一个连接，剩余的 TCP 连接还可以正常传输数据。<br />那么可能就会有人考虑到去修改 TCP 协议，其实这已经是一件不可能完成的任务了。因为 TCP 存在的时间实在太长，已经充斥在各种设备中，并且这个协议是由操作系统实现的，更新起来不大现实。<br />基于这个原因，Google 就更起炉灶搞了一个基于 UDP 协议的 QUIC 协议，并且使用在了 HTTP/3 上。
