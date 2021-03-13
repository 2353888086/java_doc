> LinkedList的底层属性

~~~java
    //LinkedList实际上就是有node节点组成的一个双向的链表，同时有两个node分别为frist和last分别指向头尾的节点，元素可以有null，可以重复
	transient int size = 0;
    /**
     * Pointer to first node.
     */
    transient Node<E> first;
    /**
     * Pointer to last node.
     */
    transient Node<E> last;

	private static class Node<E> {
        E item;//记录元素数据
        Node<E> next;//指向下一个节点
        Node<E> prev;//指向上一个节点

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
~~~



> LinkedList的构造方法

~~~java
	//无参构造，就是一个空的什么都没有，只有在add的时候才会生产链表
	public LinkedList() {
    }

   //有参构造
    public LinkedList(Collection<? extends E> c) {
        this();
        addAll(c);
    }
	public boolean addAll(Collection<? extends E> c) {
        return addAll(size, c);
    }
	//根据无参构造方法创建linkedlist，使用addAll（1，a数组），checkPositionIndex这个方法会报错
	public boolean addAll(int index, Collection<? extends E> c) {
        checkPositionIndex(index);// index >= 0 && index <= size  size在0到size之间来判断index是否存在

        Object[] a = c.toArray();
        int numNew = a.length;//获取传进来的Collection的子类长度
        if (numNew == 0)
            return false;

        Node<E> pred, succ;
        if (index == size) {//如果index与size相等，就直接在原链表后面追加，调用时无参的addAll，传进来的index就是size的值
            succ = null;
            pred = last;
        } else {//不是在链表后面追加，则根据inde获取对应的node节点
            succ = node(index);
            pred = succ.prev;//node节点的前一个就记录为pred
        }

        for (Object o : a) {
            @SuppressWarnings("unchecked") E e = (E) o;
            Node<E> newNode = new Node<>(pred, e, null);
            if (pred == null)//链表是空的时候才会走这个方法，将first赋值为新创建的node节点
                first = newNode;
            else
                pred.next = newNode;//原来的node节点的下一个元素指向新的node节点
            pred = newNode;//延续链表，将新创建的node加点赋值为pred，通过循环上面的操作从而达到延续链表的操作
        }

        if (succ == null) {//在尾部条件的链表，直接把最后创建的node节点直接赋值非last下标即可
            last = pred;
        } else {//将生成的链表的next指向，节点断开的node节点，断开的node节点将pre重新指向到生成链表的尾部，组装 延长链表
            pred.next = succ;
            succ.prev = pred;
        }

        size += numNew;
        modCount++;
        return true;
    }

    Node<E> node(int index) {
        // assert isElementIndex(index);
		//index小于size的一半，从头遍历
        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {//index大于或等于size的一半，从尾部遍历
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
~~~





> add方法

~~~java
	public boolean add(E e) {
        linkLast(e);
        return true;
    }

    void linkLast(E e) {
        final Node<E> l = last;//如果是新增，则这个node节点为null
        final Node<E> newNode = new Node<>(l, e, null);//创建一个node节点，pre指向原链表的尾部节点
        last = newNode;
        if (l == null)
            first = newNode;//初始化，first为新创建节点
        else
            l.next = newNode;//原尾结点的next指向新的node节点，链接在一起组成延长链表
        size++;
        modCount++;
    }
~~~



> get方法

~~~java
 	public E get(int index) {
        checkElementIndex(index);
        return node(index).item;
    }
    Node<E> node(int index) {
        // assert isElementIndex(index);
		//index小于size的一半，从头遍历，按照循序遍历的
        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {//index大于或等于size的一半，从尾部遍历
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
~~~

> remove方法

~~~java
  	 //根据链表循环遍历，找到第一个就删除，并退出
	public boolean remove(Object o) {
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null) {
                    unlink(x);
                    return true;
                }
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item)) {
                    unlink(x);
                    return true;
                }
            }
        }
        return false;
    }

	//
	E unlink(Node<E> x) {
        // assert x != null;
        final E element = x.item;
        final Node<E> next = x.next;
        final Node<E> prev = x.prev;

        if (prev == null) {//判断是不是为first节点
            first = next;//将next赋值给头节点
        } else {
            prev.next = next;//将x节点pre属性指向的node节点的next属性指向x节点next属性的node节点
            x.prev = null;//断掉x的pre节点
        }

        if (next == null) {//判断是不是为last节点
            last = prev;//将x的pre节点赋值给last节点
        } else {
            next.prev = prev;//将x节点next属性指向的node节点的pre属性指向x节点prev属性的node节点
            x.next = null;;///断掉x的next节点
        }

        x.item = null;//清空，方便GC回收
        size--;
        modCount++;
        return element;
    }
~~~



> set方法

~~~java
    public E set(int index, E element) {
        checkElementIndex(index);
        Node<E> x = node(index);//获取节点
        E oldVal = x.item;//修改值
        x.item = element;
        return oldVal;
    }
~~~



> contains方法

~~~java
    public boolean contains(Object o) {
        return indexOf(o) != -1;
    }

	//循环遍历，知道直接返回下标	
    public int indexOf(Object o) {
        int index = 0;
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null)
                    return index;
                index++;
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item))
                    return index;
                index++;
            }
        }
        return -1;
    }
~~~



> clear方法

~~~java
    public void clear() {
        // 循环将所有的node节点属性清空，后面通过GC回收，大数据的linkedlist清空，会导致有很多node节点垃圾对象，
        for (Node<E> x = first; x != null; ) {
            Node<E> next = x.next;
            x.item = null;
            x.next = null;
            x.prev = null;
            x = next;
        }
        first = last = null;
        size = 0;
        modCount++;
    }
~~~



> Iterator迭代器（DescendingIterator）

~~~java
    public Iterator<E> descendingIterator() {
            return new DescendingIterator();
        }
    private class DescendingIterator implements Iterator<E> {
        private final ListItr itr = new ListItr(size());
        public boolean hasNext() {
            return itr.hasPrevious();
        }
        public E next() {
            return itr.previous();
        }
        public void remove() {
            itr.remove();
        }
    }
	ListItr(int index) {
            // assert isPositionIndex(index);
            next = (index == size) ? null : node(index);//index是size，next为空
            nextIndex = index;//就是linkedlist的大小
        }

	public boolean hasPrevious() {
        return nextIndex > 0;//因为linkedlist迭代器是从后往前遍历，会进行nextIndex--的操作
    }

    public E previous() {
        checkForComodification();
        if (!hasPrevious())
            throw new NoSuchElementException();

        //next是迭代器里的属性，刚开始时候是null，next第一个元素就是last，所以linkedlist是从后往前遍历的
        lastReturned = next = (next == null) ? last : next.prev;
        nextIndex--;
        return lastReturned.item;
    }

    public void remove() {
        checkForComodification();
        //需要先调用next，才能调用remove，防止下标越界
        if (lastReturned == null)
            throw new IllegalStateException();

        Node<E> lastNext = lastReturned.next;//获取lastReturned的next节点
        unlink(lastReturned);//删除lastReturned对应的节点
        //删除的节点正好是下次迭代的节点，则next重新赋值为lastReturned节点的下一个节点
        if (next == lastReturned)
            next = lastNext;
        else
            nextIndex--;
        lastReturned = null;
        expectedModCount++;
    }

	final void checkForComodification() {//判断预期修改次数和实际修改次数是不是一致的，防止出现线程修改异常，防止迭代的过程中其他线程修改了会出现bug的问题
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
~~~



> push方法，押栈

~~~java
    public void push(E e) {
        addFirst(e);
    }

	public void addFirst(E e) {
        linkFirst(e);
    }

    private void linkFirst(E e) {
        final Node<E> f = first;
        final Node<E> newNode = new Node<>(null, e, f);//创建prev为null，next为原first节点的Node对象
        first = newNode;//first重新复制
        if (f == null)//初始化时，last也是新建node节点
            last = newNode;
        else
            f.prev = newNode;//原first节点的prev指向新的Node节点
        size++;
        modCount++;
    }
~~~



> poll方法，出栈

~~~java
    public E poll() {
        final Node<E> f = first;
        return (f == null) ? null : unlinkFirst(f);
    }

    private E unlinkFirst(Node<E> f) {
        // assert f == first && f != null;
        final E element = f.item;
        final Node<E> next = f.next;//获取第二个node节点
        f.item = null;
        f.next = null; // help GC 清空原first节点的全部属性引用
        first = next;//first重新引用值原来第二个节点
        if (next == null)//判断是不是最后一个node，是last也清空
            last = null;
        else
            next.prev = null;//清空原第二节点的prev，彻底称为first节点
        size--;
        modCount++;
        return element;
    }
~~~



