# TCP 协议

# 建立连接

为了保证可靠连接，tcp建立连接需要“三次握手”，这三次握手对tcp的性能有至关重要的影响。
如何缩小这三次握手给性能带来的影响是网络性能调优的一个重要关注点。

### 三次握手

* 客户端发送`SYN(SEQ=x)`报文给服务器端，进入`SYN_SEND`状态。

net.ipv4.tcp_syn_retries: syn报文的重传次数.(至于重传机制,暂时还没搞清楚)  
net.ipv4.tcp_fastopen: 开启这个选项可以使tcp在第一个syn包中就开始传送数据.(关于这个选项还需要进一步研究:是否需要客户端也开启此选项)

* 服务器端收到SYN报文，进入`SYN_RECV`状态，回应一个`SYN(SEQ=y)ACK(ACK=x+1)`报文。

net.ipv4.tcp_max_syn_backlog: 服务器中syn队列的长度.当`syncookies`使能时,此时并不会有一个合法的最大值,因此这个值会被忽略.
详见`man listen`  
sysctl -w net.ipv4.tcp_syncookies=1 ,　打开syncookie，在syn backlog队列不足的时候，提供一种机制临时将syn链接换出  
net.ipv4.tcp_synack_retries: synack报文的重传次数.(同样地,还未搞清楚重传机制)  
net.core.somaxconn:  这个参数限制了listen()函数的syn队列默认值为128,需适当提高这个值.


* 客户端收到服务器端的SYN报文，回应一个`ACK(ACK=y+1)`报文，进入`Established`状态。

至此，整个连接建立完成。



下面分析“三次握手”的过程中包含的几种状态：

* SYN_SEND 
* SYN_RECV
* ESTABLISHED

# 拥塞控制

拥塞控制机制是传输层为了保证网络质量而施加的保护措施，但是这样一来，
整个拥塞控制机制会使得tcp连接在很多时候难以发挥出最好的性能，
如何减小tcp拥塞控制带来的性能损失，也是tcp性能调优的一个关键点。

拥塞控制的几个关键词

慢开始，拥塞预防，快恢复，快重传

tcp_available_congestion_control: 系目前支持的拥塞控制算法列表.  
tcp_congestion_control: 用来设置当前使用的拥塞控制算法.  
tcp_allowed_congestion_control: 设置可选的拥塞控制算法列表.  
net.ipv4.tcp_window_scaling: 开启窗口缩放,以支持更大范围的接收窗口.  
net.ipv4.tcp_slow_start_after_idle: 关闭慢启动重启.


拥塞控制的几个关键参数：

* initcwnd 初始拥塞窗口 
设置初始拥塞窗口: `ip route change`  
* ssthresh 慢开始门限
* rwnd 接收窗口大小.

增大TCP连接的`initcwnd`可以有效提升网络的性能,代价就是这样会使得整个网络有可能变得拥堵.
同样地,增大`ssthresh`也可以提升网络性能.

需要注意的是,以上两个参数调整的是系统自己的设置,比如`initcwnd`只能调整自己发送报文的速度,
并不能控制对端对报文的接收速度.

# 释放连接
TCP连接的释放也是一个复杂的过程，在这个地方，提升网络性能的关键是如何快速释放一个不再使用的连接，
或者如何重用一个连接。

* 客户端发起关闭请求，发送一个信息：`FIN(m)`，此时客户端进入`FIN_WAIT1`状态,服务器端在接收到FIN后,进入`CLOSE_WAIT`状态。
* 服务端接受到信息后，首先返回`ACK(m+1)`,表明自己已经收到消息。客户端在收到服务器端的ACK之后,进入`FIN_WAIT2`状态.

net.ipv4.tcp_fin_timeout: 可设定客户端在`FIN_WAIT2`状态下最多持续多长时间后会自动关闭该连接.

* 服务端在准备好关闭之前，最后发送给客户端一个`FIN(n)`消息，询问客户端是否准备好关闭了。此时服务器进入`LASK_ACK`状态.
* 客户端接受到服务端发送的消息后，返回一个确认信息: `ACK(n+1)`,此时客户端进入`TIME_WAIT`状态.

net.ipv4.tcp_tw_recycle: 快速回收`TIME_WAIT`状态下的连接.  
net.ipv4.tcp_tw_reuse: 允许重用`TIME_WAIT`状态下的连接,当然需要该连接是"协议安全的"(关于什么是协议安全,去读源代码)  
tcp\_max\_tw\_buckets: 任意时刻系统中最多允许存在的 `TIME_WAIT`状态的tcp连接.

* 最后，服务端和客户端在双方都得到确认时，再经过一定时间之后,双方状态都变为`CLOSED`,此时连接关闭。

# 特殊机制
tcp中还存一些特殊机制来应对各种不同的情况，这些地方的性能问题也是值得考察的。

## KeepAlive
KeepAlive 是tcp协议中用来检测tcp连接是否仍然处可用状态的机制.

net.ipv4.tcp_keepalive_time: 如果连接持续这么长时间没有数据传输,那么就开始发送检测报文.默认为两个小时,太长了,需要缩小.  
net.ipv4.tcp_keepalive_probes: 当系统发送多少次 keepalive 消息而没有收到回复，此时认为该连接已失效。
默认值为：9 ，也就是说，9个 keepalive 消息没有得到回应，服务器认为这个连接已经失效，将关闭这个连接。  
net.ipv4.tcp_keepalive_intvl: 系统重新发送 keepalive 消息的时间间隔，默认为 75 s.  

## SACK 选择重传
net.ipv4.tcp_sack: 启用选择重传机制,这个机制可以使系统只重传真正丢失的数据报.  
net.ipv4.tcp_thin_dsack: 允许发送两个sack数据报

# 其他影响性能的问题
## 孤儿套接字
net.ipv4.tcp_max_orphans: 系统中最大的孤儿套接字数量,在调节时不要降低这个值,需要注意的是,每个孤儿套接字会吃掉约64k内存,
这个值的初始设置为和`NR_FILE`相等,详见`man tcp`  
net.ipv4.tcp_orphan_retries: 当对端变成一个孤儿套接字时重试多少次直到关闭我们的tcp连接.

## 内存使用
net.ipv4.tcp_meme:  
net.ipv4.tcp_rmem:  
net.ipv4.tcp_wmem:  
net.core.wmem_max:  
net.core.rmem_max:  

## 其他参数限制
因为Linux系统中**一切皆是文件**,所以系统中对文件的许多限制也会间接地限制到socket连接.
net.nf_conntrack_max: 系统支持的最大连接数

# kernel中与网络有关的文件