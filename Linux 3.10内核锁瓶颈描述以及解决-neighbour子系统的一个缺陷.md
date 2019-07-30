---
title: Linux 3.10内核锁瓶颈描述以及解决-neighbour子系统的一个缺陷
date: 2019-06-07 08:41:17
tags: CSDN迁移
---
 版权声明：本文为博主原创，无版权，未经博主允许可以随意转载，无需注明出处，随意修改或保持可作为原创！ https://blog.csdn.net/dog250/article/details/91047124   
  ZheJiang WenZhou skinshoe 👞 wet，rain ☔️flooding water will not fat！

 
--------
 事情过去很久了，2016年初的事了，当时排查了一起自旋锁导致CPU飙高的问题，当时没有把问题记录下来，现在趁着假期重新回顾。

 引用下面的一篇文章，隐隐约约记录着些来龙去脉。  
 _**Linux3.5内核对路由子系统的重构对Redirect路由以及neighbour子系统的影响：**_ [https://blog.csdn.net/dog250/article/details/50754780](https://blog.csdn.net/dog250/article/details/50754780)

 好吧，下面才是细节。

 
### []()正文

 别以为数据包出了路由子系统，拿到了 _**下一跳的dst entry**_ 就从此大吉大利了，七七八十一难还差四十九难呢！

 数据包在通过了路由查找逻辑后，下一跳dst entry会附着在数据包skb本身，指导数据包真正发送。具体的发送逻辑由 _**邻居子系统**_ 来统筹安排。

 不同的网卡类型对接的是不同的网络，这意味着数据包从不同的网卡发送时的行为将有所不同。比如对于以太网或者NBMA网络这种多点接入的网络，需要一个明确的gateway作为下一跳，否则数据包不知道发给谁：

 
```
ip route add 100.100.100.0/24 via 192.168.56.40

```
 而对于类似点对点的网络，数据包只需要从简单发送到网线上即可，因为对端肯定只有一个设备，它不收谁收:

 
```
ip route add 100.100.100.0/24 dev tunp2p

```
 比较复杂的情况是上述的多点接入网络，由于需要指定一个明确的下一跳，比如上述例子中的192.168.56.40，而和这个所谓的下一跳同等地位的设备可能会非常多，为了保证数据包仅仅发给它而不是错误地发给别的设备，那么就需要一种机制来保证在更底层的协议层面，保证正确的寻址，从而确保数据包被正确投递。

 不用说大家也知道，这就是 _**地址解析协议**_， 对于以太网而言，解析IPv4下一跳用的是ARP协议，解析IPv6下一跳用的是ICMPv6协议(注意！不存在ARPv6！)。

 Linux作为常用的服务器系统以及lastmile转发设备的系统，几乎最常遇到的网络类型就是以太网了。而Linux的ARP可以说是实现的非常完备，没啥说的，可以说，Linux内核的邻居解析就是 _**专门为以太网设计的**_。这点不多说，如果谁手边有Linux系统的网卡直接接入了非以太网，比如老式的X.25这种，请求让我登录玩玩。

 那么，当Linux面对非以太网设备时，比如POINTOPOINT设备时，就玩不转了。毕竟这是Linux非典型的应用场景。

 在描述问题之前，我们先来看一下迄至5.0-rc2版本的Linux内核邻居子系统是 _**如何定义邻居**_ 的。

  
  * **对于多点接入网络：下一跳gatewat即邻居** 
  * **对于点对点网络：目标IP即邻居**  如果再看看Linux内核是如何管理这些邻居的，会出现什么问题就一目了然了。还是老方法，一张图足以解释，胜过代码分析：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190606172327383.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RvZzI1MA==,size_16,color_FFFFFF,t_70)  
 试想一下一个极端情况，你在一个满负载B类网段的同一个以太网同时往65534台机器同时发包会怎样，答案是大量的邻居会被创建，争抢那些write lock！

 事实上，没人会用这么大一个以太网，要是真用了，光是各种广播就把网络给flood掉了吧。我们平时常用的C类网段在服务器动辄24+核的机器上，不足以让这个问题暴露。

 那么我们如何复现这个问题以证实它确实是个问题呢？这也简单，我们用虚拟的POINTOPOINT设备来复现。

 **被测机，设为C1：**

  
  * 配置：  
```
ip addr add dev enp0s3 192.168.56.101/24

ip tun add test mode ipip remote 1.1.1.2 local 1.1.1.1
ip link set test up
ip add add 2.2.2.1 brd 255.255.255.255 peer 2.2.2.2 dev test

iptables -t mangle -A OUTPUT -p udp -j MARK --set-mark 100
iptables -t mangle -A OUTPUT -p udp -j MARK --set-mark 100

ip ru add fwmark 100 tab vtab

ip r add 0.0.0.0/0 dev test tab vtab

```
 **测试机，设为C2：**

  
  * 配置：  
```
ip addr add dev enp0s3 192.168.56.201/24

```
 跑下面的脚本，绑定不同的源IP地址往C1上发包，触发C1往不同IP回包创建邻居，可以多跑几个实例。脚本如下：

 
```
#!/usr/bin/python
from scapy.all import *
import socket

msg = "aa"

while True:
    s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    s.setsockopt(socket.IPPROTO_IP, 19, 1)
    addr = RandIP()
    try:
        s.bind((str(addr), 1234))
    except Exception as e:
        pass
    address = ('192.168.56.101', 31500)  
    try:
        # 往C1发包，触发其疯狂回复ICMP unreachable，以创建邻居
        s.sendto(msg, address)
    except Exception as e:
        #print e
        pass
    s.close()

```
 此时，在C1上执行：

 
```
ip nei ls nud noarp

```
 你将绝望地看到一刹那时间数以万计的neigh被创建了出来，并且这个过程会一直持续，很快触发GC，于是乎大家都在争锁。

 top结果以及perf top的结果请测试后自行拉取。

 和之前关于IPv6以及inet peer的问题几乎一模一样，都是锁的不合理使用导致的，就连我这几幅图都是同一个图复制小修小改的。

 光破不立不是真本事，既然是锁导致的，那就解锁呗。

 其实，这个问题我在2016年初就遇到并解决了，参见：  
 [https://blog.csdn.net/dog250/article/details/50754780](https://blog.csdn.net/dog250/article/details/50754780)  
 今天只是重新梳理，就着几天前解决的那个IPv6的soft lockup问题，我发现它们竟然属于同一类，所以就做个一致性的总结。

 当时还提了一个patch：  
 [https://lore.kernel.org/patchwork/patch/652657/](https://lore.kernel.org/patchwork/patch/652657/)  
 然而并没有人搭理我这种虽然对内核感兴趣但并不care它的外人。David Miller不理人。倒是西邮王聪说了几句，他的意思是说，我这个patch回滚了David Miller的一个bugfix：

 
> Well, you just basically revert another bug fix:  
>    
>  commit 0bb4087cbec0ef74fd416789d6aad67957063057  
>  Author: David S. Miller [davem@davemloft.net](mailto:davem@davemloft.net)  
>  Date: Fri Jul 20 16:00:53 2012 -0700
> 
>  
 但如果真的回滚了一个bug的fix，那为什么不采用另外的方案呢？

 我还被怼了几句，内核社区就一熟人社区，谁认识谁啊！我超级看不惯那些社区的人，牛X轰轰的。

 时间到了2018年，又有人提出了类似的patch：  
 [https://patchwork.ozlabs.org/patch/860390/](https://patchwork.ozlabs.org/patch/860390/)  
 同样的，没有得到回应！

 正确的做法应该是这个样子的：

 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190606172838569.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RvZzI1MA==,size_16,color_FFFFFF,t_70)  
 修正这个问题，非常之简单，patch如下：

 
```
diff --git a/net/ipv4/ip_output.c b/net/ipv4/ip_output.c
index 64878ef..d7c0594 100644
--- a/net/ipv4/ip_output.c
+++ b/net/ipv4/ip_output.c
@@ -202,6 +202,8 @@ static int ip_finish_output2(struct net *net, struct sock *sk, struct sk_buff *s
 
 	rcu_read_lock_bh();
 	nexthop = (__force u32) rt_nexthop(rt, ip_hdr(skb)->daddr);
+	if (dev->flags & (IFF_LOOPBACK | IFF_POINTOPOINT))
+		nexthop = 0;
 	neigh = __ipv4_neigh_lookup_noref(dev, nexthop);
 	if (unlikely(!neigh))
 		neigh = __neigh_create(&arp_tbl, &nexthop, dev, false);

```
 问题是解决了，然而这不算完。

 需要明确的几个问题：

  
  * 为什么内核社区没有发现这个问题？ 
  * 这个问题是固有的，还是半途被引入的？  又该溯源了。

 来吧，看看这个：  
 [https://lists.openwall.net/netdev/2012/07/20/199](https://lists.openwall.net/netdev/2012/07/20/199)

 ![在这里插入图片描述](https://img-blog.csdnimg.cn/2019060617292288.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RvZzI1MA==,size_16,color_FFFFFF,t_70)  
 就是这样被引入的。其实在上面贴的那个 _**ipv4: Make neigh lookup keys for loopback/point-to-point devices be INADDR_ANY**_ patch中，也有提到：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190606172949578.png)  
 David Miller的风格我是不敢苟同的，他可能是一个优秀的代码设计者，却不是一个优秀的工程师，而绝顶的高手，一般是all for one的类型，很遗憾，显然David Miller并不是，他只是一个coder。

 这并不是David Miller的第一次，前文中我描述的IPv6的问题，都是如此风格一致地引入了锁的问题。在添加新的功能或者解决旧的Bug时，David Miller习惯于以下的策略：

  
  * **试图用一种统一的方式，去处理所有的场景，以保持代码的简洁明了！**  当然，所有的maintainer都这样。换位思考，也不是不应该。

 在IPv6的soft lockup问题的引入中，David Miller采用了 _**"always"**_ 这个词，而在这个neigh lock问题的引入中，David Miller采用了 _**"entirely"**_ 这个词，这足以显现其风格。

 以至于，再出现类似的问题导致的故障，我必须去review一下David Miller提交的所有patch了，十有八九会中标。

 
### []()缘由

  
  * _**2016年初解决了这个neigh锁导致的CPU stall问题**_ 
  * _**2019年初解决了那个IPv6 rt cache锁导致的CPU stall问题**_  两件事看起来都是离散的独立事件，但是在解决完IPv6的rt cache CPU stall问题后，总觉得似曾相识…

 今天终于回忆起了2016年初解决的neigh锁的问题，并进行了溯源，发现二者竟然是兄弟问题啊！两件事联系起来，竟然貌似得到了一种 _**解决问题并引入Bug的模式**_ ，哈哈，原来写Bug也是和经验以及风格相关的哦！

 
--------
 ZheJiang WenZhou skinshoe 👞 wet，rain ☔️flooding water will not fat！

   
  