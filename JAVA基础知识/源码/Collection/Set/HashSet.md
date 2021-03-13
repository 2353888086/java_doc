> HashSet底层原理就是HashMap的针对与Key的方法

~~~java
    //HashSet底层就是Hashmap，不过他的数据时放到HashMap的Key中的
	public HashSet() {
        map = new HashMap<>();
    }
	//HashMap中Key的计算
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

> set方法

~~~java
    public boolean add(E e) {
        return map.put(e, PRESENT)==null;
    }
	private static final Object PRESENT = new Object();//私有静态常量
	//具体看HashMap的put方法
	public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
~~~

