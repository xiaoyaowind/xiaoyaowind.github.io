---
layout: post
title: ConcurrentHashMap源码剖析
category: Thread
tags: [like]
keywords: HashMap,java,ConcurrentHashMap
---

## 前言

由于HashMap是线程不安全的，hashmap在多线程下重哈希（resize）方法会导致get的时候死循环,所以在多线程编程下我们不能使用HashMap,
那么多线程并发下怎么友好的解决这个问题呢，Doug Lea大神就友好的写了ConcurrentHashMap这个并发类，那么并发下此类是怎么工作的呢，
那么我就这几个方面了解一下ConcurrentHashMap：
```
(1)ConcurrentHashMap在JDK8里结构
(2)ConcurrentHashMap的put方法、szie方法等
(3)ConcurrentHashMap的扩容
(4)HashMap、Hashtable、ConccurentHashMap三者的区别
(5)ConcurrentHashMap在JDK7和JDK8的区别
```
### 在JDK8中ConcurrentHashMap结构
ConcurrentHashMap底层结构和JDK1.8中的HashMap结构差不多，都是数组+链表+红黑树。当链表节点数超过指定阈值,转换成红黑树，大体结构是一样的。
![ConcurrentHashMap](http://www.xiaoyaowind.com/assets/images/2019/thread/ConcurrentHashMap.jpg)

那么java8中ConcurrentHashMap是如何实现线程安全的呢？
    在jdk1.7中ConcurrentHashMap是采用分段锁Segment来保证的，简单理解，
ConcurrentHashMap是一个Segment数组，Segment通过集成ReentrantLock来进行加锁，每次需要加锁的操作是锁定一个Segment，
这样只要保证每个Segment是线程安全的，则就保证了全局是线程安全的。
    在JDK1.8中ConcurrentHashMap抛弃了Segment，采用了CAS+synchronized来保证安全并发性。具体咱们接着往下走。
 ### 初始化
   首先分析下ConcurrentHashMap初始化
```
    //默认构造函数
    public ConcurrentHashMap() {
       }
        //传入参数为指定初始容量
       public ConcurrentHashMap(int var1) {
            //判断传入参数是否小于0(有的人总喜欢皮一下，次数就是防止皮的)
           if (var1 < 0) {
               throw new IllegalArgumentException();
           } else {
                //由指定的初始容量计算而来，再找最近的2的幂次方。比如传入6，计算公式为8+8/2+1=13，最近的2的幂次方为16，所以sizeCtl就为16。
               int var2 = var1 >= 536870912 ? 1073741824 : tableSizeFor(var1 + (var1 >>> 1) + 1);
               this.sizeCtl = var2;
           }
       }
```
 这个初始化方法是通过提供的初始容量，计算sizeCtl，来控制数组table大小
 
## put方法

```
     public V put(K key, V value) {
         return this.putVal(key, value, false);
     }
```
 put方法调用putval方法，下面看putval方法
```
final V putVal(K key, V value, boolean onlyIfAbsent) {
    //先除去key或value为空的情况
    if (key == null || value == null) throw new NullPointerException();
    
    //计算key的hash值
    int hash = spread(key.hashCode());
    //桶中元素的大小，如果大于等于8，则旋转为红黑树
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {//CAS经典写法，不成功无限重试，让再次进行循环进行相应操作。
        Node<K,V> f; int n, i, fh;
        //如果当前table还没有初始化先调用initTable方法将tab进行初始化
        if (tab == null || (n = tab.length) == 0)
            //初始化数组，后面再详细介绍
            tab = initTable();

        //找该 hash 值对应的数组下标，得到链表第一个节点 f,
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            //如果数组该位置为空
            //用一次 CAS 操作将这个新值放入其中即可，这个 put 操作差不多就结束了，可以拉到最后面了
            //如果 CAS 失败，那就是有并发操作，进到下一个循环就好了
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))//如果f为 null表示链表没有值，则此次放入的key/value存入链表的根 (1)
                break;                   // no lock when adding to empty bin
        }
        // hash 居然可以等于 MOVED，这个需要到后面才能看明白，不过从名字上能猜到，如果在进行扩容先进行扩容
        else if ((fh = f.hash) == MOVED)
            // 帮助数据迁移，这个等到看完数据迁移部分的介绍后，再理解这个就很简单了
            tab = helpTransfer(tab, f);
            
        else { // 到这里是就说明，f 是该位置的头结点，而且不为空
        
            V oldVal = null;
            
            // 获取数组该位置的头结点的监视器锁
            synchronized (f) {//此次添加key/value的链表有元素，则对链表/红黑树进行加锁 这里是java8和java7的不同之处 java8采用的加锁桶，而不是一段桶
                if (tabAt(tab, i) == f) {//出现hash碰撞（情况1:线程1和线程2同时插入在上面(1) 由于是CAS操作只有一个线程会成功，第二个线程会进入到这一步 情况2:普通的hash碰撞）
                    if (fh >= 0) { // 头结点的 hash 值大于 0表示桶是链表 TREEBIN   = -2 桶是红黑树
                        // 用于累加，记录链表的长度
                        binCount = 1;
                        // 遍历链表
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;//链表中的一个元素的key
                            // 如果hash和key都和已存在的元素相等则根据onlyIfAbsebt的值，确定是用之前的值还是新值覆盖，然后也就可以 break 了
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)//如果onlyIfAbsent为fasle，新值覆盖老值
                                    e.val = value;
                                break;//退出,操作完成
                            }
                            // 到了链表的最末端，将这个新值放到链表的最后面，即链表最末尾的值作为新值的前一个元素
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {//如果已经到了末尾值，则创建新的node存放此次插入的key/value
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) { // 红黑树
                        Node<K,V> p;
                        binCount = 2;
                        // 调用红黑树的插值方法插入新节点
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {//如果不等于0判断是否需要旋转为红黑树
               //如果大于8则旋转为红黑树，临界值和 HashMap 一样，也是 8
                if (binCount >= TREEIFY_THRESHOLD)
                    // 这个方法和 HashMap 中稍微有一点点不同，那就是它不是一定会进行红黑树转换，
                    // 如果当前数组的长度小于 64，那么会选择进行数组扩容，而不是转换为红黑树
                    // 具体源码我们就不看了，扩容部分后面说
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    // 
    addCount(1L, binCount);
    return null;
}
```

put 的主流程看完了，但是至少留下了几个问题，第一个是初始化数组，第二个是扩容数组，第三个是帮助数据迁移，这些我们都会在后面进行逐一介绍。
### spread
```
static final int spread(int h) {
        //无符号右移加入高位影响，与HASH_BITS做与操作保留对hash有用的比特位，有让hash>0的意思
        return (h ^ (h >>> 16)) & HASH_BITS;
    }
```
h是原始的hash返回的值,是int类型，而int取值范围:-2147483648到2147483648,前后加起来大概有四十亿的映射空间。只要hash函数映射的比较松散，一般是很难出现碰撞的。
但是考虑到实际的内存的大小，很难放下这么大的数组。

所以为了空间上的考虑,上述中的扰动函数，对原始计算出来的hash值（int类型是4个字节32位）再次处理，右移16位，自己的高半区和低半区做异或，就是为了混合原始hash值的高位和低位，以此来加大低位的随机性。而且混合后的低位参杂了高位的部分特征，
这样高位的信息也被变相的保留下来了。

### 初始化数组：initTable
initTable用于table数组的初始化，注意，table的初始化是没有加锁的，那么如何处理并发呢？
由下面代码可以看到，当要初始化时会通过CAS操作将sizeCtl设为-1，而sizeCtl由volatile修饰，因而保证了修改对后面线程可见。
在这之后如果再有线程执行到此方法时检测到sizeCtl为负数，说明已经有线程正在给扩容了，此时这个线程就会调用Thread.yield()让出一次CPU执行时间。

```
private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;//tab指向数组，sc（sizeCtl）数组初始化/扩容标志位
        while ((tab = table) == null || tab.length == 0) {//数组为null或长度为o的可以进行初始化
            if ((sc = sizeCtl) < 0)//如果有其他线程在初始化/扩容，则本线程进入就绪状态
                Thread.yield(); 
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {//CAS操作，将 sizeCtl 设置为 -1，代表抢到了锁
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;  //默认数组大小为16
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;//table是volatile的
                        sc = n - (n >>> 2);   //扩容阈值为新容量的0.75倍，n=16的话 sc = 16-4 =12 = 16*0.75
                    }
                } finally {
                    sizeCtl = sc;   //扩容保护
                }
                break;
            }
        }
        return tab;
    }

```
### 链表转红黑树: treeifyBin
前面我们在 put 源码分析也说过，treeifyBin 不一定就会进行红黑树转换，也可能是仅仅做数组扩容。具体看源码吧
```
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    if (tab != null) {
        // MIN_TREEIFY_CAPACITY 为 64
        // 如果数组长度小于 64 的时候，其实也就是 32 或者 16 或者更小的时候，会进行数组扩容
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
            // 后面我们再详细分析这个方法
            tryPresize(n << 1);
        // b 是头结点
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            // 加锁
            synchronized (b) {

                if (tabAt(tab, index) == b) {
                    // 下面就是遍历链表，将链表转为红黑树
                    TreeNode<K,V> hd = null, tl = null;
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val,
                                              null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                    // 将红黑树设置到数组相应位置中
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}
```

### tabAt()/casTabAt()/setTabAt()
ABASE表示table中首个元素的内存偏移地址，所以(long)i << ASHIFT) + ABASE得到table[i]的内存偏移地址：

```
    static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }

    static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
    }

    static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
        U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
    }
```
对i位置结点的写操作有两个方法，casTabAt()与setTabAt()。源码中有这样一段注释：

```
 * Note that calls to setTabAt always occur within locked regions,
     * and so in principle require only release ordering, not
     * full volatile semantics, but are currently coded as volatile
     * writes to be conservative.
```
所以要原子语义的写操作需要使用casTabAt()，setTabAt()是在锁定桶的状态下才会被调用，之所以实现成这样只是带保守性的一种写法而已。

### 扩容实现：tryPresize
扩容也就是做翻倍扩容的，扩容后数组容量为原来的 2 倍。
```
// 需要说明的是，方法参数 size 传进来时已经变为原来的2倍了，tryPresize(n << 1),先参数变为2倍，再传
private final void tryPresize(int size) {
    //如果size>=0.5倍最大容量，则取最大容量
    int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
    // 否则c：size 的 1.5 倍，再加 1，再往上取最近的 2 的 n 次方。
        tableSizeFor(size + (size >>> 1) + 1);
    int sc;
    while ((sc = sizeCtl) >= 0) {
        Node<K,V>[] tab = table; int n;

        // 这个 if 分支和之前说的初始化数组的代码基本上是一样的，在这里，我们可以不用管这块代码
        if (tab == null || (n = tab.length) == 0) {
            n = (sc > c) ? sc : c;
            if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if (table == tab) {
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = nt;
                        sc = n - (n >>> 2); // 0.75 * n
                    }
                } finally {
                    sizeCtl = sc;
                }
            }
        }
        else if (c <= sc || n >= MAXIMUM_CAPACITY)
            break;
        else if (tab == table) {
            //返回值(rs) 高16位置0，第16位为1，低15位存放当前容量n，用于表示是对n的扩容，具体后面分析
            int rs = resizeStamp(n);

            if (sc < 0) {
                Node<K,V>[] nt;
                 //条件1：检查是对容量n的扩容，保证sizeCtl与n是一块修改好的
                 //条件2与条件3：是进行sc的最小值或最大值判断。
                 //条件4与条件5: 确保tranfer()中的nextTable相关初始化逻辑已走完。
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
              
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))//有新线程参与则sizeCtl加1，然后执行 transfer 方法
                    transfer(tab, nt);
            }
           没有线程在进行扩容，将sizeCtl的值改为(rs << RESIZE_STAMP_SHIFT) + 2)，原因见下面sizeCtl值的计算分析。
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
        }
    }
}
```
这个方法的核心在于 sizeCtl 值的操作，首先将其设置为一个负数，然后执行 transfer(tab, null)，再下一个循环将 sizeCtl 加 1，并执行 transfer(tab, nt)，之后可能是继续 sizeCtl 加 1，并执行 transfer(tab, nt)。

所以，可能的操作就是执行 1 次 transfer(tab, null) + 多次 transfer(tab, nt)，这里怎么结束循环的需要看完 transfer 源码才清楚。
### resizeStamp()
在上面的代码中有调用到这样的一个方法。
```
/**
 * The number of bits used for generation stamp in sizeCtl.
 * Must be at least 6 for 32bit arrays.
*/
private static int RESIZE_STAMP_BITS = 16;

/**
 * The bit shift for recording size stamp in sizeCtl.
 */
private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;

static final int resizeStamp(int n) {
        return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}
```
Integer.numberOfLeadingZeros(n)用于计算n转换成二进制后前面有几个0。这个有什么作用呢？
首先ConcurrentHashMap的容量必定是2的幂次方，所以不同的容量n前面0的个数必然不同，这样可以保证是在原容量为n的情况下进行扩容。
(1 << (RESIZE_STAMP_BITS - 1)即是1<<15，表示为二进制即是高16位为0，低16位为1(int4字节32位)：
```
1| 0000 0000 0000 0000 1000 0000 0000 0000
```
所以resizeStamp()的返回值(rs) 高16位置0，第16位为1，低15位存放当前容量n，用于表示是对n的扩容。
rs与RESIZE_STAMP_SHIFT配合可以求出新的sizeCtl的值，分情况如下：

sc < 0
已经有线程在扩容，将sizeCtl+1并调用transfer()让当前线程参与扩容。
sc >= 0
表示没有线程在扩容，使用CAS将sizeCtl的值改为(rs << RESIZE_STAMP_SHIFT) + 2)。
rs即resizeStamp(n)，记temp=rs << RESIZE_STAMP_SHIFT。如当前容量为8时rs的值：

```
//rs
0000 0000 0000 0000 1000 0000 0000 1000
//temp = rs << RESIZE_STAMP_SHIFT，即 temp = rs << 16，左移16后temp最高位为1，所以temp成了一个负数。
1000 0000 0000 1000 0000 0000 0000 0000
//sc = (rs << RESIZE_STAMP_SHIFT) + 2)
1000 0000 0000 1000 0000 0000 0000 0010
```
那么在扩容时sizeCtl值的意义便如下图所示：
高15位 | 低16位 |
容量n  |并行扩容线程数+1 |



### 数据迁移：transfer
jdk1.8版本的ConcurrentHashMap支持并发扩容，上面已经分析了一小部分，下面这个方法是真正进行并行扩容的地方。
这个方法有点长，将原来的 tab 数组的元素迁移到新的 nextTab 数组中。

虽然我们之前说的 tryPresize 方法中多次调用 transfer 不涉及多线程，但是这个 transfer 方法可以在其他地方被调用，典型地，我们之前在说 put 方法的时候就说过了，请往上看 put 方法，是不是有个地方调用了 helpTransfer 方法，helpTransfer 方法会调用 transfer 方法的。
此方法支持多线程执行，外围调用此方法的时候，会保证第一个发起数据迁移的线程，nextTab 参数为 null，之后再调用此方法的时候，nextTab 不会为 null。
阅读源码之前，先要理解并发操作的机制。原数组长度为 n，所以我们有 n 个迁移任务，让每个线程每次负责一个小任务是最简单的，每做完一个任务再检测是否有其他没做完的任务，帮助迁移就可以了，而 Doug Lea 使用了一个 stride，简单理解就是步长，每个线程每次负责迁移其中的一部分，如每次迁移 16 个小任务。所以，我们就需要一个全局的调度者来安排哪个线程执行哪几个任务，这个就是属性 transferIndex 的作用。
第一个发起数据迁移的线程会将 transferIndex 指向原数组最后的位置，然后从后往前的 stride 个任务属于第一个线程，然后将 transferIndex 指向新的位置，再往前的 stride 个任务属于第二个线程，依此类推。当然，这里说的第二个线程不是真的一定指代了第二个线程，也可以是同一个线程，这个读者应该能理解吧。其实就是将一个大的迁移任务分为了一个个任务包。
```
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;

    // stride 在单核下直接等于 n，多核模式下为 (n>>>3)/NCPU，最小值是 16
    // stride 可以理解为”步长“，有 n 个位置是需要进行迁移的，
    //   将这 n 个任务分为多个任务包，每个任务包有 stride 个任务
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range

    // 如果 nextTab 为 null，先进行一次初始化
    //    前面我们说了，外围会保证第一个发起迁移的线程调用此方法时，参数 nextTab 为 null
    //       之后参与迁移的线程调用此方法时，nextTab 不会为 null
    if (nextTab == null) {
        try {
            // 容量翻倍
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        // nextTable 是 ConcurrentHashMap 中的属性
        nextTable = nextTab;
        // transferIndex 也是 ConcurrentHashMap 的属性，用于控制迁移的位置
        transferIndex = n;
    }

    int nextn = nextTab.length;

    // ForwardingNode 翻译过来就是正在被迁移的 Node
    // 这个构造方法会生成一个Node，key、value 和 next 都为 null，关键是 hash 为 MOVED
    // 后面我们会看到，原数组中位置 i 处的节点完成迁移工作后，
    //    就会将位置 i 处设置为这个 ForwardingNode，用来告诉其他线程该位置已经处理过了
    //    所以它其实相当于是一个标志。
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);


    // advance 指的是做完了一个位置的迁移工作，可以准备做下一个位置的了
    boolean advance = true;
    boolean finishing = false; // to ensure sweep before committing nextTab

    /*
     * 下面这个 for 循环，最难理解的在前面，而要看懂它们，应该先看懂后面的，然后再倒回来看
     * 
     */

    // i 是位置索引，bound 是边界，注意是从后往前
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;

        // 下面这个 while 真的是不好理解
        // advance 为 true 表示可以进行下一个位置的迁移了
        //   简单理解结局：i 指向了 transferIndex，bound 指向了 transferIndex-stride
        while (advance) {
            int nextIndex, nextBound;
            if (--i >= bound || finishing)
                advance = false;

            // 将 transferIndex 值赋给 nextIndex
            // 这里 transferIndex 一旦小于等于 0，说明原数组的所有位置都有相应的线程去处理了
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                // 看括号中的代码，nextBound 是这次迁移任务的边界，注意，是从后往前
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) {
                // 所有的迁移操作已经完成
                nextTable = null;
                // 将新的 nextTab 赋值给 table 属性，完成迁移
                table = nextTab;
                // 重新计算 sizeCtl：n 是原数组长度，所以 sizeCtl 得出的值将是新数组长度的 0.75 倍
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }

            // 之前我们说过，sizeCtl 在迁移前会设置为 (rs << RESIZE_STAMP_SHIFT) + 2
            // 然后，每有一个线程参与迁移就会将 sizeCtl 加 1，
            // 这里使用 CAS 操作对 sizeCtl 进行减 1，代表做完了属于自己的任务
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                // 任务结束，方法退出
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;

                // 到这里，说明 (sc - 2) == resizeStamp(n) << RESIZE_STAMP_SHIFT，
                // 也就是说，所有的迁移任务都做完了，也就会进入到上面的 if(finishing){} 分支了
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        // 如果位置 i 处是空的，没有任何节点，那么放入刚刚初始化的 ForwardingNode ”空节点“
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        // 该位置处是一个 ForwardingNode，代表该位置已经迁移过了
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        else {
            // 对数组该位置处的结点加锁，开始处理数组该位置处的迁移工作
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    // 头结点的 hash 大于 0，说明是链表的 Node 节点
                    if (fh >= 0) {
                        // 下面这一块和 Java7 中的 ConcurrentHashMap 迁移是差不多的，
                        // 需要将链表一分为二，
                        //   找到原链表中的 lastRun，然后 lastRun 及其之后的节点是一起进行迁移的
                        //   lastRun 之前的节点需要进行克隆，然后分到两个链表中
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        // 其中的一个链表放在新数组的位置 i
                        setTabAt(nextTab, i, ln);
                        // 另一个链表放在新数组的位置 i+n
                        setTabAt(nextTab, i + n, hn);
                        // 将原数组该位置处设置为 fwd，代表该位置已经处理完毕，
                        //    其他线程一旦看到该位置的 hash 值为 MOVED，就不会进行迁移了
                        setTabAt(tab, i, fwd);
                        // advance 设置为 true，代表该位置已经迁移完毕
                        advance = true;
                    }
                    else if (f instanceof TreeBin) {
                        // 红黑树的迁移
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        // 如果一分为二后，节点数少于 8，那么将红黑树转换回链表
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;

                        // 将 ln 放置在新数组的位置 i
                        setTabAt(nextTab, i, ln);
                        // 将 hn 放置在新数组的位置 i+n
                        setTabAt(nextTab, i + n, hn);
                        // 将原数组该位置处设置为 fwd，代表该位置已经处理完毕，
                        //    其他线程一旦看到该位置的 hash 值为 MOVED，就不会进行迁移了
                        setTabAt(tab, i, fwd);
                        // advance 设置为 true，代表该位置已经迁移完毕
                        advance = true;
                    }
                }
            }
        }
    }
}
```
说到底，transfer 这个方法并没有实现所有的迁移任务，每次调用这个方法只实现了 transferIndex 往前 stride 个位置的迁移工作，其他的需要由外围来控制。

这个时候，再回去仔细看 tryPresize 方法可能就会更加清晰一些了。

### get 过程分析
get 方法从来都是最简单的，这里也不例外：

1、计算 hash 值 
2、根据 hash 值找到数组对应位置: (n - 1) & h 
3、根据该位置处结点性质进行相应查找

如果该位置为 null，那么直接返回 null 就可以了
如果该位置处的节点刚好就是我们需要的，返回该节点的值即可
如果该位置节点的 hash 值小于 0，说明正在扩容，或者是红黑树此时根据 find 方法即可
如果以上 3 条都不满足，那就是链表，进行遍历比对即可

```
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        // 判断头结点是否就是我们需要的节点
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        // 如果头结点的 hash 小于 0，说明 正在扩容，或者该位置是红黑树
        else if (eh < 0)
            // 参考 ForwardingNode.find(int h, Object k) 和 TreeBin.find(int h, Object k)
            return (p = e.find(h, key)) != null ? p.val : null;

        // 遍历链表
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```
简单说一句，此方法的大部分内容都很简单，只有正好碰到扩容的情况，ForwardingNode.find(int h, Object k) 稍微复杂一些，不过在了解了数据迁移的过程后，这个也就不难了，所以限于篇幅这里也不展开说了。