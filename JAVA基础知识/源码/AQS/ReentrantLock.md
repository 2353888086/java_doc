> ReentrantLock公平锁实现 （FairSync类的lock方法）

~~~java
    final void lock() {
        acquire(1);
    }

    public final void acquire(int arg) {
        //tryAcquire判断是否为加锁状态,加锁成功返回true，
        //线程加锁失败，进入队列
        //addWaiter根据线程创建节点，未初始化时初始化一下head和tail，并将tail指向node节点并返回，从而达到node节点加入到链表中
        //已经初始化将tail指向创建的node节点，node节点热prev执行原tail节点，并且达到了node节点加入到node的链表中，并返回这个node节点
        //acquireQueued将node节点放入队列中
        //加入队列之后线程会立马park，等到解锁之后会被unpark，醒来之后判断自己是否被打断了
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
~~~



> tryAcquire(int acquires)方法

~~~java
 /** Fair version of tryAcquire.  Don't grant access unless
   * recursive call or no waiters or is first.
   * !tryAcquire(arg) */

protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    //判断线程状态，c==0说明线程为自由态
    if (c == 0) {
        //hasQueuedPredecessors 判断线程是否需要入队，需要入队返回true,不进进行CAS操作进行加锁，加锁失败，最后返回false
        //不排队进，行CAS操作进行加锁（原子性操作），加锁成功后，exclusiveOwnerThread设置为当前线程，并返回true
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    //线程非自由态，判断当前线程是否为exclusiveOwnerThread（独占资源的线程，即加锁的线程），
    //为当前线程表示线程重入（说明Reentrantlock是可以重入的），state就进行+1的操作，返回true表示加锁成功
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);//state大于1的时候就证明线程重入了
        return true;
    }
    //线程的状态非自由态当前线程非加锁线程、需要进行排队、CAS操作未成功，都表示线程是未加锁成功
    return false;
}
~~~



> hasQueuedPredecessors()方法

```java
    public final boolean hasQueuedPredecessors() {
        // 
        Node t = tail; //队列尾
        Node h = head; //队列头
        Node s;
        return h != t && ((s = h.next) == null || s.thread != Thread.currentThread());
        /**
         * h != t 判断节点是否需要排队
         *     在java1.6之前。ReentrantLock比synchronized快的只要原因就是
         *      在线程交替执行，没有阻塞的的时候ReentrantLock是在JAVA层面解决问题，没有OS（没有将用户态转成内核态）
         *      synchronized是重量级锁，就算是线程交替执行也会有OS操作，（将用户态转成内核态）
         *
         * h != t && (s = h.next) == null || s.thread != Thread.currentThread()
         *  node节点的头尾有一个不为空，并且（尾节点有值 || 尾部节点的线程与当前线程不一致），
         *      返回false，!hasQueuedPredecessors()为true，线程进行排队。
         *
         * 代码逻辑分析：
         *  h != t为false：
         *      一、表示队列（存放线程的队列）未被初始化
         *          h != t为false表示即队列的头和尾都为null，队列未被初始化返回的是false就表示不需要排队，直接进行CAS修改state值，
         *          但compareAndSetState可能会失败，如果两个线程同时进行CAS操作只会有一个线程可以执行成功,另一个线程肯定失败，则失败的线程则会进行排队。
         *
         *      二、表示队列（存放线程的队列）被初始化，但是对立长度为1
         *          队列中的线程都加锁后释放了锁，A线程刚好处理最后一个线程结束，那A线程就不用进行排队，直接进行加锁操作
         *          如同，A排队买票，A刚准备排队的时候在他前面的B买票结束了，那A就不用排队了，直接买就可以了
         *
         *  h != t为true：表示队列（存放线程的队列）肯定被初始化，且队列长度大于1
         *
         *  (s = h.next) == null为false表示队列长度大于1
         *      如果s.thread != Thread.currentThread()返回true，最终结果就是false就表示
         *          当前线程，不是 队列中排在第一位线程，则就要进行排队，
         *
         *          （队列中排在第一位线程并不是指队列的位置为0的节点。而是对立中位置为1的节点，
         *          队列中位置为0的元素可以理解为是正在加锁执行的线程，但是加锁的线程是不在队列中的，所以虚拟一个空节点占据队列中0的位置说明有线程在执行加锁代码了，
         *          如同：排队买票A正在买票的不能说是排队的只能说是正在办理就如线程正在执行加锁的代码，只有B等待A票结束，才能说B在排队且是排在第一个位置）
         *
         *      如果s.thread != Thread.currentThread()返回false，最终结果就是false就表示不需要排队，
         *           当前线程，是 队列中排在第一位线程，则就要进行排队当前线程，不需要排队，
         *           则调用 compareAndSetState(0, acquires) 去改变计数器尝试上锁；
         *              两种情况：
         *              一加锁成功：
         *                  持有锁的那个线程执行完释放了锁，s线程就会加锁成功
         *              二加锁失败：
         *                  也就是持有锁的那个线程没执行完没释放锁，s加锁就会失败直接返回false。
         *
         *
         */

    }
```





> addWaiter(Node mode)方法

~~~java
    private Node addWaiter(Node mode) {
        //根据线程创建一个Node节点
        Node node = new Node(Thread.currentThread(), mode);//创建的node节点的，waitStatus是空的，因为mode是null
        // Try the fast path of enq; backup to full enq on failure
        //将tail尾部节点取出放到pred
        Node pred = tail;//head和tail可以理解为这个队列指向头和尾的指针
        //判断tail是否为空，不为空表示head和tail已经初始化了（未被初始化时head和tail都是null）
        //存放node节点的队列只有一个元素头和尾都是同一个元素
        if (pred != null) {
            //将线程创建的node节点prev指向链表的尾部节点
            node.prev = pred;
            //通过CAS操作将tail尾部指针指向线程创建的node节点，修改tail的指向，
            //使用CAS是防止多线程操作tail指针，确保原子操作
            if (compareAndSetTail(pred, node)) {
                //操作成功后，将原尾部节点的next指向新的尾部节点node，从而达到node节点如队列的操作
                pred.next = node;
                //将放入队尾的node节点返回，通过acquireQueued放入到队列中
                return node;
            }
        }
        //上面的操作针对的是链表已经初始化了，对Node节点进行了一次入队操作，
        //没有成功则通过enq方法的链表已经初始化分支进行自旋，将Node节点放入到链表中（链表已经初始化分支和上面的操作是一样的）
        enq(node);
        return node;
    }

	private Node enq(final Node node) {
        //开启自旋，node为根据线程创建的节点
        for (;;) {
            Node t = tail;
            //tail 尾部节点为空，队列未被初始化，进行初始化操作，
            if (t == null) { // Must initialize
                //通过CAS操作，创建一个空的node节点放入head，这就解释了为啥队列排队是以下标为1开始的原因
                //头部的节点是Node节点，是因为用这个空的node节点来代表正在独占资源得线程，后面的线程就是正在排队的线程
                //CAS成功将head放入tail，则头和尾指向同一个元素
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                //初始化成功，根据线程创建的节点的prev指向，原tail的节点
                node.prev = t;
                //通过CAS操作将tail指向线程创建的node节点
                if (compareAndSetTail(t, node)) {
                    //原tail节点的next指向线程创建的node节点（即新的tail节点所指向的元素）
                    t.next = node;
                    return t;//结束循环
                }
            }
        }
    }
~~~





> acquireQueued(final Node node, int arg)方法



~~~java
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            //线程是否被打断的标志
            boolean interrupted = false;
            for (;;) {
                //获取node（A）节点的上一个node（B）节点，一node（B）为head节点，二、node（B）为非head节点
                final Node p = node.predecessor();
                //node（B）为head节点，则node（A）会进行一次尝试加锁
                //因为尝试加锁，会判断这个节点是否需要进行排队，如果上个线程释放了锁，并且node（A）这个节点不需要进行排队，那么node（A）就可以不用进行park的操作
                if (p == head && tryAcquire(arg)) {
                    //setHead和 p.next 是为了将head节点指向node（A）这个节点，
                    // 并且将原head指向的节点node（B）的引用指向全部去除（将node（B）节点出队列），以便后续进行GC回收
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;//不执行cancelAcquire操作
                    //返回interrupted为false，不会调用selfInterrupt()会再次进行对线程进行interrupt()操作，
                    return interrupted;
                }
                //shouldParkAfterFailedAcquire(p, node)会修改node（A）节点之前的node（B）节点的waitStatus 为-1返回true；
                // 且parkAndCheckInterrupt()会对线程进行park操作，并调用Thread.interrupted()方法，
                // 从而最终返回interrupted=true，导致lock（）方法里会调用selfInterrupt(),再次打断线程，
                // 还原node（A）线程进入之前interrupted的值 从而达到还原用户行为的作用，
                // 因为lockInterruptibly() 这个方法调用的也是acquireQueued()方法,会进行Thread.interrupted()操作,
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
~~~



> shouldParkAfterFailedAcquire(Node pred, Node node)

~~~java
   /** waitStatus value to indicate successor's thread needs unparking */
    static final int SIGNAL    = -1;//后继线程需要unparking

	private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;//默认是0.node节点标记线程的等待状态
        if (ws == Node.SIGNAL)
            //pred线程等待状态为-1.node节点可以进行unpark，
            //如果前驱变成了head，并且head的代表线程exclusiveOwnerThread释放了锁，就会来根据这个SIGNAL来唤醒自己
            return true;
        if (ws > 0) {
             //的前驱的状态大于0，即CANCELLED。说明前驱节点已经因为超时或响应了中断，
             //消了自己。所以需要跨越掉这些CANCELLED节点，直到找到一个<=0的节点
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            //ws只能是0或PROPAGATE。CAS设置ws为SIGNAL
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }

shouldParkAfterFailedAcquire只有在检测到前驱的状态为SIGNAL才能返回true，只有true才会执行到parkAndCheckInterrupt。
shouldParkAfterFailedAcquire返回false后，进入下一次循环，当前线程又会再次尝试获取锁（p == head && tryAcquire(arg)）。或者说，每次执行shouldParkAfterFailedAcquire，都说明当前循环 尝试过获取锁了，但失败了。
如果刚开始前驱的状态为0，那么需要第一次执行compareAndSetWaitStatus(pred, ws, Node.SIGNAL)返回false并进入下一次循环，第二次才能进入if (ws == Node.SIGNAL)分支，所以说 至少执行两次。
死循环保证了最终一定能设置前驱为SIGNAL成功的。（考虑当前线程一直不能获取到锁）

~~~



> unlock方法

~~~java
	public void unlock() {
        sync.release(1);
    }

	//
    public final boolean release(int arg) {
        if (tryRelease(arg)) {//释放持有资源的线程
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);//唤醒后继节点
            return true;
        }
        return false;
    }
~~~



> tryRelease(int arg)方法

~~~java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    // 如果当前线程不是持有锁的线程，抛出异常。判断了线程是不是持有资源得线程所以可以不适用CAS操作
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);//将持有资源的线程释放
    }
    setState(c);
    return free;
}
~~~



> unparkSuccessor(Node node)方法

~~~java
    private void unparkSuccessor(Node node) {
        
        int ws = node.waitStatus;//node节点值小于0，因为在再阻塞之前有设置状态
        if (ws < 0)
            //如果当前节点的状态小于0，就通过CAS操作尝试修改状态为0(初始状态)
            node.compareAndSetWaitStatus(ws, 0);

        Node s = node.next;
        if (s == null || s.waitStatus > 0) {//如果为空或者已经取消/被中断
            s = null;
             //从尾部倒着遍历，找到离当前节点最近的节点状态小于等于0的节点（回溯）
            for (Node p = tail; p != node && p != null; p = p.prev)
                if (p.waitStatus <= 0)
                    s = p;
        }
        if (s != null)
            LockSupport.unpark(s.thread);//唤醒下个节点线程
    }
~~~

