---
title: SACK Panic的POC演示与修复建议
date: 2019-07-10 23:06:04
tags: CSDN迁移
---
  如果谁有旋转升降座椅爆炸的短视频，麻烦提供给我。

 
--------
 按照昨天文章里写的那个思路：  
 [https://blog.csdn.net/dog250/article/details/95252740](https://blog.csdn.net/dog250/article/details/95252740)  
 刚刚复现并录制了一个SACK Panic的POC动画，我用两台直连虚拟机测试的结果：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190710220224389.gif)

 复现率超过90%，每次重试次数几乎不超过3次即可使机器panic。

 然而在实际网络环境中测试，复现率必然会降低。

 和一位仁兄讨论的结果，所见略同，大概的问题基本就是一对矛盾：

  
  * _**长肥管道和丢包**_  若想憋准大于64KB段数的的skb，保证17*32768字节的长度在发送缓冲区未被确认，就要BDP很大，即长肥管道，但是长肥管道丢包代价太大且难恢复。我在虚拟机网络，内网以及家庭路由器网络测试的复现率大致分别是90%，50%，30%，这些都是 _**好网络**_ ，可想而知，公网上很难触发这个漏洞。

 公网上很多限速，清洗，丢包率无法保证，并且你也很难算出来。此外，即便你营造了一个长肥管道，如果服务器端没有做cork，skb还是容易细水长流走，也就是说你很难积累判断发送端是否积累了足够的未确认数据，如果pacing rate控制不好，细水长流走是难免的。这个可以通过采用burst积累ack的方式模拟发送端的cork，但是时序还是很难控制。

 也许是我这个POC不好吧，如果谁有更好的，欢迎一起探讨。不过我还是只提供思路，详见：[https://blog.csdn.net/dog250/article/details/95252740](https://blog.csdn.net/dog250/article/details/95252740)  
 另一个POC等过段时间再拿出来。

 不过，我觉得以上这些技术困难都不是问题，本来这种DDoS漏洞就不是用来精准狙击的，既然近距离有中枪的概率，那意味着互联网上扫一波，总有机器中招，有点类似古代攻城战前盲射的那一波弓箭。至少起到 _**让你害怕**_ 的作用就足够了，这非常像街头的那些铁链子纹身花衬衫光头的小混混，就是吓唬作用，大的他们也搞不来，反而那些温文尔雅的长袍礼帽的喝茶大叔更可怕一些。

 
--------
 下面该解药了。

 Netflix&Redhat官方以及社区给出的方案都是 _**限制合并frags的总大小不要超过65535**_ 。参见下面的patch：  
 [https://github.com/Netflix/security-bulletins/blob/master/advisories/third-party/2019-001/PATCH_net_1_4.patch](https://github.com/Netflix/security-bulletins/blob/master/advisories/third-party/2019-001/PATCH_net_1_4.patch)  
 其核心是增加了这么一个限定函数：

 
```
+int tcp_skb_shift(struct sk_buff *to, struct sk_buff *from,
+		  int pcount, int shiftlen)
+{
+	/* TCP min gso_size is 8 bytes (TCP_MIN_GSO_SIZE)
+	 * Since TCP_SKB_CB(skb)->tcp_gso_segs is 16 bits, we need
+	 * to make sure not storing more than 65535 * 8 bytes per skb,
+	 * even if current MSS is bigger.
+	 */
+	if (unlikely(to->len + shiftlen >= 65535 * TCP_MIN_GSO_SIZE))
+		return 0;
+	if (unlikely(tcp_skb_pcount(to) + pcount > 65535))
+		return 0;
+	return skb_shift(to, from, shiftlen);
+}

```
 但是在我看来，这是不对的！

 这种解法根本就没有踩到问题的根源！

 _**由于总量超过65535了，所以溢出，那么不让总量超过65535不就行了吗？**_

 无可厚非，能解决问题，算紧急补救措施吧，是个人第一个想到的都是这个方案，没啥问题。

 但是，若想彻底解决这个问题，不妨刨下根源。

 内核有两个限制性的设定：

  
  2. 一个frag在x86至多是32KB字节的大小。 
  4. 一个skb至多有17个frags。  现在让我们简单算一下17个32768字节的数据总量一共是多少，很简单，17×3276817\times 3276817×32768字节。而gso_segs字段的类型是u16，即最多65535个segs。

 问题是， _**当初为什么人们认为gso的segs最多65535就够了？**_ 我想用u16表示segs字段，他们是为了节省内存吧。按照常规的mss，65535确实够了，但是精算起来，如果raw data的mss为8的话，u16就要溢出了，而我们看下，u16，mss为8，17个段，最大表示65535个segs，这四者之间确实有点微妙的关系：

  
  * 32768×2=6553632768\times 2=6553632768×2=65536 
  * 172=8+1\dfrac{17}{2}=8+1217​=8+1  如果想到mss可能是8的话，就差那么一点点就不溢出了！是的，就差一点！8已经是最小的raw data的mss了，当然，如果有新的TCP option可能会更小，但现如今，常规情况下，能凑出的比较小的raw data mss就是8了。这样算来，17×327688=65536+327688\dfrac{17\times 32768}{8}=65536+\dfrac{32768}{8}817×32768​=65536+832768​，就差这么一点点。选择gso_segs用u16真的比较合适，但是溢出了327688\dfrac{32768}{8}832768​的数据，就是错了。

 但是现实中，mss会为8吗？很少，但是理论上确实会存在！

 所以问题的根源在于 _**gso_segs字段选型错误！它应该用u32来表示，而不是u16！**_

 内核规定每一个frag最多32KB没有错，内核每一个skb最多携带17个非线性frags也没有错，这些是体系结构相关的约定俗成，mss可以是48字节且携带40字节options更是没有错，这是网络的约定俗成，是gso_segs没有适配它们，而不是反过来啊！

 一个是系统的约定，一个是编程的失误，为什么程序员们总是修改代码去适配约定而不喜欢修改数据类型呢？！

 嗯？修改数据类型的话，每一个skb多了2个字节！是的，多了两个字节！那么100万个包就多了2M个字节，关键是内存贱啊，2M怕什么！为什么程序员总是喜欢改代码呢？改一个数据类型不就好了吗？明显就是gso_segs定义的有问题啊。

 噢，对了，紧急修复必须是kpatch，而kpatch不支持结构体数据类型的修改。嗯，很不错的自我安慰…可是我在最新版的mainline上看到的gso_segs依然是u16啊！

 如果程序员(特别是那么David Miller，没啥了不起的)如此执着，就是不改数据类型，那么也不要增加if逻辑啊，代码多了不是好事。既然gso_segs字段溢出了，那就让tcp_shifted_skb的pcount参数也溢出呗，将它定义成u16而不是int，问题就解决了。我们知道，计算机里的数字，就是一个环，大家都同样回绕，相对大小关系不会发生任何改变。

 所以，我的patch建议，要么如下：

 
```
- unsigned short gso_segs;
+ unsigned int gso_segs;

```
 要么就这样：

 
```
static bool tcp_shifted_skb(struct sock *sk, struct sk_buff *skb,
                struct tcp_sacktag_state *state,
-               unsigned int pcount, int shifted, int mss,
+               unsigned short pcount, int shifted, int mss,
                bool dup_sack)

```
 而不是去添加什么限制代码，那样只会越来越乱！别忘了糟糕的RFC5961。

 
--------
 浙江温州皮鞋湿，下雨进水不会胖！

   
  