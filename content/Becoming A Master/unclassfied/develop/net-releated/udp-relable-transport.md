有空来论坛走走，发现讨论udp可靠传输又热了起来，有人认为udp高效率，有人认为udp丢包重传机制容易控制，还有朋友搞极限测试，当然也有人推销自己的东西，这里写一点我个人的看法。

  udp可靠传输其实非常非常的简单,我最开始接触udp可靠传输大约是在2005年，因为那时候开发FtpAnywhere,由于路由的映射和网关nat处理方面，认为udp具有天生优势，因此开始编写自己的udp可靠传输协议，好象那个时候已经有了udt,我也下了源代码看了下，不过很快就看不下去了，因为它用了定时器，加上跨平台处理，导致它的代码，反正我看着很乱，理不出一个完整的逻辑图。但是原理和tcp的基本一样，并没有什么特殊的。后来我写了我自己的第一个udp可靠传输类，QTUdp,这个是最简单的，也就是和tcp一样，实现2点之间的传输，并不是我现在写的多点之间无中心p2p传输，效率很高，但是当时的编码我还没有养成现在的习惯，用了大量的DWORD之类的数据定义，包括包的定义，从现在角度看，不及格，不过至少它可以实用，并整合到了FtpAnywhere软件中，在调试和运行过程中，慢慢的统计和发现udp包的传输中，最影响性能的部分以及其他一些细节，后来的ut 1.x多点传输协议就是在QTUdp基础上开发的，在到现在 ut 2.x , phoenix 2.x，实现了几个跨越。

  请相信，可靠udp传输从来都不是高效率可靠传输的代名词，影响传输效率的最重要因素在于，sendto函数，每次只能投递一个mtu长度的包，频繁的系统调用极大的影响了极限性能，也许你会说，udp默认可以达到64KB,你可以投递大包，是的，可以投递，但是由于网路上mtu设备的限制，大包会被拆成小包，如果你定义一个包大于mtu,那么当其中任何一个小包发生丢包的时候，会导致整个包需要重传，这个开销非常巨大，特别是在Internet上，而采用mtu大小限制内的包进行传输，丢失一个，只需要重传一个，开销小的多。udp可靠传输的自定义校验是另外一个限制，为了避免伪造的udp包，我们需要在我们自己的可靠udp包中加入自定义的校验，这个校验方法也直接影响到性能，最快的是直接套用crc32校验，由于目前cpu指令集对这个计算进行了优化，因此它的计算速度几乎是最快的，但是代价是，人家要破解或者伪造你的udp包也很容易，因为算法是透明的，以前暴出tcp伪造漏洞也是这样，由于它的包组成是透明公开，唯一有保密性的是序列号，结果有些系统初始序列号存在规律，结果就导致了安全问题。最后，发送和接收，由于是在应用层进行[无论你是采用api epoll select overlap io,最终的执行都是在应用层]，这个发送-确认过程中，由于应用层不是象tcp/ip协议栈那样在内核态运行，因此可能有延迟，不过，目前的cpu核心数量和频率，这个影响在工业应用[internet 等]已经几乎可以忽略，唯一产生影响是在进行本机极限性能测试中。TCP的效率要高过udp可靠传输，因为它的send函数,几乎每次都可以拷贝几十KB,当然你可以将缓冲调整的很大,例如几百KB或者几MB,但是默认情况下的几十KB足够了,想当于几十次调用udp sendto , 而tcp只需要调用一次,其次,tcp的包处理是在内核态进行,确认也是内核态,这就足以与应用层的udp可靠传输拉开距离,更别说硬件层的优化了. 那么,你可能会认为,为什么书上说udp性能好呢?其实这是针对不可靠udp传输,并且是内网 大包,例如

char buffs[32*1024];

memset(buffs,0,32*1024);

sendto(s , buffs,32*1024,.... 发射后不管,无论是否丢包

象这样发包,效率才会超过tcp,不过,主要就是包头大小的差距.

   使用udp可靠传输的目的是为了它的灵活性,在很多协议传输中,例如,远程视频流传输,jrtlib,通常分为关键帧和普通帧,关键帧的丢失,会导致普通帧失去作用,这个时候就需要使用udp的可靠传输+不可靠传输,实现方法是首先以可靠模式投递关键帧,在关键帧处理完成后,剩余的时间用非可靠模式投递普通帧,每隔指定时间重复这一流程. 如果使用tcp,这个效果就不好了,因为网络带宽就那么多,而且带宽变化很大,把所有的关键帧和普通帧都进行可靠传输,可能导致不流畅,网络阻塞等.

  关于udp的穿透能力,也就是传说中的打洞,这根本是个伪命题,有人还在那里搞什么测试说打通了4种 nat 模型, 你认为可能吗? 还有人用猜端口的方法进行打洞,我了个去,工业上能这么用? udp的穿透其实完全取决于路由[网关]的配置,国产的家用或者soho针对p2p进行过调整,但是你用cisco 等专业的大型设备进行打洞看看,为了保护用户的安全,一般管理员都设置了高安全非透明,通常,就算是内部同一台电脑,同一个udp端口,发送给不同的目标ip数据包,路由都会随机重新分配一个端口,打洞根本不可能成功的. 所以,如果你为了nat处理而使用udp,建议你还是放弃吧,直接使用tcp+upnp就行.

  关于udp可靠传输的性能,有人说使用epoll , overlap io 等,效率高于使用select模型, 我可以根据我的这几年开发的经历和测试结果告诉你,在校验模式,投递模式以及数据处理逻辑相同的情况下,多核心cpu平台下,本机效率差距不超过1%,而如果使用在Internet上,效率差距无限接近0,现在是cpu过剩的年代,linux  windows的调度下,如果没有内核态运算,那么就会从等待的线程中挑选一个,你不要以为从内核态切换到用户态是不需要开销的,这个开销同样很大,频繁的调度同样影响性能,虽然不算在你的代码中而算在系统开销中. 当然,如果你需要在linux系统中同时运行apache等web服务,那效率差距会比较明显,因为进程和线程太多,得不到第一时间的调度.

关于udp可靠传输下缓冲大小,说实话,我是第一次见识,最简单的点对点 udp可靠传输,有人开了上百KB的缓冲,缓冲是个好东西,通常,缓冲越大,效率越高,因为一次投递的包数量多,IO性能就高,但是,这在Internet上是不提倡的,这种实现我不知道是否经过严格的网络丢包和负载测试,在有家用路由小带宽上传模式下,如果有多人或者多个网络程序使用带宽,这个延迟参数变动非常频繁,不必要的重传率会非常高,最简单的测试方法,找个铜包铝网线,使用1 1对应的非标准接头法,一头接电脑,一头接路由,然后与Internet上远端进行测试,估计这个丢包重传的概率会高的吓人.

  udp可靠传输比TCP的慢启动好?我想不一定,还是我上面那个网线做测试,如果采用快速启动,在丢包严重的网络环境下,带宽浪费太离谱了,我调整我自己的udp可靠传输启动方法好多次,越来越觉得tcp的慢启动是非常有道理的,虽然恢复的慢,但是,它因为错误重传而产生带宽的浪费是非常小的. 当然,如果考虑到性能,还是可以适当调整的快点.

关于udp可靠传输的文件传输效率,这是个伪命题,这个极限是硬件本身造成的,首先,正常的udp可靠传输效率肯定高于普通硬盘的读写盘速度,即使用memory map 技术对文件执行加速,还是跟不上udp本身的速度,其次,文件传输协议会影响到传输效率,最快的模式是什么? 就是直接发送文件从开始到结尾,和流 [FTP数据连接 HTTP]一样,这是最快的,但是通常,为了避免数据差错,会对文件传输内容进行分块传输,并加入校验.这就影响到了传输效率. 你说一辆公共汽车是从起点到终点直达快?还是一站站的停过去快?很明显,第一个假设脱离实际的.

关于udp可靠传输和cpu的关系,有吗?肯定有,但是在普通应用下,基本没影响,只有在追求极限,例如本机测试,2G+光钎网络等,才会有明显的影响,可问题是,如果你的服务器[电脑]接入的是2G+光钎,你这服务器得什么硬件配置? 在一般应用下,cpu和内存开销以及错误的丢包重传才是第一位的. 如果一个udp可靠传输,100mbps网络下,必须要Intel piii 1Ghz以上,那在这个基础上开发出来的应用可能比现在的电脑游戏还离谱了.  如果不是多对多[因为这个逻辑比普通点对点udp可靠传输复杂的太多],普通的点对点udp可靠传输, 给几个以前的测试数据, QtUdp ,环境是Pentium M 1.7G[单核心], Ati xpress主板 768m ddr2 533[单通道] , 本机器模拟传输当时测试的数据大约在 78MB/s , MTU=1380 , 缓冲是 17 * MTU.  , CPU开销大约是75%, 如果使用目前双核心,估计可以翻倍. 你可能会觉得这个数字太低,但是请注意当时的硬件环境,这个速度已经超过udt好多好多了. 最基本的点对点udp可靠传输,标准crc32校验,以目前的硬件环境, i74核心, DDR3 1600 , 加上我们的一种特殊包处理技术,本机器UDP小包[1400]可靠传输的极限大约在270MB/s,如果还要提升,估计只能和ms 的IIS一样,编写网络驱动了,但是x86处理器和内存速度始终在提升,说不定明天主频直接翻倍了...

其实,我转向udp可靠传输的一个非常重要原因,是IPV6取消了传输中ip层的数据校验,这导致tcp层完全负担起了数据校验任务,TCP数据是否还象IPV4下那么可靠,需要加个问号了,虽然IP层本身的差错率非常低,但是取消掉校验,直接带来的风险到底有多大,并没有经过实践的评估,靠ipv6实验网络得出的是理论结果,也许有一天会出现一种突破性的IPV6 TCP伪造技术. 而如果采用udp可靠传输,由于udp本身有校验,加上我们自己设计的校验和序列号,这个可靠程度完全超过了IPV4 下的TCP,更别提IPV6下的TCP了.

udp可靠传输,其实非常简单,只要你有tcp/ip的基础,最简单的点对点udp可靠传输是非常容易编写的,无非就是封包,校验,发送,确认,重新组包,接口可以模仿tcp的几个函数,这其中的一个难点是,判断是否需要重发,这需要根据以前包的确认时间来推导本包,如果你不希望那么复杂,也可以,经验数字 [不适合网通到电信], 第一次的重传时间 750 ms , 第二次是 1600 , 以后每次都是 2000 , 虽然不科学,但是用这个延迟数字,可以保证你足够的传输效率[即使存在小概率丢包],也不会产生大量的重传,当然最好的方法是从之前包的延迟来推导.

总之,udp可靠传输没那么复杂和神秘,它非常简单,而且,它的效率也没有各位想象中的那么高,可能会让各位失望,但是,它很可爱,你可以随意的塑造你自己的包,实现各种扩展,其中的取舍完全在于聪明的你.

-----------------------------------------------------------------------------------------------------