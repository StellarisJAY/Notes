## TCP的特点

1. **面向连接**：TCP是面向连接的运输层协议，通过TCP发送数据需要先建立连接，通信结束后需要释放连接
2. **可靠传输**：TCP实现了可靠传输，使得数据能够无丢失、无差错、不重复地到达接收端
3. **面向字节流**：TCP会将应用层的数据划分成大小不等的数据块，数据块以字节为单位



## TCP连接

使用TCP传输数据需要建立连接，而连接的端点并不是主机，也不是进程，而是叫做套接字（Socket）。

套接字由ip地址和端口号构成，即socket=（ip：端口号）。每一条TCP连接被通信两端的套接字唯一确定。

### 建立连接，三次握手

TCP建立连接时有三次数据传递，所以称为三次握手。三次握手的具体过程如图：


![tcp三次握手.PNG](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a39a2a094d5946569c99dcd5814448ab~tplv-k3u1fbpfcp-watermark.image)

- 第一次握手是由客户端发起的**SYN**，并附带了序列号seq
- 第二次握手是服务端在收到客户端的SYN后，反馈给客户端**SYN/ACK**，并带seq和ack
- 第三次握手是客户端收到服务端的SYN/ACK后，发送**ACK**。
- 当服务端收到了最后一次ACK后，连接就算成功建立了

### 为什么三次握手？

#### **1、避免重复连接报文造成混乱**


![重复连接请求.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7b574ed004734855ab458190d07fee36~tplv-k3u1fbpfcp-watermark.image)

如图上的情况，加入客户端先发起一个连接，seq=100，但是因为某种原因该报文在网络中阻塞，没有到达服务端。

因为长时间没有收到服务端的回复，客户端将重新发起连接，seq=200。

如果此时**seq=100**的报文先到达了服务端，服务端将会返回**ack=100+1**的SYN/ACK报文。客户端收到该报文后会发现ack与自己期望的**ack=200+1**不同，因此客户端会发送**RST报文**中止这次连接。

当**seq=200**到达服务端后，就会正常地完成三次握手。



##### 假如是两次握手会怎样？


![两次握手.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/319e4e6e60c94fe7927a0b0264a1ebbc~tplv-k3u1fbpfcp-watermark.image)

客户端单方面中止了重复的连接，但是服务端并不知道。如果seq=200在之后到达，连接会陷入混乱。

#### **2、确认双方的接收、发送能力**

要确认双方的接收、发送能力，那么双方必须各作为发送方和接收方一次。既然这样就需要两个来回，即四次握手。


![四次握手.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e8e10efcbd684037804cebd26f96c3da~tplv-k3u1fbpfcp-watermark.image)

四次握手的中间两次都是从服务端到客户端的，既然如此可以将这两次握手合并，从而得到了精简的三次握手。



### 释放连接，四次挥手


![tcp四次挥手.PNG](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2201246a666c42648febe6077eb0728a~tplv-k3u1fbpfcp-watermark.image)

TCP连接的释放经历了四次数据传送，所以称为四次挥手。

1. 首先由客户端发起关闭，发送FIN报文，并附带seq=u。
2. 服务端收到后，发送ACK报文，附带seq以及ack=u+1。
3. 当服务端完成剩下的数据传送后，会发送FIN/ACK报文，并附带seq=w，ack=u+1。
4. 客户端收到后，发送ACK报文，seq=u+1，ack=w+1表示收到，然后服务端关闭，连接释放完成。

#### 为什么四次挥手？


![why四次挥手.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/649a695127e643e48cbd2ffa5369a36b~tplv-k3u1fbpfcp-watermark.image)

假如我有一个朋友，我正在与他交流。我想说的话说完了，于是我提出了”我说完了“。朋友此时会回应他知道了，但是他可能没有说完。于是他讲继续讲，直到讲完后说”我也说完了，再见“。此时我再回复”再见“才表示此次交流结束了。

#### 为什么等待2MSL

**MSL（Max Segment Lifetime）最大报文生存时间**，它是一个报文最长的存活时间。

加入客户端最后发送的ACK没有到达，那么服务端会认为是自己的FIN/ACK未送达，所以服务端会重发FIN/ACK。

网络正常的情况下，客户端的ACK最迟在**1MSL**后到达，服务端重发的FIN/ACK也是在**1MSL**后到达。

也就是说，从客户端收到FIN/ACK开始，到客户端接收到服务端第二次发送的FIN/ACK的时间不会超过2MSL。

所以客户端只需要等待2 MSL来处理服务端重发的FIN/ACK即可。


![2MSL.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0dd03fa739384e5bb374631f6e5e02ff~tplv-k3u1fbpfcp-watermark.image)
