

端口扫描器在今天也有非常多的分支了, 面向不同的用户不同的场景不同的环境. 

- 有专注于端口开放情况的MX1014(TCP), masscan(SYN), 无法解决漏报问题, 或者功能太过单一
- 有传统的老牌全能扫描器nmap, 功能强大但最强的地方不是端口探测, 而是传输层的各种手法
- 有自定义传输层协议的扫描器sx, naabu, 类似go版本的nmap
- 以及这两年很火的各种红队向扫描器: fscan,kscan,yasso等等, 功能较多, 但缺乏打磨
- 内网网段探测扫描器, netspy, 功能较为单一

还有非常多的各类细枝末节的扫描器. 

而gogo的定位则是内网的网段探测,端口探测, 指纹识别与poc验证. 抛弃了各种服务弱口令爆破的功能.

原因是 端口探测, 指纹识别与poc验证 实际上是一条流水线上的工作, 逻辑上前后连续一致, 甚至不会带来多大的性能负担. 各种服务的弱口令则引入了不一致性, 需要新增很多配置参数, 引入很多库, 引入各种各样的代码. 因此,弱口令爆破这一块,将独立实现.

本文将描述gogo设计中需要解决的问题,以及解决的方案.

## 快
在对一些常见的扫描器调研之后以及对编程，网络的原理学习之后，发现快并非是线程数越高越好。要达到快还有很多限制条件以及对应的解决方案。
### 并发
最常见地解决方案就是提高线程数，例如早期扫描器多是单线程，或者10，20的线程数，例如御剑，python开发的一些扫描器以及一些国外的简单发包工具。 虽然他们支持多线程，但是对高线程数的支持并不好。原因可能在于当年协程这一概念没有广泛使用，也有可能是生态中没有简单的解决方案。使用系统的多线程API，将会在多个方面带来大量额外的性能消耗。

1. 需要在内核层与用户层不断切换
2. 不断还原与保存高达4M的线程堆栈
3. 为了线程安全使用的各种锁带来的耗时
...

以及一些我尚不知道的性能消耗，导致多线程的应用很难进行数百甚至数千线程的并发。

在2021年的今天，go提供了简单可靠方便的高并发解决方案，可以在一C1G的低配VPS上实现数千个goroutine的调度。只需要使用go关键字，就能随意的实现数千个协程的并发，协程的调度将由go进行，不再需要系统调度的大量无用消耗。

当然，go的协程调度也不是无损，为了减少这个消耗，我们可以采用复用协程池的方式尽可能地减少消耗.


### 网络
#### tcp拥塞控制
都学过tcp有拥塞控制, 但是这个拥塞控制是针对每一条tcp信道自身的,不会影响到其他tcp信道. 因此只有在传输大数据的时候,才会有性能影响. 对于端口扫描这样的场景, 每条tcp信道并不会有非常多的数据交互, 因此不受tcp拥塞控制影响. 但是网络拥塞会确确实实的影响到扫描准确率. 这是因为刚才提到的路由器的问题, 在路由的过程中, 网络拥塞 每个ttl的时间就会增加.

所以判断网络是否拥塞,可以判断ttl耗时的变化.  
#### 路由器的tail drop
在测试扫描的过程中, 有几次把家里的华为路由器打挂了, 直接重启了.

后来才知道, 扫描的限制可能不来自代码, 也不来自系统, 而是路由器. 如果路由器网络拥塞了, 会采用tail drop(丢到队列末尾的数据包)来告诉客户端的tcp拥塞了,启用tcp拥塞控制,慢点发包. 而如果再负载再大一点, 路由器可能会直接重启. 重启这种问题主要再家用路由器上, 企业级路由器面对几千的并发还是没有任何问题的.
#### tcp keep-alive
http1.1中,实现了keep-alive,不再需要每次http重写建立一次握手,浪费大量资源. 

因此在http端口扫描,指纹识别,以及打poc的过程中, 都可以利用这个keep-alive长连接.

但是对于tcp端口, 则不适用这个keep-alive, 因为某些服务, 只有第一个包正确了才会返回对应的信息, 否则要么server主动断开连接,要么不再返回信息. 所以每个tcp端口扫描都建议重新建立连接.
#### time-wait
time-wait状态代表着tcp连接已经关闭, 但是fd还被占用,等待可能的后续数据处理, 防止有包迷失在网络中被污染下一个使用这个端口的fd.

如果不能正确的处理这个问题, 可能导致fd与端口资源耗尽.


### 系统限制
gogo选择了尽可能不依赖第三方库, 以此来做到最大可能的兼容下保证. 目前可以在linux低版本内核(最低测试到2.x), bsd, 各种版本的mips与arm架构以及 最低到windows server 2003上跑. 应该能覆盖到99%的场景了, 有些系统没有环境去测试, 不过大概率是兼容的. 可以自行手动编译.

go.sum可以看到中有些递归的依赖, 实际上并没有调用到相关函数, 因此在编译的时候会自动跳过.
#### windows 线程调度
windows 的线程调度性能显然不如linux. 这里指的不只是并发控制, 还有tcp堆栈以及其他各种各样的消耗.

在使用windows进行扫描的时候, 经常会导致网络崩溃, 需要好几分钟才能回复. 或者是产生非常大量的漏报, 或者是识别不到http协议,只能建立tcp握手等等问题

在windows进行扫描遇到了非常多的信息, 最终只能降低线程数妥协windows.


#### 最大fd限制
老版本的linux默认的fd限制为1024, 部分新版本的linux发行版改成了65535, 如果要修改需要root权限指定`limits -n 65535`修改.

windows中也有类似的限制, 默认大概是5000, 需要修改注册表修改, 万幸(带引号)的是windows大部分情况根本跑不到4000网络就崩溃了.


#### 65535最大端口数限制
每个系统可用的端口都只有65535个, 而在http扫描的时候, 部分语言,例如go带复用连接池,自动开启keep-alive, 导致端口被长时间占用, 不能正确的松开. 其他一些语言,例如C#也有类似问题.


#### icmp rate limit
todo


## 可拓展
为了可拓展性, xray采用了强依赖go-cel解释器编写poc, 导致poc很难移植, 因此放弃了xray的poc. 部分python poc框架, 例如pocsuite则是直接采用python编写代码, 更难迁移到其他平台. goby提供了一个图形化的poc生成工具, 方便了不少, 不过并不开源.

因此最终poc上选择了兼容nuclei, 好在nuclei的社区是目前最活跃的漏洞风险社区, 囊括了绝大部分用得到的漏洞. 

gogo也在尽可能在其他地方追求可拓展性. 目前可通过配置文件拓展的功能有很多, 可见 [plugin编写](plugin编写.md) 与 [poc编写](poc编写.md)
