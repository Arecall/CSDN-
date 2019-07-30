---
title: Linux 3.10内核锁瓶颈描述以及解决-overlayfs的性能缺陷
date: 2019-06-06 23:55:32
tags: CSDN迁移
---
 版权声明：本文为博主原创，无版权，未经博主允许可以随意转载，无需注明出处，随意修改或保持可作为原创！ https://blog.csdn.net/dog250/article/details/90900272   
  又是一个假期，早早早退，然而高速公路离家最近的出口封闭，无奈只能穿越景区，就为了去吃顿饭。饭罢了，该思考和总结了。

 
--------
 大量线程争抢锁导致CPU自旋乃至内核hang住的例子层出不穷。

 我曾经解过很多关于这方面的内核bug：

  
  * nat模块复制tso结构不完全导致SSL握手弹证书慢。 
  * IP路由neighbour系统对pointopoint设备的处理不合理导致争锁。 
  * IPv6路由缓存设计不合理导致争锁。 
  * Overlayfs的mount设计不合理导致争锁。  本文解一例。即描述 _**Overlayfs的mount设计不合理导致争锁**_ 问题以及相应的解法。

 _**后面几篇文章，我会描述其它的几个例子。**_ 

 Linux内核里存在着很多垃圾代码，盲从Linux内核视其无比神圣的人，便违背了规则丧失了独立。Linux内核不是神话，它只是一个可以运行的，功能性能还可以的一个操作系统内核，它给我们多了一种选择，除此之外便不是什么了。

 Linix内核并不完美，相反，没有什么是完美的，Linux内核存在很多缺陷！

 从业至今，很多个节假日前遇到 _**大量线程抢锁**_ 的问题，但是但凡节假日，这种问题总是被我捕获并解锁，然后让我可以过一个比较爽的节假日。

 
--------
 Linux内核解锁案例已经够多了，这种事很多人都知道，但都藏着掖着，觉得教会徒弟饿死师傅，因为一旦掌握了这些，这将是他们求职就业以及晋升的杀手锏。

 但我却不这么想。

 我觉得解决这种事都是些很Low的事情，你怎么能拿Linux内核的缺陷来当自己的资本呢？！

 无论如何，每当我发现Linux内核有这样那样给的缺陷，并且我比较感兴趣，我会怀着比较激动的心情解决之，然后将问题本身和解决方案完全公开。内核都公开了，问题还藏着吗？就算不看我的解决方案，社区十有八九早就已经把问题修复了…

 解锁的过程更多的是提供一种分析解决问题的过程和思路。

 我不是想解决问题，我只是想多玩一会儿而已。

 
--------
 近日，发现有Docker环境在使能overlayfs的情况下，CPU利用率飙高，经过简单排查是mntput函数里面的一个自旋锁占用了大量的CPU，引发问题的应用程序都毫无例外地在执行频繁open/close文件的动作。确切地说，是 _**频繁的close操作**_ 导致了问题。

 为什么？这是为什么？

 为什么在这个行业干了十几年了只有这次碰到了频繁打开关闭文件导致的问题？如果这是一个ext2/ext3/ext4文件系统的固有问题，那么早在200X年问题就该被发现并解决了的，为什么现在都2019年了还存在问题？

 这肯定是新问题！

 事后言，这是overlayfs的问题。但是锅的本源，还是vfs的设计问题，换句话说，这锅，overlayfs不该背。

 overlayfs在文件被close的时候，消耗了大量的CPU事件在一个根本没有必要的大写锁上。说它没有必要，也不是很应该，只是说说罢了。毕竟，事后所言，造成这个性能问题的根源，在于 _**一个优化没有做好。**_

 依然事后所言，你看，一个优化没有覆盖完全，还不如不优化，不优化的话还不会被埋冤。

 
--------
 这一切还要从现象说起。

 Linux内核在操作任何结构体对象的时候，基本都遵循一个风格，即 _**懒惰释放**_ ！不管有关无关，在释放结构对象的时候，只要涉及，Linux内核就会去dec其引用计数。这非常合理，但是Linux内核的vfs子系统却在这些之外，借用了一些trick。看我们看个究竟吧。

 
--------
 无论是什么文件，当一个文件被close的时候，跟踪调用栈，最终会调用 _**mntput函数**_ 。

 让我们看看 _**mntput函数**_ 的逻辑，它最终由 _**mntput_no_expire**_ 表示：

 
```

static void mntput_no_expire(struct mount *mnt)
{
put_again:
    br_read_lock(&vfsmount_lock);

    // 问题出在这里！虽然是likely，但likely只是一厢情愿。
    // 问题是谁能保证每一个文件系统都能遵守规则去维护mnt_ns字段呢？？
    // 比如overlayfs干脆就没有为upper/lower的mount结构设置mnt_ns！
    if (likely(mnt->mnt_ns)) { 
        /* shouldn't be the last one */
        mnt_add_count(mnt, -1);
        br_read_unlock(&vfsmount_lock);
        return;
    }

    br_read_unlock(&vfsmount_lock);

    br_write_lock(&vfsmount_lock);
    mnt_add_count(mnt, -1);

    // 超大概率会从下面的if块返回，因为文件close的mntput要比umount的mntput高频太多
    if (mnt_get_count(mnt)) { 
        br_write_unlock(&vfsmount_lock);
        return;
    }
    ...

	// 下面是真正的最后清理工作，脱链表，释放内存等
	// 只有在真正的mount结构销毁的时候才会到达这里，比如unmount操作
    list_del(&mnt->mnt_instance);
    br_write_unlock(&vfsmount_lock);
    mntfree(mnt);
}


```
 我们知道的事实是，一个文件，关联于其上的是：

  
  * mount对象 
  * mnt_namespace对象 
  * file对象 
  * inode对象 
  * …  其中，和Linux常见的对象之间的关系不同。mount对象和mnt_namespace对象的关系比较微妙，它们是 _**附着**_ 关系，而不是 _**引用**_ 关系。即：

  
  * _**mnt_namespace对象附着在mount对象上，而不是mount对象引用mnt_namespace对象。**_  它们二者分别是独立被维护的。当mount对象初始化需要关联一个mnt_namesace对象时，有一个来附着便是，并不需要获取其引用计数。对应的，只有在umount操作时，才会清空mount对象的mnt_ns这个附着于其上mnt_namespace对象，注意⚠️，是直接赋空：

 
```

void umount_tree(struct mount *mnt, int propagate)
{
    LIST_HEAD(tmp_list);
    struct mount *p;

    for (p = mnt; p; p = next_mnt(p, mnt))
        list_move(&p->mnt_hash, &tmp_list);

    if (propagate)
        propagate_umount(&tmp_list);

    list_for_each_entry(p, &tmp_list, mnt_hash) {
        list_del_init(&p->mnt_expire);
        list_del_init(&p->mnt_list);
        __touch_mnt_namespace(p->mnt_ns);

        // 没有递减引用计数，直接赋值为NULL。
        p->mnt_ns = NULL;

        ...

```
 在 _**mntput_no_expire**_ 中，文件系统非常巧妙但不合理地利用上述mount对象和mnt_namespace对象的这种关系 _**企图用来优化mount的put操作。**_

 优化效果还不错，但是如果并不理解或者某种原因不能利用上述关系的文件系统，将无法获得这个优化所带来的收益，成为优化效果的弃婴。

 overlayfs就是其中一例。overlayfs的mount对象将没有任何mnt_namespace附着于其上。所以操作overlayfs的每一个文件的每一次close操作，最终均会落到 _**br_write_lock**_ 这个大写锁上，造成系统的CPU跑高。

 
--------
 下面是复现和解决的步骤。

 为了离线重现和解决问题，采用下面的步骤。

 首先准备一个overlay文件系统：

 
```
[root@localhost overlay]# mount -t overlay overlay -olowerdir=./lower,upperdir=./upper,workdir=./worker ./merge
[root@localhost overlay]# tree
.
├── lower
├── merge
├── upper
└── worker
    └── work

```
 编写一个超级简单的不断open/close的程序：

 
```
/* loop.c */

#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>

void loop(unsigned char *file)
{
    int fd;

    while (1) {
        fd = open(file, O_RDWR|O_CREAT);
        close(fd);
    }
}

int main(int argc, char **argv)
{
    loop(argv[1]);
    return 0;

}

```
 执行10个进程(太多也行，但我的虚拟机撑不住…)，不断打开关闭overlayfs的文件：

 
```

[root@localhost overlay]# ./a.out merge/test &
[1] 1834
[root@localhost overlay]# ./a.out merge/test &
[2] 1835
[root@localhost overlay]# ./a.out merge/test &
[3] 1836
[root@localhost overlay]# ./a.out merge/test &
[4] 1837
[root@localhost overlay]# ./a.out merge/test &
[5] 1838
[root@localhost overlay]# ./a.out merge/test &
[6] 1839
[root@localhost overlay]# ./a.out merge/test &
[7] 1840
[root@localhost overlay]# ./a.out merge/test &
[8] 1841
[root@localhost overlay]# ./a.out merge/test &
[9] 1842
[root@localhost overlay]# ./a.out merge/test &
[10] 1843


```
 下面是perf热点：

 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190606161052504.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RvZzI1MA==,size_16,color_FFFFFF,t_70)

 systemtap确认在调用sys_close进而到达mntput_no_expire时，mount的mnt_ns字段确实是NULL。问题确认。

 问题是为什么overlayfs的mount 对象会没有mnt_namespace？

 按照上述vfs mount对象和mnt_namespace对象的关系的分析，除非是频度很低的umount操作，否则一个正常的mount对象的mnt_ns字段一定不会为NULL，就一定会进入只有read lock的快速路径。

 那么overlayfs的mount对象便属于少见的不正常的mount对象。

 我猜是作者遗漏了。

 我们看overlayfs的mount的生成过程：

 
```
static int ovl_fill_super(struct super_block *sb, void *data, int silent)
{
    ...
    if (ufs->config.upperdir) {
        ufs->upper_mnt = clone_private_mount(&upperpath);
    ....
    for (i = 0; i < numlower; i++) {
        struct vfsmount *mnt = clone_private_mount(&stack[i]);

```
 根源在 _**clone_private_mount**_ 函数：

 
```
// 看注释，原来是故意的。
/**
 * clone_private_mount - create a private clone of a path
 * This creates a new vfsmount, which will be the clone of @path.  The new will
 * not be attached anywhere in the namespace and will be private (i.e. changes
 * to the originating mount won't be propagated into this).
 * Release with mntput().
 */
struct vfsmount *clone_private_mount(struct path *path)
{
    struct mount *old_mnt = real_mount(path->mnt);
    struct mount *new_mnt;

    if (IS_MNT_UNBINDABLE(old_mnt))
        return ERR_PTR(-EINVAL);

    down_read(&namespace_sem);
    new_mnt = clone_mnt(old_mnt, path->dentry, CL_PRIVATE);
    up_read(&namespace_sem);
    if (IS_ERR(new_mnt))
        return ERR_CAST(new_mnt);

    return &new_mnt->mnt;
}

```
 看注释就够了：

 
> The new will not be attached anywhere in the namespace and will be private (i.e. changes to the originating mount won’t be propagated into this).
> 
>  
 所以从注释的建议看，mount对象从一开始创建就没有打算附着任何mnt_namespace，只是为了对外不可见！也因为如此，无法在mntput_no_expire中被优化。

 是迎合内核函数API的注释要求，还是迎合优化trick？二者可以兼得！

 为overlayfs的mount对象附着一个脱链的dummy mnt_namespace对象不就得了吗？既对外不可见，又可以被优化。

 缺什么，加上便是了。

 基于RH centos 7.2的 _**Linux kernel 3.10**_ 内核的patch如下：

 
```

--- super.c.orig    2019-05-23 20:43:30.071000000 +0800
+++ super.c    2019-05-23 19:58:33.800000000 +0800
@@ -18,7 +18,9 @@
 #include <linux/sched.h>
 #include <linux/statfs.h>
 #include <linux/seq_file.h>
+#include <linux/kallsyms.h>
 #include "overlayfs.h"
+#include "mount.h"

 MODULE_AUTHOR("Miklos Szeredi <miklos@szeredi.hu>");
 MODULE_DESCRIPTION("Overlay filesystem");
@@ -884,6 +886,7 @@
     return ctr;
 }

+struct mnt_namespace *(*sym_create_mnt_ns)(struct vfsmount *mnt);
 static int ovl_fill_super(struct super_block *sb, void *data, int silent)
 {
     struct path upperpath = { NULL, NULL };
@@ -1011,6 +1014,13 @@
             pr_err("overlayfs: failed to clone upperpath\n");
             goto out_put_lowerpath;
         }
+
+        if (sym_create_mnt_ns) {
+            struct mnt_namespace *stub = sym_create_mnt_ns(ufs->upper_mnt);
+            if (stub) {
+                printk("upper stub ok\n");
+            }
+        }

         ufs->workdir = ovl_workdir_create(ufs->upper_mnt, workpath.dentry);
         err = PTR_ERR(ufs->workdir);
@@ -1039,6 +1049,12 @@
          * will fail instead of modifying lower fs.
          */
         mnt->mnt_flags |= MNT_READONLY;
+        if (sym_create_mnt_ns) {
+            struct mnt_namespace *stub = sym_create_mnt_ns(mnt);
+            if (stub) {
+                printk("lower stub ok\n");
+            }
+        }

         ufs->lower_mnt[ufs->numlower] = mnt;
         ufs->numlower++;
@@ -1134,6 +1150,7 @@

 static int __init ovl_init(void)
 {
+    sym_create_mnt_ns = (void *)kallsyms_lookup_name("create_mnt_ns");
     return register_filesystem(&ovl_fs_type);
 }


```
 这个patch里还埋着个大坑。它会内存泄露！

 上面的分析结论明确指出，mount对象和mnt_namespace对象是独立维护的，mount对象并没有拿mnt_namespace对象的引用计数，只是在mount时赋值，在umount时赋NULL即可。

 之所以可以如此鲁莽，是因为内核的vfs子系统将所有的mount对象按照mnt_namespace组织成了N棵树，每一个mnt_namespace对象都会有一棵mount树。所以说，除非是mount根，否则其所有的mnt_namespace均继承mount根的mnt_namespace。

 mount根拿mnt_namespace就够了。mount根释放前，mnt_namespace不会释放。而mount子对象又不会在mount根对象之后释放，所以 _**维护好mount根的前提下，二者有此必有彼！**_

 overlayfs作为一个堆叠组合的文件系统，本身并非内核vfs子系统内建的文件系统类型，因此不应该纳入到这个树形结构体系。

 因此，在这个patch中，我平白无故的给mount对象new了一个mnt_namespace对象，这个mnt_namespace对象并不属于树形体系，因此，既然我分配了它，我就要负责销毁它！

 那么如何销毁以及在哪销毁呢？

 应该在overlayfs的super超级块的ovl_put_super回调函数中进行销毁。将在fill_super里分配的若干mnt_namespace对象给销毁，这才完美！这才不至于内存泄漏！

 
```
static void ovl_put_super(struct super_block *sb)
{
    struct ovl_fs *ufs = sb->s_fs_info;
    unsigned i;
    struct mount *m;

    dput(ufs->workdir);
    m = real_mount(ufs->upper_mnt);
    if (m->mnt_ns)
        put_mnt_ns(m->mnt_ns);
    mntput(ufs->upper_mnt);
    for (i = 0; i < ufs->numlower; i++) {
        m = real_mount(ufs->lower_mnt[i]);
        if (m->mnt_ns)
            put_mnt_ns(m->mnt_ns);
        mntput(ufs->lower_mnt[i]);
    }

    kfree(ufs->config.lowerdir);
    kfree(ufs->config.upperdir);
    kfree(ufs->config.workdir);
    kfree(ufs);
}

```
 
--------
 值得注意的是，高版本内核，即 _**>3.10**_ 版本的内核，它们下意识将rw lock/spinlock改成了rcu lock，但是这并不正确！

 _**将spinlock改成rwlock，将rwlock改成rcu lock，这几乎成了一个范式，但是真正懂这个范式的并不多！**_

 本文描述的问题，根本就不在于用什么锁的问题，根本在于mount对象和mnt_namespace对象的关系以及它们如何结合的问题！

 我们来看最新的，或者说比较新的5.2内核：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190605204803665.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RvZzI1MA==,size_16,color_FFFFFF,t_70)

 最终，当mnt_ns存在的判断失败后，还是会掉入万劫不复的自旋锁的：  
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190605205119724.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RvZzI1MA==,size_16,color_FFFFFF,t_70)

 还是老问题，谁能确保mount对象的mnt_ns就一定会被设置呢？

 此时，你可能需要重新思考Linux内核社区的那些所谓的文人墨客。怼天怼地整天show me the code的放旷之外，其实他们也会犯错。

 在很多人看来他们写出了不正确的代码，然而在社区看来确实根深蒂固而又铁石心肠地认为这是佳作，这无疑证明Linux内核社区根本就是一个熟人社区。

 这里必须要提到的一个人，David Miller，此人崇尚代码的简介可维护可复用，有点过头，以至于他觉得代码整洁之道带来的性能问题都是无关紧要的，together with一群卫道的尚士，也就不多说了。

 David Miller代码整洁之道无可厚非，然则其为此引入的性能bug也是众矢之的！从业的几年中，遭遇了其不下五次的自行引入的bug，以至于…

 以至于每当我排查分析网络性能问题的时候，90%的概率，呃…至少80%的概率吧，按照David Miller的范式风格去排查，基本也就药到病除了…

 不是为了怼而怼，而是他真真的出了差错。而且不止一次。

 关于细节，今天太晚，且听我来日分解，明早演绎。

 
--------
 浙江温州皮鞋湿，下雨进水不会胖。

   
  