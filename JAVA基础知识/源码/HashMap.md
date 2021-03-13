# HashMap

## 初始化

> 创建HashMap对象的时候是加载了默认的加载因子0.75f

```java
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }

	DEFAULT_LOAD_FACTOR = 0.75f
```

> 为什么加载因子默认是0.75

```
这个加载因子实际上就是HashMap进行扩容的判断条件，
0.75只是一个折中的方式，
0.5的话会导致链表长度达到一半就会进行扩容操作，最终会导致使用空间和未使用空间的差值会逐渐增加，空间利用率低下
1的话会导致链表被全部占用才会进扩容，在一定程度上会增加put时候的时间

JDK中
作为一般规则，默认负载因子（0.75）在时间和空间成本上提供了很好的折衷。
较高的值会降低空间开销，但提高查找成本（体现在大多数的HashMap类的操作，包括get和put）。
设置初始大小时，应该考虑预计的entry数在map及其负载系数，并且尽量减少rehash操作的次数。
如果初始容量大于最大条目数除以负载因子，rehash操作将不会发生。

在随机哈希值的情况，对于loadfactor = 0.75 ，虽然由于粒度调整会产生较大的方差，桶中的Node的分布频率服从参数为0.5的泊松分布。 

```

## map的put方法

> (hash(key)方法详解

~~~java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

//JDK1.8 中作用：为了获取的hashcode更为分散
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);//>>>是无符号右移
}
//JDK1.7 代码
static int indexFor(int h, int length) {
    return h & (length-1);
}

/*
	h & (length-1) 
	由于绝大多数情况下length一般都小于2^16即小于65536，所以return h & (length-1);结果始终是h的低16位与（length-1）进行&运算
	
	例如1：假设length为8。HashMap的默认初始容量为16
	length = 8;  （length-1） = 7；转换二进制为111；
	假设一个key的 hashcode = 78897121 转换二进制：100101100111101111111100001，与（length-1）& 运算如下

    0000 0100 1011 0011 1101 1111 1110 0001
	&运算
    0000 0000 0000 0000 0000 0000 0000 0111
 
	=   0000 0000 0000 0000 0000 0000 0000 0001 （就是十进制1，所以下标为1）
	上述运算实质是：001 与 111 & 运算。也就是哈希值的低三位与length与运算。
	如果让哈希值的低三位更加随机，那么&结果就更加随机，如何让哈希值的低三位更加随机，那么就是让其与高位异或。
	
	JDK 1.8 改动原因：
	由于和（length-1）运算，length 绝大多数情况小于2的16次方。所以始终是hashcode 的低16位（甚至更低）参与运算。要是高16位也参与运算，会让得到的下标更加散列。

	所以这样高16位是用不到的，如何让高16也参与运算呢。所以才有hash(Object key)方法。
	让他的hashCode()和自己的高16位^运算。所以(h >>> 16)得到他的高16位与hashCode()进行^运算。
	
	使用^而不用&和|，是因为&和|都会使得结果偏向0或者1 ,并不是均匀的概念,所以用^，使其更加分散
	
*/


~~~

> putVal方法详解，插入方式是头插

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;.
        //首次put数据会调用resize对进行HashMap进行初始化，并进行put操作
        //初始化的是一个Node节点数组，数组元素是Node对象，数组的node对象会记录下面的Node对象，组成了一个单向链表（如同T），所以可以说hashMap是一个数组和链表的结合，
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
    	//i = (n - 1) & hash    hashcode 与 table.length-1进行&运行，最低值0最高值时length-1，来确定下标位置，并且确保下标不会超过最大值
    	//且计算的结果在数组中不存在则直接新增一个Node节点
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            //p为的数组key所对应小标的Node对象，即数组中的Node对象
            Node<K,V> e; K k;
            //判断p这个Node对象的hash值和key是否相等，就是判断链表头是不是要找的元素
            //(k = p.key) == key  这句话就说明key可以为null，
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            //TreeNode<K,V>这个节点是 LinkedHashMap.Entry，这个TreeNode是Node节点的子类所以可以强转
            //p这个Node对象不在数组中，并且是一个树节点
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                 //p这个Node对象不在数组中 ，但是确定了下标进而需要链表的属性往下寻找Node节点，进而在链表的尾部新增一个Node节点
                for (int binCount = 0; ; ++binCount) {//一直循环寻找，直到满足需求
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);//尾部插入
                        if (binCount >= TREEIFY_THRESHOLD - 1) //链表长度大于8，链表进行转换
                            treeifyBin(tab, hash);
                        break;
                    }
                    //根据key找到node对象，直接退出for循环
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;//将p的next节点重新赋值给p，进行循环知道找到对应的node对象
                }
            }
            //循环结束，找到或者已经插入了Node对象，且值e就是Node对象，判断不为空，进行value更换，并将老的value返回
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);//提供扩展的方法
                return oldValue;
            }
        }
    	//操作数+1
        ++modCount;
    	//Map大小+1，判断是否需要进行扩容
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);//提供扩展的方法
        return null;
    }
```



> treeifyBin方法详解

~~~java

//当链表长度大于8的并不是直接转成红黑树，而是先扩容，当数组长度大于等于64时候，链表再转化为红黑树
//tab.length数组的长度小于64且链表长度大于8的时候才会转成红黑树
final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
    	// 如果元素数组长度已经大于等于了 MIN_TREEIFY_CAPACITY，那么就有必要进行结构转换了
   		 // 根据hash值和数组长度进行取模运算后，得到链表的首节点
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            TreeNode<K,V> hd = null, tl = null;
            do {
                TreeNode<K,V> p = replacementTreeNode(e, null);// 将Node节点转换为 树节点
                if (tl == null)
                    hd = p;
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
            // 到目前为止 也只是把Node对象转换成了TreeNode对象，把单向链表转换成了双向链表
        	// 把转换后的双向链表，替换原来位置上的单向链表
            if ((tab[index] = hd) != null)
                hd.treeify(tab);
        }
    }
~~~



> resize方法详解

- 什么时候进行resize操作？

  ​	有两种情况会进行resize：1、初始化table；2、在size超过threshold之后进行扩容

- 扩容后的新数组容量为多大比较合适？

  ​	扩容后的数组应该为原数组的两倍，并且这里的数组大小必须是2的幂

- 节点在转移的过程中是一个个节点复制还是一串一串的转移？

  ​	从源码中我们可以看出，扩容时是先找到拆分后处于同一个桶的节点，将这些节点连接好，然后把头节点存入桶中即可



~~~java
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
    	//初始化时oldCap和oldThr都是0
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
    	//oldCap > 0的时候就说明，hashmap是已经初始化了，再次进入就代表是要进行扩容或者是改变hashMap的数据结构了
        if (oldCap > 0) {
            //扩容超过最大值，直接返回
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            //扩容直接为原数组2的次幂直接+1，就是2倍（oldCap << 1）
            //扩容后小于最大值，且原来的数组必须大于等于初始值16才能扩容
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
    	//oldThr = threshold,threshold这个值在进行初始化的是没有赋值的，初始化的时候这个值是0
    	//HashMap(int initialCapacity)这个构造方法会有一步 this.threshold = tableSizeFor(initialCapacity);
    	//根据threshold进行初始化hashMap就是上面指定Map的大小
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            //初始化的默认值，在没有指定构造参数的时候
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
    	// HashMap(int initialCapacity) 根据threshold进行初始化hashMap只设置了newCap
    	//根据newCap计算下次需要调整大小的值
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
    	//初始化的步骤结束
        @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
    
    	//初始化并且链表长度大于8以后，调整数据结构具体实现
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    //oldTab[j]=null清除旧表的引用,后续进行垃圾回收
                    oldTab[j] = null;
                    //表示oldtale中，这个Node对象是没有形成链表就一个单独的节点
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;//直接把这个下标的职位赋值给e这个node对象
                    //如果是树节点就进行树节点的拆分
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do { 
                            next = e.next;
                            //正常通过e.hash & (oldCap - 1)判断位置
                            //扩容后 e.hash & oldCap
                            //因为数组的长度是2的次幂
                            //假设 原来的数组长度为 8 ，
                            //(oldCap - 1)操作后 2进制就是 0000 0111 
                            //oldCap的二进制就是           0000 1111
                            //进而会导致 e.hash & (oldCap - 1) 和 e.hash & oldCap 最后的运算结果 ：要么原值不变 和 要么为 原值 + 0000 1000     
                            //所以直接通过(e.hash & oldCap) == 0作为判断条件，如果不为0 则直接原来位置加上0000 1000。
                            //do循环里的操作就是 将原来的链表重新分配， 分为两组 原来位置的一组 和 新位置的一组 ，最后将头部放入数组对应的位置
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
                        //最后将头部放入数组对应的位置
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
        return newTab;
    }
~~~



> Node对象

~~~java
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
    }
~~~

