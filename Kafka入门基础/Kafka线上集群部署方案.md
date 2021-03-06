# Kafka线上集群部署方案

### 操作系统方面

​	Kafka由Scala语言和Java语言编写而成，编译之后的源代码就是普通的.class文件，不同操作系统的差异还是会给Kakfa集群带来比较大的影响。当然Kafka可以部署在Linux、windows、macOS系统上，但是在Linux上部署是最常见的，主要是在I/O模型、数据网络传输、社区支持度上略胜一筹。

​	主流的I/O模型通常有5种类型：阻塞式I/O、非阻塞式I/O、I/O多路复用、信号驱动I/O和异步I/O。每种I/O模型都有各自典型的使用场景，比如Java中Socket对象的阻塞模式和非阻塞模式就对应前两种模型；而Linux中的系统调用select函数就属于I/O多路复用模型；大名鼎鼎的epoll系统调用则介于第三种和第四种模型之间；至于第五种模型，其实很少有Linux系统支持，反而是Windows系统提供了一个叫IOCP线程模型属于这一种。

​	那么I/O模型与Kakfa的关系是什么呢？实际上Kafka客户端底层使用了Java的selector，selector在Linux上的实现机制是epoll，因此在这一点上将Kafka部署在Linux上是有优势的，因为能够获得更高效的I/O性能。其次是网络传输效率的差别，Kafka生产和消费的消息都是通过网络传输的，而消息保存在磁盘，故Kafka需要在磁盘和网络间进行大量数据传输，在Linux部署Kafka能够享受到零拷贝技术所带来的快速数据传输特性，零拷贝就是当数据在磁盘和网络进行传输时避免昂贵的内核态数据拷贝从而实现快速的数据传输。

### 磁盘

​	如果说哪种资源对Kafka性能最重要，磁盘无疑是要排名靠前的，那么Kafka应该选择普通的机械磁盘还是固态硬盘呢？作者的建议是使用普通机械盘即可。因为Kafka是顺序读写操作，一定程度上规避了机械磁盘最大的劣势，即随机读写操作慢，从这一点上来说，使用SSD似乎没有太大的性能优势。

### 磁盘容量

​	Kafka需要将消息保存在底层磁盘上，这些消息默认会被保存一段时间然后自动被删除，虽然这段时间是可以配置的，但是如何结合自身业务场景和存储需求来规划Kafka集群的存储容量？假设公司有个业务每天需要向Kafka集群发送1亿条消息，每条消息保存两份以防止数据丢失，另外消息默认保存两周时间，现在假设消息的平均大小是1KB，可以计算一下：每天1亿条1KB大小的消息，保存两份且留存两周的时间，那么总的空间大小就等于1亿 * 1KB * 2 / 1000 / 1000 = 200GB。一般情况下Kafka集群除了消息数据还有其他类型的数据,比如索引数据等，故再为这些数据预留出10%的磁盘空间，因此总的存储容量就是220GB。既然要保存两周，那么整体容量即为220GB * 14，大约3TB左右。Kafka支持数据的压缩，按照压缩比为0.75，那么最后需要规划的存储空间就是0.75 * 3 = 2.25TB。

### 带宽

​	对于Kafka这种通过网络大量进行数据传输的框架而言，带宽特别容易成为瓶颈。假设公司机房环境是千兆网络，即1Gbps，现在有个业务，其业务目标或SLA是在1小时内处理1TB的业务数据，那么问题来了，你到底需要多少台Kafka服务器？计算一下，由于带宽是1Gbps，即每秒处理1Gb的数据，假设每台Kafka服务器都是安装在专属的机器上，也就是说每台Kafka服务器上没有混部其他服务。根据实际使用情况，超过70%的阈值就有网络丢包的可能性了，故70%的设定是一个比较合理的值，也就是说但台Kafka服务器最多也就能使用大约700Mb的带宽资源，通常我们要额外预留出2/3的资源，即单台服务器使用带宽700Mb / 3 约等于240Mbps。有了240Mbps，我们就可以计算1小时内处理1TB数据所需要的服务器数量了，根据这个目标，我们每秒需要处理2336Mb的数据，除以240，约等于10台服务器。如果消息还需要额外复制两份，那么总的服务器台数还要乘以3，即30台。



