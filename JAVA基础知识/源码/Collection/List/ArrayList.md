> ArrayList的底层数据结构

~~~java
//list底层就是一个Object数组
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
//空Object数组
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
//指定大小就创建指定大小的数组
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}
~~~

> 继承RandomAccess, Cloneable, java.io.Serializable的作用

~~~java
RandomAccess这是标记接口，继承这个接口则随机访问速度更快
		//如果list对象是RandomAccess，使用随机访问，通过下标去迭代
        if(a instanceof RandomAccess){
            for (int i1 = 0; i1 < a.size(); i1++) {
                
            }
        }else{
        	//否则就是使用顺序访问
            for (Object o : a) {
                
            }
        }


Cloneable标记接口，继承这个接口重写colne方法，可以对类进行克隆操作
克隆有浅克隆和深克隆两种
浅克隆针对于属性是引用对象类型的时候，因为克隆的对象和源对象的引用类型的属性是同一个，所以在修该引用对象的时候，克隆的对象的属性也被修改了
需要实现接口 Cloneable 和重写 clone()方法
	@Override //浅克隆
    public Object clone() throws CloneNotSupportedException {
        return super.clone();
    }
深克隆就是将对象的引用类型的属性也克隆一份，重新复制给克隆对象
    @Override   //深克隆
    public Object clone() throws CloneNotSupportedException {
        BEobj b =  (BEobj)bEobj.clone();
        AEobj aEobj = (AEobj)super.clone();
        aEobj.setbEobj(b);
        return aEobj;
    }
~~~



> 构造方法

~~~java
无参构造放初始化是一个空的数据，为10的数据时在add方法调用的时，如果为空则创建一个为长度10的数组
~~~



> add方法

~~~java
    public boolean add(E e) {
        //计算数组是否需要扩容，需要扩容者进行扩容操作
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }

    private void ensureCapacityInternal(int minCapacity) {
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
    }
	
	//计算数组大小，如果数组为空则返回初始大小DEFAULT_CAPACITY = 10，否则返回传进来的值
    private static int calculateCapacity(Object[] elementData, int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        return minCapacity;
    }

	
    private void ensureExplicitCapacity(int minCapacity) {
        //记录修改次数。
        modCount++;
        // overflow-conscious code
        //下标长度大于数组长度进行扩容
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }

    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);//扩容大小为原数据的1.5倍  (oldCapacity >> 1)右移1位。大小缩小为原来的2分之一
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        //扩容是要把原来的数组复制一份到新数组中，所以扩容消耗内存和时间
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

public void add(int index, E element) {
        rangeCheckForAdd(index);
		//计算和判断数据是否需要扩容，需要则进行扩容
        ensureCapacityInternal(size + 1);  // Increments modCount!!
    	//在index两端复制，并空出来index的位置并赋值为element
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }

~~~



> get方法

~~~java
	public E get(int index) {
        rangeCheck(index);
		//直接返回数组对应位置的值
        return elementData(index);
    }

	//判断是否越界，判断越界是根据size并不是根据数据的大小来确认的
	private void rangeCheck(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }

~~~



> remove方法

~~~java
    public boolean remove(Object o) {
        //循环数组将数组中第一个为null的元素移除
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            //循环数组将数组中第一个为o的元素移除
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }

	//每次移除数组都要复制一份
 	private void fastRemove(int index) {
        modCount++;
        //计算需要移动的个数，比如：数组长度为5，要移除的下标是2 ，那么计算后的numMoved就是2，即下标后面还需要复制的长度
        int numMoved = size - index - 1;
        if (numMoved > 0)
            //根据下标把数组这个链表从指定的下标出分开，分成两个链，第一条是从index+1开始，一条到index开始，但是index这个存的是index+1下标的值
            //elementData, index,目标数组是从0到index，原数组 elementData, index+1开始，复制numMoved的长度
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }


~~~



> 修改方法

~~~java
	public E set(int index, E element) {
        rangeCheck(index);
		//直接根据下标找到对应位置的值替换，并返回oldvalue
        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }
~~~



> 清空方法

~~~java
    //循环遍历将数组所有的元素设为null，size设置为0，list对外的大小是根据size来确认的，
	//实际上底层的数组大小是没有改变的，还是清空前的数组大小，如果没有被使用后面会被GC回收 
	public void clear() {
        modCount++;

        // clear to let GC do its work
        for (int i = 0; i < size; i++)
            elementData[i] = null;

        size = 0;
    }
~~~



> 包含方法

~~~java 
	public boolean contains(Object o) {
        return indexOf(o) >= 0;
    }
	//遍历寻找o对应的下标，找到第一个就直接返回，并不会寻找全部的o对应的下标，结束循环
    public int indexOf(Object o) {
        if (o == null) {
            for (int i = 0; i < size; i++)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = 0; i < size; i++)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }
~~~



>迭代器

~~~java

        //游标，要返回的下一个元素的索引
        int cursor = 0;

        //返回的元素的索引，默认为-1
        int lastRet = -1;

      	//将修改次数赋值给预期修改次数，
        int expectedModCount = modCount;
    	
        public boolean hasNext() {
            return cursor != size();
        }

        public E next() {
            checkForComodification();//判断list是否被修改
            int i = cursor;//返回索引赋值给i，i理论上是不会和size相等的，在hasNext方法进了判断，肯定不是与size相等
            if (i >= size)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i + 1;//将下个元素的下标给cursor
            return (E) elementData[lastRet = i];//返回当前下标的值
        }

        public void remove() {
            //必须先调用next方法才可以调用remove，因为lastRet默认为-1
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();

            try {
                ArrayList.this.remove(lastRet);//移除lastRet对应的元素
                cursor = lastRet;//将cursor重新定位到lastRet这个位置
                lastRet = -1;//lastRet重置
                expectedModCount = modCount;//x修改预期次数与mod次数一致
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }

    	//主要是根据修改修改次数和预期修改次数判断是否会发生线程修改异常，
    	//创建了迭代器以后，ArrayList发生了修改操作，导致这个异常
     	final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
~~~





