---
layout: post
title: JAVA类集框架之HashMap源码剖析
category: thread
tags: [like]
keywords: HashMap,java
---
HashMap是Java程序员使用频次最高的用于映射(键值对)处理的数据模型，最近学习了HashMap的底层实现，
本文是自己对HashMap的底层结构实现和功能原理理解，如有不足，望指正。


简介
  <p style="text-indent:2em">Java为数据结构映射中的定义了一个接口java.util.Map,此接口有四个常用实现类，分别是HashMap,HashTable,LinkedHashMap和treeMap,类继承关系如下：</p>
  
![java-collection-relationship](http://www.xiaoyaowind.com/assets/images/2019/thread/java-collection-relationship.jpg)

## 四个实现类说明:

  <p style="text-indent:2em">(1) HashMap:根据键的hashcode的值存储数据，大多数情况下能直接定位到它的值，因而具有很快的访问速度，但是遍历顺序却是不确定的，HashMap最多只允许一条记录的键为空，但允许多条记录的值为空。
HashMap为非线程安全的，即任意时刻可以有多个线程同时写HashMap,可能会导致线程数据不一致。如果要线程安全，可以用Collections的synchronizedMap方法使HashMap具有线程安全的能力。
或者使用ConcurrentHashMap。</p>

  <p style="text-indent:2em">(2) Hashtable：Hashtable是遗留类，很多映射的常用功能与HashMap类似，不同的是它承自Dictionary类，并且是线程安全的，任一时间只有一个线程能写Hashtable，并发性不如ConcurrentHashMap，
因为ConcurrentHashMap引入了分段锁。Hashtable不建议在新代码中使用，不需要线程安全的场合可以用HashMap替换，需要线程安全的场合可以用ConcurrentHashMap替换。</p>

  <p style="text-indent:2em">(3) LinkedHashMap：LinkedHashMap是HashMap的一个子类，保存了记录的插入顺序，
在用Iterator遍历LinkedHashMap时，先得到的记录肯定是先插入的，也可以在构造时带参数，按照访问次序排序。</p>

  <p style="text-indent:2em">(4) TreeMap：TreeMap实现SortedMap接口，能够把它保存的记录根据键排序，默认是按键值的升序排序，也可以指定排序的比较器，当用Iterator遍历TreeMap时，
得到的记录是排过序的。如果使用排序的映射，建议使用TreeMap。在使用TreeMap时，key必须实现Comparable接口或者在构造TreeMap传入自定义的Comparator，
否则会在运行时抛出java.lang.ClassCastException类型的异常。</p>

  <p style="text-indent:2em">通过上面的比较，我们知道了HashMap是Java的Map家族中一个普通成员，鉴于它可以满足大多数场景的使用条件，所以是使用频度最高的一个。
下文我们主要结合源码，从存储结构、常用方法分析、扩容以及安全性等方面深入讲解HashMap的工作原理。</p>

  <p style="text-indent:2em">在JDK1.6，JDK1.7中，HashMap采用位桶+链表实现，即使用链表处理冲突，同一hash值的链表都存储在一个链表里。但是当位于一个桶中的元素较多，即hash值相等的元素较多时，通过key值依次查找的效率较低。而JDK1.8中，HashMap采用位桶+链表+红黑树实现，当链表长度超过阈值（8）时，将链表转换为红黑树，这样大大减少了查找时间。</p>

#### 内部实现

   <p style="text-indent:2em">搞清楚HashMap，首先需要知道HashMap是什么，即它的存储结构-字段；其次弄明白它能干什么，即它的功能实现-方法。下面我们针对这两个方面详细展开讲解。</p>

## 存储结构-字段

   <p style="text-indent:2em">从结构实现来讲，HashMap是数组+链表+红黑树（JDK1.8增加了红黑树部分）实现的，如下如所示。</p>
   
![HashMap存储结构](http://www.xiaoyaowind.com/assets/images/2019/thread/HashMap存储结构.jpg)

#### hashMap的属性：

<p style="text-indent:2em">上面简单的说了一下其结构，可能大家还不是很理解，下面从源码开始看，就应该很容易去理解。</p>

```
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {
    /*序列号，序列化时候使用*/
    private static final long serialVersionUID = 362498820763181265L;
    /**默认容量，1向左移位4个，00000001变成00010000，也就是2的4次方为16，
    使用移位是因为移位是计算机基础运算，效率比加减乘除快。**/
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
    //最大容量，2的30次方。
    static final int MAXIMUM_CAPACITY = 1 << 30;
    //加载因子，用于扩容使用。
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    //当某个桶节点数量大于8时，会转换为红黑树。
    static final int TREEIFY_THRESHOLD = 8;
    //当某个桶节点数量小于6时，会转换为链表，前提是它当前是红黑树结构。
    static final int UNTREEIFY_THRESHOLD = 6;
    //当满足桶节点数量>8且整个hashMap中元素数量大于64时，也会进行转为红黑树结构。
    static final int MIN_TREEIFY_CAPACITY = 64;
    //存储元素的数组，transient关键字表示该属性不能被序列化
    transient Node<K,V>[] table;
    //将数据转换成set的另一种存储形式，这个变量主要用于迭代功能。
    transient Set<Map.Entry<K,V>> entrySet;
    //元素数量
    transient int size;
    //统计该map修改的次数
    transient int modCount;
    //临界值，也就是元素数量达到临界值时，会进行扩容。
    int threshold;
    //也是加载因子，只不过这个是变量。
    final float loadFactor; 
```
    
上面是hashMap的属性，下面再说说它里面的内部类，并不是所有的内部类，只说常用的。
  
```  
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        TreeNode<K,V> parent;  
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    
        boolean red;
        TreeNode(int hash, K key, V val, Node<K,V> next) {
            super(hash, key, val, next);
        }
}
```
使用静态内部类，是为了方便调用，而不用每次调用里面的属性或者方法都需要new一个对象。这是一个红黑树的结构。(不了解红黑树的可以去了解一下)

```
 static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;
 
        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
```
里面还包含了一个结点内部类，是一个单向链表。上面这两个内部类再加上之前的Node<K,V>[] table属性，组成了hashMap的结构，哈希桶。
### 构造方法：
大致懂了hashMap的结构，我们来看看构造方法，一共有3个。
```  
    public HashMap() {
         this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
     }
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }
```
第一个，空参构造，使用默认的加载因子0.75；第二个，设置初始容量，并使用默认的加载因子；第三个，设置初始容量和加载因子，第二个构造方法其实也是调用了第三个构造函数。下面，我们再来看看最后一个构造函数。
```
 public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }
    
 final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    //获取该map的实际长度
    int s = m.size();
    if (s > 0) {
        //判断table是否初始化，如果没有初始化
        if (table == null) { // pre-size
            /**求出需要的容量，因为实际使用的长度=容量*0.75得来的，+1是因为小数相除，基本都不会是整数，容量大小不能为小数的，后面转换为int，多余的小数就要被丢掉，所以+1，例如，map实际长度22，22/0.75=29.3,所需要的容量肯定为30，有人会问如果刚刚好除得整数呢，除得整数的话，容量大小多1也没什么影响**/
            float ft = ((float)s / loadFactor) + 1.0F;
            //判断该容量大小是否超出上限。
            int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                     (int)ft : MAXIMUM_CAPACITY);
            /**对临界值进行初始化，tableSizeFor(t)这个方法会返回大于t值的，且离其最近的2次幂，例如t为29，则返回的值是32**/
            if (t > threshold)
                threshold = tableSizeFor(t);
        }
        //如果table已经初始化，则进行扩容操作，resize()就是扩容。
        else if (s > threshold)
            resize();
            
        //遍历，把map中的数据转到hashMap中。
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
                K key = e.getKey();
                V value = e.getValue();
                putVal(hash(key), key, value, false, evict);
        }
    }
 }
```

该构造函数，传入一个Map，然后把该Map转为hashMap，resize方法在下面添加元素的时候会详细讲解，在上面中entrySet方法会返回一个Set<Map.Entry<K, V>>，泛型为Map的内部类Entry，它是一个存放key-value的实例，也就是Map中的每一个key-value就是一个Entry实例，为什么使用这个方式进行遍历，因为效率高，具体自己百度一波，putVal方法把取出来的每个key-value存入到hashMap中，待会会仔细讲解。

构造函数和属性讲得差不多了，下面要讲解的是增删改查的操作以及常用的、重要的方法，毕竟里面的方法太多了，其它的就自己去看看吧。

添加元素：
    在讲解put方法之前，先看看hash方法，看怎么计算哈希值的。
```
   static final int hash(Object key) {
        int h;
        /**先获取到key的hashCode，然后进行移位再进行异或运算，为什么这么复杂，不用想肯定是为了减少hash冲突**/
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
   }
```
接着再来看看put方法。
```
 public V put(K key, V value) {
        /**四个参数，第一个hash值，第四个参数表示如果该key存在值，如果为null的话，则插入新的value，最后一个参数，在hashMap中没有用，可以不用管，使用默认的即可**/
        return putVal(hash(key), key, value, false, true);
    }
 
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        //tab 哈希数组，p 该哈希桶的首节点，n hashMap的长度，i 计算出的数组下标
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        ①//获取长度并进行扩容，使用的是懒加载，table一开始是没有加载的，等put后才开始加载
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        ②/**如果计算出的该哈希桶的位置没有值，则把新插入的key-value放到此处，此处就算没有插入成功，也就是发生哈希冲突时也会把哈希桶的首节点赋予p**/
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        //发生哈希冲突的几种情况
        else {
            // e 临时节点的作用， k 存放该当前节点的key 
            Node<K,V> e; K k;
            ③//第一种，插入的key-value的hash值，key都与当前节点的相等，e = p，则表示为首节点
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            ④//第二种，hash值不等于首节点，判断该p是否属于红黑树的节点
            else if (p instanceof TreeNode)
                /**为红黑树的节点，则在红黑树中进行添加，如果该节点已经存在，则返回该节点（不为null），该值很重要，用来判断put操作是否成功，如果添加成功返回null**/
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
           ⑤ //第三种，hash值不等于首节点，不为红黑树的节点，则为链表的节点
            else {
                //遍历该链表
                for (int binCount = 0; ; ++binCount) {
                    //如果找到尾部，则表明添加的key-value没有重复，在尾部进行添加
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        //判断是否要转换为红黑树结构
                        if (binCount >= TREEIFY_THRESHOLD - 1) 
                            treeifyBin(tab, hash);
                        break;
                    }
                    //如果链表中有重复的key，e则为当前重复的节点，结束循环
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            //有重复的key，则用待插入值进行覆盖，返回旧值。
            if (e != null) { 
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        //到了此步骤，则表明待插入的key-value是没有key的重复，因为插入成功e节点的值为null
        //修改次数+1
        ++modCount;
        ⑥//实际长度+1，判断是否大于临界值，大于则扩容
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        //添加成功
        return null;
    }
```
上面就是具体的元素添加,可以整理成下图
![HashMap的put方法图](http://www.xiaoyaowind.com/assets/images/2019/thread/HashMap-put.png)

①.判断键值对数组table[i]是否为空或为null，否则执行resize()进行扩容；

②.根据键值key计算hash值得到插入的数组索引i，如果table[i]==null，直接新建节点添加，转向⑥，如果table[i]不为空，转向③；

③.判断table[i]的首个元素是否和key一样，如果相同直接覆盖value，否则转向④，这里的相同指的是hashCode以及equals；

④.判断table[i] 是否为treeNode，即table[i] 是否是红黑树，如果是红黑树，则直接在树中插入键值对，否则转向⑤；

⑤.遍历table[i]，判断链表长度是否大于8，大于8的话把链表转换为红黑树，在红黑树中执行插入操作，否则进行链表的插入操作；遍历过程中若发现key已经存在直接覆盖value即可；

⑥.插入成功后，判断实际存在的键值对数量size是否超多了最大容量threshold，如果超过，进行扩容。

 ### 扩容机制
 扩容(resize)就是重新计算容量，向HashMap对象里不停的添加元素，而HashMap对象内部的数组无法装载更多的元素时，对象就需要扩大数组的长度，以便能装入更多的元素。当然Java里的数组是无法自动扩容的，方法是使用一个新的数组代替已有的容量小的数组，新的数组是已有数组的2倍。就像我们用一个小桶装水，如果想装更多的水，就得换大水桶。
在元素添加里面涉及到扩容，我们来看看扩容方法resize。

```
final Node<K,V>[] resize() {
        //把没插入之前的哈希数组赋值为oldTal
        Node<K,V>[] oldTab = table;
        //old的长度
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        //old的临界值
        int oldThr = threshold;
        //初始化new的长度和临界值
        int newCap, newThr = 0;
        //oldCap > 0也就是说不是首次初始化，因为hashMap用的是懒加载
        if (oldCap > 0) {
            //大于最大值
            if (oldCap >= MAXIMUM_CAPACITY) {
                //临界值为整数的最大值
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            //标记##，其它情况，扩容两倍，并且扩容后的长度要小于最大值，old长度也要大于16
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                //临界值也扩容为old的临界值2倍
                newThr = oldThr << 1; 
        }
        /**如果oldCap<0，但是已经初始化了，像把元素删除完之后的情况，那么它的临界值肯定还存在，        
           如果是首次初始化，它的临界值则为0
        **/
        else if (oldThr > 0) 
            newCap = oldThr;
        //首次初始化，给与默认的值
        else {               
            newCap = DEFAULT_INITIAL_CAPACITY;
            //临界值等于容量*加载因子
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        //此处的if为上面标记##的补充，也就是初始化时容量小于默认值16的，此时newThr没有赋值
        if (newThr == 0) {
            //new的临界值
            float ft = (float)newCap * loadFactor;
            //判断是否new容量是否大于最大值，临界值是否大于最大值
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        //把上面各种情况分析出的临界值，在此处真正进行改变，也就是容量和临界值都改变了。
        threshold = newThr;
        //表示忽略该警告
        @SuppressWarnings({"rawtypes","unchecked"})
            //初始化
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        //赋予当前的table
        table = newTab;
        //此处自然是把old中的元素，遍历到new中
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                //临时变量
                Node<K,V> e;
                //当前哈希桶的位置值不为null，也就是数组下标处有值，因为有值表示可能会发生冲突
                if ((e = oldTab[j]) != null) {
                    //把已经赋值之后的变量置位null，当然是为了好回收，释放内存
                    oldTab[j] = null;
                    //如果下标处的节点没有下一个元素
                    if (e.next == null)
                        //把该变量的值存入newCap中，e.hash & (newCap - 1)并不等于j
                        newTab[e.hash & (newCap - 1)] = e;
                    //该节点为红黑树结构，也就是存在哈希冲突，该哈希桶中有多个元素
                    else if (e instanceof TreeNode)
                        //把此树进行转移到newCap中
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { /**此处表示为链表结构，同样把链表转移到newCap中，就是把链表遍历后，把值转过去，在置位null**/
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        //返回扩容后的hashMap
        return newTab;
    }
```

上部分内容就是整个扩容过程的操作，下面再来看看删除方法，remove。
 删除元素：
 ```
 public V remove(Object key) {
         //临时变量
         Node<K,V> e;
         /**调用removeNode(hash(key), key, null, false, true)进行删除，第三个value为null，表示，把key的节点直接都删除了，不需要用到值，如果设为值，则还需要去进行查找操作**/
         return (e = removeNode(hash(key), key, null, false, true)) == null ?
             null : e.value;
     }
     
     /**第一参数为哈希值，第二个为key，第三个value，第四个为是为true的话，则表示删除它key对应的value，不删除key,第四个如果为false，则表示删除后，不移动节点**/
     final Node<K,V> removeNode(int hash, Object key, Object value,
                                boolean matchValue, boolean movable) {
         //tab 哈希数组，p 数组下标的节点，n 长度，index 当前数组下标
         Node<K,V>[] tab; Node<K,V> p; int n, index;
         //哈希数组不为null，且长度大于0，然后获得到要删除key的节点所在是数组下标位置
         if ((tab = table) != null && (n = tab.length) > 0 &&
             (p = tab[index = (n - 1) & hash]) != null) {
             //nodee 存储要删除的节点，e 临时变量，k 当前节点的key，v 当前节点的value
             Node<K,V> node = null, e; K k; V v;
             //如果数组下标的节点正好是要删除的节点，把值赋给临时变量node
             if (p.hash == hash &&
                 ((k = p.key) == key || (key != null && key.equals(k))))
                 node = p;
             //也就是要删除的节点，在链表或者红黑树上，先判断是否为红黑树的节点
             else if ((e = p.next) != null) {
                 if (p instanceof TreeNode)
                     //遍历红黑树，找到该节点并返回
                     node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                 else { //表示为链表节点，一样的遍历找到该节点
                     do {
                         if (e.hash == hash &&
                             ((k = e.key) == key ||
                              (key != null && key.equals(k)))) {
                             node = e;
                             break;
                         }
                         /**注意，如果进入了链表中的遍历，那么此处的p不再是数组下标的节点，而是要删除结点的上一个结点**/
                         p = e;
                     } while ((e = e.next) != null);
                 }
             }
             //找到要删除的节点后，判断!matchValue，我们正常的remove删除，!matchValue都为true
             if (node != null && (!matchValue || (v = node.value) == value ||
                                  (value != null && value.equals(v)))) {
                 //如果删除的节点是红黑树结构，则去红黑树中删除
                 if (node instanceof TreeNode)
                     ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                 //如果是链表结构，且删除的节点为数组下标节点，也就是头结点，直接让下一个作为头
                 else if (node == p)
                     tab[index] = node.next;
                 else /**为链表结构，删除的节点在链表中，把要删除的下一个结点设为上一个结点的下一个节点**/
                     p.next = node.next;
                 //修改计数器
                 ++modCount;
                 //长度减一
                 --size;
                 /**此方法在hashMap中是为了让子类去实现，主要是对删除结点后的链表关系进行处理**/
                 afterNodeRemoval(node);
                 //返回删除的节点
                 return node;
             }
         }
         //返回null则表示没有该节点，删除失败
         return null;
     }
```

 删除还有clear方法，把所有的数组下标元素都置位null，下面在来看看较为简单的获取元素与修改元素操作。
 
 获取元素：
 ```
  public V get(Object key) {
         Node<K,V> e;
         //也是调用getNode方法来完成的
         return (e = getNode(hash(key), key)) == null ? null : e.value;
     }
  
     final Node<K,V> getNode(int hash, Object key) {
         //first 头结点，e 临时变量，n 长度,k key
         Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
         //头结点也就是数组下标的节点
         if ((tab = table) != null && (n = tab.length) > 0 &&
             (first = tab[(n - 1) & hash]) != null) {
             //如果是头结点，则直接返回头结点
             if (first.hash == hash && 
                 ((k = first.key) == key || (key != null && key.equals(k))))
                 return first;
             //不是头结点
             if ((e = first.next) != null) {
                 //判断是否是红黑树结构
                 if (first instanceof TreeNode)
                     //去红黑树中找，然后返回
                     return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                 do { //链表节点，一样遍历链表，找到该节点并返回
                     if (e.hash == hash &&
                         ((k = e.key) == key || (key != null && key.equals(k))))
                         return e;
                 } while ((e = e.next) != null);
             }
         }
         //找不到，表示不存在该节点
         return null;
     }
```
修改元素：
      元素的修改也是put方法，因为key是唯一的，所以修改元素，是把新值覆盖旧值。

hashMap的源码分析暂时到这里，主要的增删改查操作都已经分析完了。