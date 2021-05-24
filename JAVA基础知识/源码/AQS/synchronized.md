> synchronized关键字

~~~java
Synchronized关键字解决的是多个线程之间访问资源的同步性，synchronized关键字可以保证被它修饰的方法或者代码块在任意时刻只能有一个线程执行。
其他线程 必须等待当前线程执行完该方法 / 代码块后才能继续执行该方法 / 代码块
~~~



> 并发中的问题

1. 可见性
2. 原子性
3. 有序性

~~~java
可见性：一个线程对共享变量进行修改，另外一个线程会立即得到修改后的最新值
	线程操作的共享资源是线程资源的每个线程都是会一份共享资源的副本，只有共享资源被同步了以后其他线程，线程		才会重新获取共享资源
原子性：在一次或多次操作中，所有的操作都执行且不会受到其他因素的干扰而中断或者的操作都不执行
	因为线程在操作共享资源的时候可能并不是一条指令就完成对共享资源进行了操作，所以可能A线程在执行一条指令后	B线程也在操作了共享资源就会可能导致A线程已经修改了共享资源但是B线程还是拿到是未被修改的共享资源从而结果	 不一致
有序性：是指程序中代码的执行顺序
	因为JAVA在编译和运行时会对代码进行优化，从而导致程序最终的执行顺序不一定和编写的代码顺序一致
	 
~~~



> java内存模型

~~~java
java内存模型是Java虚拟机中所定义一种内存模型，Java内存模型是标准化的，屏蔽了底层不同计算记得区别
	
主内存
	主内存是所有线程共享的，所有的共享变量都存储于主内存

工作内存（栈？）
	每个线程都有自己的工作内存，工作内存值存储该线程多共享变量的副本，线程堆变量的所有操作都必须在工作内存中完成，不能直接读写主内存中的共享资源，不同线程之间也不能直接访问对方工内存中的变量
~~~



> 主内存和工作内存交互

~~~java
Java内存模型定义了以下8种操作来完成，它们都是原子操作（除了对long和double类型的变量）。
- lock(锁定)
作用于主内存中的变量，它将一个变量标志为一个线程独占的状态。
- read(读取)
作用于主内存中的变量，它把一个变量的值从主内存中传递到工作内存，以便进行下一步的load操作。
- load(载入)
作用于工作内存中的变量，它把read操作传递来的变量值放到工作内存中的变量副本中。
- use(使用)
作用于工作内存中的变量，这个操作把变量副本中的值传递给执行引擎。当执行需要使用到变量值的字节码指令的时候就会执行这个操作。
- assign(赋值)
作用于工作内存中的变量，接收执行引擎传递过来的值，将其赋给工作内存中的变量。当执行赋值的字节码指令的时候就会执行这个操作。
- store(存储)
作用于工作内存中的变量，它把工作内存中的值传递到主内存中来，以便进行下一步write操作。
- write(写入)
作用于主内存中的变量，它把store传递过来的值放到主内存的变量中。
- unlock(解锁)
作用于主内存中的变量，解除变量的锁定状态，被解除锁定状态的变量才能被其他线程锁定。

~~~

> synchronized保证原子性的原理

~~~java
synchronized保证只有有一个线程拿到锁，能够进入代码块
~~~

> synchronized保证可见性的原理

~~~java
执行synchronize时，会对应lock原子操作，从而会刷新工作内存中共享变量的值
~~~

> synchronized保证有序性的原理

~~~java
synchronized依然会发生重排序，只是因为有同步代码块，从而保证了只有单线程执行同步代码块的内容，从而保证了有序性
~~~



> synchronized 可重入原理

~~~java
synchronized 锁的对象的中会有一个技术器（recursions变量）会记录线程获的几次锁

可重入的好处：
	避免死锁、更好的封装代码
	
锁的对象的中会有一个技术器（recursions变量）会记录线程获的几次锁，在执行完同步代码块是，计数器会-1，直到清零为止

~~~



> synchronized 不可中断性

~~~java
不可中断：一个线程获得锁以后，另一个线程想要获取锁，必须处于阻塞或等待状态，如果线程不释放锁，第二个线程会一直阻塞等待，不可被中断。
~~~



> synchronized 原理

~~~text
monitorenter 的JVM规范说明：
 	每一个对象都会创建一个monitor对象（C++对象），监视器被占用时会被锁住，其他线程无法获取该monitor。当JVM执行某个线程的某个内部的monitorenter是，它会尝试获取当前对象对应的monitor的所有权，过程如下：
 	1.若monitor的进入数为0，线程可以进入monitor，并将monitor的进入数设置为1，当前线程成为monitor的			owener（所有者）
 	2.若线程已经拥有monitor的所有权，运行它重入monitor，则进入monitor的进入数据+1
	3.如其他线程已占据了monitor的所有权，那么当前尝试获取monitor的所有线程会被阻塞，直到monitor的进入数为0，才能尝试获取monitor的所有权
    
monitor对象的重要的成员变量：owener 持有锁的线程   recursions 记录获取锁的次数

synchronized的锁对象会关联一个 monitor，这个monitor是由JVM的线程执行到同步代码块，发现锁的对象没有monitor则会创建一个C++的monitor对象，此对象有两个比较重要的属性owener 持有锁的线程   recursions 记录获取锁的次数，当一个线程拥有monitor后，其他的线程只有等待



monitorexit 的JVM规范说明：
	1.能执行monitorexit指令的线程一定是拥有当前对象monitor所有权的线程
	2.执行monitorexit时会将monitor的进入次数-1，当monitor的进入为0，当前线程退出monitor，不在拥有monitor，此时其他被monitor阻塞的线程可以尝试获取这个monitor的所有权
	monitorexit插入在方法的结束出和异常处，从而保证每个monitorenter必须有对应的monitorexit
	
同步方法
同步方法会增减ACC_SYNCHRONIZED修饰，会隐式调用monitorenter和monitorexit两个指令，在执行同步方法前会调用monitorenter，在执行完同步方法以后调用monitorexit
~~~



> monitor是重量级锁

~~~java
ObjectMonitor的函数调用涉及到Atomic::cmpxchg_ptr,Atomic::inc_ptr等内核函数，执行同步代码块，没有竞争到锁的对象就会被park（）挂起，竞争到线程会unpark（）唤醒。这个时候就会存在操作系统用户态和内核态的转换，这种切换回消耗大量的系统资源。所以synchronized是java语言中一个重量级的操作
~~~



> 用户态和内核态

~~~java
linux系统的体系架构分为：用户空间（应用程序的活动空间）和内核
内核：本质也是程序，控制计算机的硬件资源，并提供上层应用程序运行的环境
用户空间：上层应用程序活动的空间，应用程序的执行必须依托于内核提供的资源，包括CPU资源，存储资源、I/O资源等
系统调用：为了使上层应用能访问到硬件资源，内核必须提供访问的接口，即系统调用

所有程序初始都是运行与用户空间，此时即为用户运行状态（用户态），但当它调用 系统调用 来执行某些操作时，例：I/O调用，此时需要陷入内核中运行，我们就称进程处于内核运行（内核态），系统调用的过程可以理解为：
	1.用户态程序将一些数据放到寄存器中，或者使用参数创建一个堆栈，以此来表明需要操作系统提供的服务
	2.用户态执行系统调用
	3.CPU切换到内核态，并跳转到内存指定位置的指令
	4.系统调用处理器（system call handler）会读取程序放入内存的数据参数，并执行程序请求的服务
	5.系统调用完成后，操作系统会重置CPU为用户态并返回系统调用的结果
	
	
~~~



> synchronized升级过程



~~~java
对象在内存中分布分为三块：对象头、实例数据、对其填充

HostPost采用instanceOopDesc
~~~





















































































> synchronized原理

~~~java
64位虚拟机，对象头是12byte
对象头组成：Mark Word 占用64bit、Class Pointer（Class Method Address）占用32bit或者是64bit，为32bit是因为开启了指针压缩


com.threadlearn.synchronize.A object internals:
 OFFSET  SIZE      		TYPE DESCRIPTION                               				VALUE
      0     4           (object header)                          	 	01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4           (object header)                           		00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4           (object header)                           		43 c1 00 20 (01000011 11000001 00000000 00100000) (536920387)
     12     1   		boolean A.a                                      		 false    //属性占用的字节
     13     3           (loss due to the next object alignment)			//对其填充，因为虚拟机规范是8的倍数，所以需要对其填充
~~~

>对象状态 

~~~java
无状态  对象刚创建的时候
偏向锁
轻量锁
重量锁
GC标记
~~~



| Mark Word(64bit)                                             | 状态   |
| ------------------------------------------------------------ | ------ |
| unused:25\|identity_hashCode:31\|age:4\|unused:1\|blased_lock:1\|lock:2 | 无锁   |
| thread:54\|epoch:2\|unused:1\|age:4\|balsed_lock:1\|lock:2   | 偏向锁 |
| ptr_to_lock_record:62\|lock:2                                | 轻量锁 |
| ptr_to_lock_record:62\|lock:2                                | 重量锁 |
| \|lock:2                                                     | GC标记 |

~~~java
330bedb4 对象的hasCode值，上面的字段排序是从后往前排的
com.threadlearn.synchronize.A object internals:
 OFFSET  SIZE      TYPE DESCRIPTION                               VALUE
      0     4           (object header)                           01 b4 ed 0b (00000001/*age:4|unused:1|blased_lock:1|lock:2*/ 10110100 11101101 00001011) 
      4     4           (object header)                           33 00 00 00 (00110011 /*identity_hashCode*/ 00000000 00000000 00000000/*unused:25*/) 
      8     4           (object header)                           43 c1 00 20 (01000011 11000001 00000000 00100000) 
     12     1   boolean A.a                                       false
     13     3           (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 3 bytes external = 3 bytes total

~~~



