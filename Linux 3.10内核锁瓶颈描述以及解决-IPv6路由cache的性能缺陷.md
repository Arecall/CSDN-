---
title: Linux 3.10内核锁瓶颈描述以及解决-IPv6路由cache的性能缺陷
date: 2019-06-07 02:11:37
tags: CSDN迁移
---
 版权声明：本文为博主原创，无版权，未经博主允许可以随意转载，无需注明出处，随意修改或保持可作为原创！ https://blog.csdn.net/dog250/article/details/91046131   
  大量线程争抢锁导致CPU自旋乃至内核hang住的例子层出不穷。

 我曾经解过很多关于这方面的内核bug：

  
  * nat模块复制tso结构不完全导致SSL握手弹证书慢。 
  * IP路由neighbour系统对pointopoint设备的处理不合理导致争锁。 
  * IPv6路由缓存设计不合理导致争锁。 
  * Overlayfs的mount设计不合理导致争锁。  凌晨将近两点半，本文再解一例。即描述 _**IPv6路由cache的设计缺陷**_ 问题以及相应的解法。

 
--------
 IPv4的路由cache在3.6内核及以后被彻底干掉了。但是IPv6的路由cache并没有被干掉，相反，IPv6的路由cache存在非常大的性能问题，直到4.2-rc1版本才被解决。

 这是一件非常有意思的事情，整个故事听起来非常好玩，非常值得一读。但是由于本文并不是技术文档，所以我不会去介绍什么是IPv6，也不会去分析IPv6路由查找逻辑的代码。显然，这些都是前置知识，这是读懂这个故事的前提。

 
--------
 被广泛部署的CentOS 7.2所使用的3.10内核，IPv6路由表的实现，存在性能缺陷！

 我先给出3.10内核的IPv6路由查找逻辑的框图：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190606162103150.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RvZzI1MA==,size_16,color_FFFFFF,t_70)

 看上去超级复杂，看上去很有技术含量的一个逻辑。好吧，说说要点：

  
  * 3.10内核的IPv6路由Trie树的操作采用读写锁。 
  * 3.10内核实现的IPv6路由表项和路由cache，被保存在同一棵Trie树里。 
  * 3.10内核针对查找到的结果路由项执行cow生成路由cache项，插回Trie树。  这个结构以及这些操作适合读写锁的场景吗？换句话说，这是读多写少的场景吗？

 完全不是！

 _**如果IPv6源地址分布离散**_ ，按照源/目的地址二元组定义一个路由cache项的方法，意味着每一次读之后都会伴随着一次写：

  
  * 读：查找路由，企图命中路由cache，读锁。 
  * 结果：由于源地址离散，未命中路由cache，而命中了FIB。 
  * 写：生成路由cache项，插回Trie，写锁。  这便是soft lockup的根源！

 注意，触发缺陷的条件是 _**源地址分布离散！**_ 这非常容易满足，因为：

  
  * IPv6地址空间巨大。 
  * IPv6原则上不会做NAT。  下面是一个复现脚本：

 
```
#!/usr/bin/python
from scapy.all import *
import socket

msg = "hello"

while True:
    s = socket.socket(socket.AF_INET6, socket.SOCK_DGRAM)
    # 75的意思是IPV6_TRANSPRANT，即可以bind任意源地址！
    s.setsockopt(socket.IPPROTO_IPV6, 75, 1)
    # 随机生成IPv6源地址
    addr = RandIP6("**")
    try:
        s.bind((str(addr), 1234))
    except Exception as e:
        pass
    address = ('2010:222:2002::xxx', 31500)
    try:
        # 这里将包发出去后，被压的机器将会目睹超级多的源/目标向其袭来！
        s.sendto(msg, address)
    except Exception as e:
        pass
    s.close()

```
 
--------
 问题分析清楚了，看看解法吧。下面给出我的解法中的框图：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190606163236925.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RvZzI1MA==,size_16,color_FFFFFF,t_70)

 完美解决问题，解法非常简单：

  
  * 干掉IPv6的路由cache！  再跑下脚本？

 
--------
 现在故事开始了，我来讲述一下这个性能缺陷的来龙去脉，让我们一同领略David Miller的大师风采吧！

 
--------
 程序员讨厌说历史，但历史地看问题是一种方法论，理清脉络方知问题出在哪儿，如何修正便是信手拈来了。

 如果说IPv4是在3.5版本内核之后，3.6版本内核开始废掉了路由cache，那么对于IPv6而言，同样含义的版本号则是4.2-rc1.即，在4.2-rc1版本内核之后，IPv6的路由cache几乎不再起作用了。但是IPv6对路由cache的废弃并没有IPv4做的那样决绝，那般彻底。

 让我们先从问题说起。

 本文开头，我展示了3.10内核IPv6路由cache机制存在的缺陷，该缺陷会导致系统软锁的问题。在处理完这个问题后，我决定挖一下坟，以便为我的解决问题的patch寻找一些支撑或者说可行性的依据：  
 _**这个问题是固有的呢，还是中间被引入的呢？**_

 _**如果问题是固有的，那么我相当于按照正确的思路完成了一个正确的事情，如果它是被引入的，那么我想看看引入这个问题前，事情是什么样子的。**_

 于是，我review并check了从2.6.18开始的主要kernel changelog以及patchwork上的讨论，最终发现是David Miller在2.6.38-rc3中将这个不算Bug的Bug引入了内核，对，就是他，David S. Miller！一点都不平易近人，远不如Google大神Neal Cardwell那般nice的David Miller。

 他的patch如下：  
 [http://git.emacinc.com/Linux-Kernel/linux-emac/commit/d80bc0fd262ef840ed4e82593ad6416fa1ba3fc4](http://git.emacinc.com/Linux-Kernel/linux-emac/commit/d80bc0fd262ef840ed4e82593ad6416fa1ba3fc4)  
 下面的链接展示了一个集合：  
 [https://lwn.net/Articles/425770/](https://lwn.net/Articles/425770/)

 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190606164424660.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RvZzI1MA==,size_16,color_FFFFFF,t_70)  
 接着看：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190606164545808.png)

 来来来，看下究竟是个什么东西：

 
```
diff --git a/net/ipv6/route.c b/net/ipv6/route.c
index 373bd0416f69f9ad7e4645eebb574a3ec4eb4127..1534508f6c68a3c4f010657e94051e06a7d727c4 100644
--- a/net/ipv6/route.c
+++ b/net/ipv6/route.c
@@ -72,8 +72,6 @@
 #define RT6_TRACE(x...) do { ; } while (0)
 #endif
 
-#define CLONE_OFFLINK_ROUTE 0
-
 static struct rt6_info * ip6_rt_copy(struct rt6_info *ort);
 static struct dst_entry	*ip6_dst_check(struct dst_entry *dst, u32 cookie);
 static unsigned int	 ip6_default_advmss(const struct dst_entry *dst);
@@ -738,13 +736,8 @@ restart:
 
 	if (!rt->rt6i_nexthop && !(rt->rt6i_flags & RTF_NONEXTHOP))
 		nrt = rt6_alloc_cow(rt, &fl->fl6_dst, &fl->fl6_src);
-	else {
-#if CLONE_OFFLINK_ROUTE
+	else
 		nrt = rt6_alloc_clone(rt, &fl->fl6_dst);
-#else
-		goto out2;
-#endif
-	}
 
 	dst_release(&rt->dst);
 	rt = nrt ? : net->ipv6.ip6_null_entry;

```
 正是这个 _**rt6_alloc_clone函数**_ 的调用，让事情开始变的悲哀。

 2.6.38-rc3以前的内核，如果不编译CLONE_OFFLINK_ROUTE这个宏，路由cache是不会被插入Trie树的，被David Miller这么一改，故事就开场了！这一去就是7年多啊！期间，我们可以Redhat官方网段看到一个相关的Bug报告：  
 [https://access.redhat.com/solutions/1985663](https://access.redhat.com/solutions/1985663)

 都是无条件调用rt6_alloc_clone惹的祸！

 换句话说，把这个patch回退，就是我在前文中所说的解决问题的我的patch，当然我还做了一些额外的其它处理。

 现在的问题是，David Miller为什么要引入这么一个patch呢？显然，他肯定不是故意的。是他水平不够吗？显然他是高手。

 肯定是发生了什么事情，为了解决某一个问题而引入的这个patch，然而解决一个问题却导致另一个问题的事情却经常发生，日光之下，并无新事！

 下面是2.6.38-rc3这个patch的一点线索：  
 [https://groups.google.com/forum/#!searchin/fa.linux.kernel/CLONE_OFFLINK_ROUTE/fa.linux.kernel/EBkPjNM6dp0/pzK57imIFGoJ](https://groups.google.com/forum/#!searchin/fa.linux.kernel/CLONE_OFFLINK_ROUTE/fa.linux.kernel/EBkPjNM6dp0/pzK57imIFGoJ)  
 [http://patchwork.ozlabs.org/patch/80293/](http://patchwork.ozlabs.org/patch/80293/)

 那么顺着这个2.6.38-rc3的线索开始，经由3.10等稳定版内核，这个IPv6路由cache的soft lockup问题一直存在着，虽然Redhat的3.10会回移上游的patch，然而却不包括修复这个问题的patch. 这非常令人遗憾。

 好几年的时间，人们屡次碰到这个问题，却一直没有看到有解法…2018年2月份时，有人提出了一个patch：  
 [https://patchwork.ozlabs.org/cover/877605/](https://patchwork.ozlabs.org/cover/877605/)  
 这个比较有意思，可以看看。作者提到：

 
> IPv6 uses the same data struct for both control plane (FIB entries) and  
>  data path (dst entries). This struct has elements needed for both paths  
>  adding memory overhead and complexity (taking a dst hold in most places  
>  but an additional reference on rt6i_ref in a few). Furthermore, because  
>  of the dst_alloc tie, all FIB entries are allocated with GFP_ATOMIC.
> 
>  
 然而作者却没有说：

 
> IPv6在运行时使用同一棵树保存路由项和路由cache。
> 
>  
 此外，下面这个patch也是解决问题的关键：  
 [https://git.kernel.org/pub/scm/linux/kernel/git/davem/net-next.git/commit/?id=45e4fd26683c9a5f88600d91b08a484f7f09226a](https://git.kernel.org/pub/scm/linux/kernel/git/davem/net-next.git/commit/?id=45e4fd26683c9a5f88600d91b08a484f7f09226a)  
 只要看题目就知道其意义了：

 
> **ipv6: Only create RTF_CACHE routes after encountering pmtu exception**  
>  This patch creates a RTF_CACHE routes only after encountering a pmtu  
>  exception.
> 
>  
 奋六世之余烈之后，在4.2-rc1中，事情突然(注意，这里不是悄悄地起了变化)起了变化，我们review这个版本的ip6_pol_route这个函数，会发现它和之前的实现有大不同，查找到的结果路由项不再往Trie树里插入了。Trie上的每一个路由项节点，都携带一个percpu的rt cache，可以直接按照percpu的方式获取之，附着在skb上，然后直接xmit，这看起来比2.6.38-rc3之前的做法更加高明，当然，也比我的实现更加高明，因为我的思路和2.6.38-rc3之前的处理逻辑几乎是一致的。

 不过，我依然觉得这个变化来的太慢了。我始终想不通为什么这么一大群高手这么多年却不能秒解David Miller一行代码引入的如此明显的问题，而同样对待这件事，作为一个不会编程的半吊子内核爱好者的我，却可以定位到读写锁的问题，这有点不符合逻辑。

 这个问题很明显，只需要perf top看下热点，然后看一遍代码，问题就能浮出水面，我觉得从2.6.38到4.2，整整24个大版本，没能把问题修复，这有点不可思议。

 4.2-rc1的关于ipv6路由cache的更新如下：

 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190606165152951.png)

 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190606165218934.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RvZzI1MA==,size_16,color_FFFFFF,t_70)

 [https://gitlab.ic.unicamp.br/lkcamp/linux-staging/commit/c1a34035506d3a7ad62403125d59c86b763c477d](https://gitlab.ic.unicamp.br/lkcamp/linux-staging/commit/c1a34035506d3a7ad62403125d59c86b763c477d)  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190606165300658.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RvZzI1MA==,size_16,color_FFFFFF,t_70)  
 注意下面的红色框框里的描述：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190606165314727.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RvZzI1MA==,size_16,color_FFFFFF,t_70)

 当然，在4.2版本内核之后，关于IPv6的路由cache的实现是持续优化的，它再一次发生了变化，这就跟IPv4的处理逻辑非常类似了，不再cache路由结果，而是直接使用，用完后回收到percpu专门的数据结构，而不是释放到slab。

 每一个路由项都会关联一个percpu的copy，这意味着路由项和percpu rt cache即实现了天生关联又分了层，这应该就是正确的做法吧。

 那么剩下的问题是什么？解决了这个问题之后，还有别的问题吗？

 我先重申一下，soft lockup的问题本质不在Trie树，而在锁，而锁的原因则是路由cache的滥用。那么当锁的问题解决了之后，接下来才会正面应对Trie树的查询效率问题。不过我认为，不会有太大问题，Trie已经在IPv4路由上被考验过了，而IPv4的路由表是炸裂式的，乱得很，IPv6在地址规划的时候就自然考虑了聚合，不会像IPv4早年那样乱分一通，所以IPv6的路由表会更加紧凑，这对于组织良好的Trie树结构的非常有帮助的。

 所以，我对Trie的性能是满怀信心，我相信它不会轻易畸化，当然了，遇到有备而来的DDoS，另说咯，这就是另一个话题了。

 
--------
 解决了David Miller的问题之后，还有一个问题，那就是peer的问题。

 IPv6的事情并没有由于貌似解决了soft lockup问题而结束，相反，一切才刚刚开始！

 我们需要对比IPv6和IPv4在同等压力下的性能表现。

 那好，我们看看同样脚本同样压力的IPv4路由处理时CPU利用率：_**IPv4狂暴IPv6的性能**_

 为什么IPv6路由处理性能没有IPv4好？肯定还有哪里是热点！

 于是就用perf top来看咯：

 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190606165916726.png)  
 inet_getpeer又来捣乱！干掉它就是了！Linux内核代码里很多不合理的东西，干掉保持清洁。

 inet peer和路由的关系又是什么呢？它怎么会影响移除rt cache后的处理流程呢？带着这样的问题，我重新check了ip6_pol_route这个查找路由的核心函数。原来，我在第一个版本中虽然干掉了rt cache，但是并没有做干净，事情是这样的。

 IPv6为每一个接入的源IP/目标IP都保留了一个二元祖作为rt cache项插入到Trie树中，每一项看上去是下面的样子，用ip -6 r l c 可以得到：

 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190606170037135.png)  
 这种项太多了，都会被插入到同一棵树中，这也就是前文提到的soft lockup问题的根源之所在，与此同时，每一个rt cache项还将绑定一个peer结构体，即以目标IP为健值的元素，该peer结构体里面保存一些和该目标IP相关的信息，比如到达它的MTU，到达它的RTT，到达它的CWND等等。

 由于peer的很多信息是由端到端的四层协议来使用的，为了解除四层处理和路由处理之间的强耦合，故而peer是单独被管理的，也就是说，内核中除了有一棵IPv6的包含了fib，rt cache等节点的路由树之外，还有一棵peer树，用于管理所有的peer项。

 很显然，rt cache和peer在路由处理时，难以避免地要发生关联，让rt cache关联一个peer就再好不过了，这样只需要找到rt cache项，从中取出peer即可，这样就避免了查完rt cache树，再去查peer树。

 听起来不错的样子。我自己也设计过一个在nf conntrack项中保存所有需要查找才能得到的信息，比如路由dst entry，socket，arp项等等。

 然而，在新项频繁产生的场景下，问题发生了！注意，我们的压测脚本模拟了大量的随机IPv6的客户端瞬间同时接入。我来展示一下发生了什么：

 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190606170146924.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RvZzI1MA==,size_16,color_FFFFFF,t_70)  
 哈哈，dst use这个冰山下原来还有这么大一坨东西，由于急于消除rt cache插入Trie这件事，没有细看代码，导致了spin lock将CPU微微跑高一点点。

 和上一篇文章里描述mount对象和mnt_namespace对象的关系一样，杂糅和不合理的耦合是各种问题的温床。

 没关系，干掉它就是了。干掉它的依据有吗？当然有！

  
  * **形而上意义上的缘由**  
     IP协议本来就是一个无连接，无session，尽力而为的协议，你为每一个与之通信的IP保存一个peer，并且还cache到路由项里面，这本身就是污染IP协议的行为！peer在IP层的存在，相当于为IP协议增加了一个session！ 
  * **实际上的现实缘由**  
     实际上，在3.1内核之前，peer在IPv6路由处理中根本就没有起到过作用。搞笑了。  这上面第二点，里面其实是有一个故事的，先看一个patch再说：  
 [https://git.kernel.org/pub/scm/linux/kernel/git/davem/net-next.git/commit/?id=21efcfa0ff27776902a8a15e810147be4d937d69](https://git.kernel.org/pub/scm/linux/kernel/git/davem/net-next.git/commit/?id=21efcfa0ff27776902a8a15e810147be4d937d69)  
 有点意思  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190606170339612.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RvZzI1MA==,size_16,color_FFFFFF,t_70)  
 这是又一起弄巧成拙的事故！

 有意思的是，在3.1内核之前，每一个rt cache绑定一个与cache的源地址相应的peer是IPv6路由代码作者的本意，但是代码可能没有写好，正如上面这个patch说的那样，所有的rt cache项将绑定在同一个健值为0的peer项上，啊哈，这是多么大的一个失误啊！

 赶紧改呗，就有了上面的patch。在代码意义上，这当然很完美了，一个rt cache绑定一个peer，一个peer中保存有该peer的metrics，TCP的socket上会绑定一个dst entry，然后顺着就能取到metrics，简直Perfect！

 但是，代码不是小说用来给人看的，代码是要在CPU上跑的啊！这么一改不要紧，代码是美了，性能却跪了，原因就是上面我那种图里画的，非常明显的一个性能抖降，却没人管。可笑的是，3.1版本之前IPv6路由性能之所以比3.1之后还要好，靠的居然是一个代码上的错误，这太讽刺了！

 值得注意的是，这个inet peer自旋锁问题和前文中rt cache读写锁是两个不同的问题：

  
  * **IPv6 rt cache读写锁问题：2.6.38-rc3引入，David Miller提交** 
  * **IPv6 peer自旋锁问题：3.1.10引入，David Miller提交**  既然3.1.10之前IPv6的peer在路由逻辑中就没有起到过作用，那干掉它有问题吗？显然没有问题！又是一个回退式的修补方案。

 其实，在我的移除rt cache的第一个修复版本里，路由Trie树里保存的仅仅是配置的路由，和配置的metrics，即便是某一个关联二元组的dst entry关联了特定的metrics，由于rt cache不再插入树，这个metrics也没有地方放，所以，我的方案就是，peer是peer，dst entry是dst entry，一码归一码，二者在路由层面不再纠缠。

 peer在IP层真的没有必要，peer是端到端意义上的，你看它的名字和里面的信息就知道，所以说TCP才需要peer，而不是IP。

 如果说TCP层需要peer，那么TCP自己维护peer树就好了嘛。

 好了，回而退之，这时跑同样的压测，再看perf：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190606170523585.png)  
 有点帅，但还不是特别帅，但还是红色，还是两位数，那就继续查！这直接撸代码就好了。原因已经找到：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190606170605758.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RvZzI1MA==,size_16,color_FFFFFF,t_70)  
 根据规范，IPID只要保证四层协议维度的唯一性即可，干嘛搞这么复杂，就取一个协议维度递增的ID，竟然getpeer，竟然不成功还要create，而create就要lock！

 我们看看4.20是怎么做的：

 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190606171112232.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RvZzI1MA==,size_16,color_FFFFFF,t_70)  
 清爽吧。

 好了，改之，为我自己绕开rt cache的路由全部加上NOPEER标志，测之：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190606170641977.png)  
 这里面的关键在于，注意到3.1版本之前，事实上是不会频繁插入peer项的，因为在查找的时候，尚未初始化目标IP地址，所以永远都能找到那个以0为健的peer项目，但是peer的创建和插入是避免不了的，比如上面的函数，在为IPv6数据报文选取IPID的时候，此时目标地址已经就位，此时就会创建真正的peer项并且插入peer树了，插入就会lock。

 加入DST_NOPEER标志之后，便不再执行这个逻辑，再一次解了锁，清爽多了。

 锁是令人遗憾的，是锁而不是数据结构的问题限制了Linux内核在多核心系统高效并行，锁对事情严重性的影响远大于数据结构的影响。

 数据结构是可以持续优化的，当一个线程在一个非常差劲低效率的数据结构里进行搜索时，CPU可以调度另一个承接关系的线程去干别的，然而对于锁而言，CPU只能stall，却显示很忙，事实上这段时间CPU是在瞎忙！

 有没有过面试完了等结果的经历？什么都不想干，什么也干不了，就是忙等，不能安宁。CPU在抢锁时就是这种stall状态！当我们看到CPU利用率很高时，其实这里说的所谓 _**“利用率”**_ 就有问题了，很多时候，这个高值代表的是 _**“CPU stall率”**_ ，够讽刺吧！当我们看到有个spin lock在那spin时，显然，我们知道此时CPU高并没有在干正事儿，但是你知道吗，除了显式的spin lock，CPU访问内存也会让CPU短暂stall，积水成渊，CPU的 _**“利用率”**_ 就高了起来！访存问题导致的CPU stall，需要高效的编程模式，提高cache命中率，但是lock导致的CPU stall，却更加复杂，这需要你对整个逻辑有非常清晰的理解。

 举个栗子🌰，如果你不懂IPv6，但却是个编程狂人，编程神士，你也同样会遭遇路由处理的lock hell，代码写得好，不代表它高效。一般意义上，解锁都要从数据结构开始。

 然而，悲哀的是，只要是容器类的数据结构，几乎都无法做到原子插入和删除操作，树这种复杂的数据结构自然不必说，就连最简单的单向链表，一个插入都需要两个步骤才能完成：

  
  2. 修改前序的next 
  4. 将自己的next指向后继  
      所以这种操作，锁是必须的。  假设我们已经接受了锁，另一种观点就是锁和数据结构效率之间的此消彼长了。

 越简单的数据结构，对于单独的插入操作，指令数越少(比如说单链表，插入只需要两个步骤，也就是两次指针赋值而已)，锁造成的CPU stall时间越短，但是，另一方面，越是简单的数据结构，判断一个元素需不需要插入的时间就越长，比如单链表必须每次遍历，而二叉树却可以二分查，这个方面，锁造成的CPU stall的时间越长，和单独的插入步骤简单数据结构带来的收益相比，人们普遍倾向于以复杂数据结构复杂的单独插入为代价(想象一下插入一个树节点需要的步骤)，换取查找的收益。这无可厚非。

 错误在于， _**锁的滥用！不顾一切的滥用！**_ 只要代码美，纠缠无所谓的滥用！不懂业务逻辑只求代码完美导致的锁的滥用！很多人倾向于把什么锁都改成RCU，然而很多人并不真的懂RCU。

 在这两起由于不关注流量模式而导致的lock hell事故中，锁的滥用责任不可推卸！

 从代码上看，David Miller带来的代码或者其审核通过的代码，绝对是佳作，我们谁不希望用统一的方式处理所有的情况，我们谁不希望可以把所有东西链接在一起，按图索骥获得所需，而不是每次都要查一遍，从代码上看，David Miller带来的东西似乎满足了人们的胃口。

 但是，IPv6特殊的流量模式，导致了内核陷入了锁的地狱，IPv6的地址非常多，并且IPv6没有NAT，所以，地址分布会非常广，这足以让数以万计的锁幽灵带着脚镣跳舞了！

 说实话，要不是IPv4的NAT为各个服务器汇聚了大量的源地址，这个问题早就在IPv4的场景中显露。

 以IPv4和IPv6为维度，对于IPv4，在3.5版本开始了路由处理的净化，而对于IPv6，则在4.2版本开始了同样的过程，晚了一个世代。这其实是和IPv6的部署量以及部署进度紧密相关的。

 Linux内核版本升级的过程，其实就是一个解锁的过程。我一般倾向于用自己的方法先搞一遍，然后去核对一下社区的方案，所以说在解了这个peer锁问题并且出了自己的patch之后，我跟踪了一下上游直到4.20版本(如今5.x都已经放出来了)的内核代码，已经超级干净了：

  
  * **去掉了rt cache(4.2开始就去掉了)，并且为每一条配置的路由加入了percpu的copy** 
  * **解除了rt 和peer之间的关联(没有跟踪commit)，直接浅引用路由查询结果的metrics**  大致嘛，都是这个思路，路子走对了，就不怕fix自己写的bug。

 不过对于我这种不会编程的人而言，一切宗旨就是少改代码，能一行搞定的就不两行，不然就是在写bug…

 
--------
 浙江温州皮鞋👞湿，下雨☔️进水不会胖。

   
  