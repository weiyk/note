### 一、TCP状态

TCP协议规定，对于已经建立的连接，网络双方要进行四次握手才能成功断开连接，如果缺少了其中某个步骤，将会使连接处于假死状态，连接本身占用的资源不 会被释放。网络服务器程序要同时管理大量连接，所以很有必要保证无用连接完全断开，否则大量僵死的连接会浪费许多服务器资源。在众多TCP状态中，最值得 注意的状态有两个：CLOSE_WAIT和TIME_WAIT。  

1、LISTENING状态
	FTP服务启动后首先处于侦听（LISTENING）状态。
2、ESTABLISHED状态
	ESTABLISHED的意思是建立连接。表示两台机器正在通信。
3、CLOSE_WAIT
    对方主动关闭连接或者网络异常导致连接中断，这时我方的状态会变成CLOSE_WAIT 此时我方要调用close()来使得连接正确关闭
4、TIME_WAIT
    我方主动调用close()断开连接，收到对方确认后状态变为TIME_WAIT。TCP协议规定TIME_WAIT状态会一直持续2MSL(即两倍的分 段最大生存期)，以此来确保旧的连接状态不会对新连接产生影响。处于TIME_WAIT状态的连接占用的资源不会被内核释放，所以作为服务器，在可能的情 况下，尽量不要主动断开连接，以减少TIME_WAIT状态造成的资源浪费。
    目前有一种避免TIME_WAIT资源浪费的方法，就是关闭socket的LINGER选项。但这种做法是TCP协议不推荐使用的，在某些情况下这个操作可能会带来错误。
5、SYN_SENT状态
　 　SYN_SENT状态表示请求连接，当你要访问其它的计算机的服务时首先要发个同步信号给该端口，此时状态为SYN_SENT，如果连接成功了就变为 ESTABLISHED，此时SYN_SENT状态非常短暂。但如果发现SYN_SENT非常多且在向不同的机器发出，那你的机器可能中了冲击波或震荡波 之类的病毒了。这类病毒为了感染别的计算机，它就要扫描别的计算机，在扫描的过程中对每个要扫描的计算机都要发出了同步请求，这也是出现许多 SYN_SENT的原因。

根据TCP协议定义的3次握手断开连接规定,发起socket主动关闭的一方 socket将进入TIME_WAIT状态,TIME_WAIT状态将持续2个MSL(Max Segment Lifetime),在Windows下默认为4分钟,即240秒,TIME_WAIT状态下的socket不能被回收使用. 具体现象是对于一个处理大量短连接的服务器,如果是由服务器主动关闭客户端的连接,将导致服务器端存在大量的处于TIME_WAIT状态的socket, 甚至比处于Established状态下的socket多的多,严重影响服务器的处理能力,甚至耗尽可用的socket,停止服务. TIME_WAIT是TCP协议用以保证被重新分配的socket不会受到之前残留的延迟重发报文影响的机制,是必要的逻辑保证.

### 二、HttpClient超时配置参数说明

- SocketTimeout 是 5s
  	连接建立后，数据传输过程中数据包之间间隔的最大时间
- ConnectTimeout 是 3s
  	连接建立时间，即三次握手完成时间
- ConnectionRequestTimeout 是默认值
  	httpclient使用连接池来管理连接，这个时间就是从连接池获取连接的超时时间
  ![这里写图片描述](https://img-blog.csdn.net/20180103213605325?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHJ5MjAxNQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

虽然报文(“abc”)返回总共用了6秒，如果SocketTimeout设置成5秒，实际程序执行的时候是不会抛出java.net.SocketTimeoutException: Read timed out异常的。
因为SocketTimeout的值表示的是“a”、”b”、”c”这三个报文，每两个相邻的报文的间隔时间没有能超过SocketTimeout
