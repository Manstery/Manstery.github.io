@[TOC](ArrayBlockingQueue)

## ArrayBlockingQueue
&#8195;&#8195;ArrayBlockingQueue底层使用环形数组实现阻塞队列，因此为有界队列，其容量上限在实例化时通过传入的参数capacity决定，本质上就是实例化了一个长度为capacity的数组。
![ArrayBlockingQueue](https://img-blog.csdnimg.cn/20210418221225466.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2NjkwNjg=,size_16,color_FFFFFF,t_70#pic_center)

```java
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {
```
+ #### 使用变量
  + Condition 对象简介：
&#8195;&#8195;Condition是在java 1.5中才出现的，它用来替代传统的Object的wait()、notify()实现线程间的协作，相比使用Object的wait()、notify()，使用Condition的await()、signal()这种方式实现线程间协作更加安全和高效。
&#8195;&#8195;Condition是个接口，基本的方法就是await()和signal()方法；Condition依赖于Lock接口，生成一个Condition的基本代码是lock.newCondition()；调用Condition的await()和signal()方法，都必须在lock保护之内，就是说必须在lock.lock()和lock.unlock之间才可以使用。
&#8195;&#8195;Conditon中的await()对应Object的wait()；Condition中的signal()对应Object的notify()；Condition中的signalAll()对应Object的notifyAll()。
引用自：<https://blog.csdn.net/a1439775520/article/details/98471610>
  + Itrs 对象简介：
&#8195;&#8195;ArrayBlockingQueue队列集合中所有的迭代器都在Itrs迭代器组中进行管理，这些迭代器将在Itrs迭代器组中以单向链表的方式进行排列。所以ArrayBlockingQueue队列需要在特定的场景下，对已经失效、甚至已经被垃圾回收的迭代器管理节点进行清理。
&#8195;&#8195;例如，当ArrayBlockingQueue队列有新的迭代器被创建时（并为非独立/无效工作模式），Itrs迭代器组就会尝试清理那些无效的迭代器，其工作逻辑主要由Itrs.doSomeSweeping(boolean)方法进行实现。
引用自：<https://blog.csdn.net/yinwenjie/article/details/105869156>
```java
    final Object[] items; //底层数组实现
    int takeIndex; //队列头指针
    int putIndex; //队列尾指针
    int count; // 当前队列中的对象（任务）数
    final ReentrantLock lock; //使用可重入(默认非公平)锁对象加锁
    private final Condition notEmpty; // 用于在队列满发生写阻塞时进行线程通信
    private final Condition notFull; //  用于在队列空发生读阻塞时进行线程通信
    transient Itrs itrs = null; // 迭代器组对象
```
+ #### 底层调用方法
  + `checkNotNull(Object v)`：检查当前传入的任务对象是否为null，若为null报空指针异常
  +  `enqueue(E x)`：向队列尾插入元素，内部构建了环形队列，并维护了当前任务数
 ![enqueue](https://img-blog.csdnimg.cn/20210418221732452.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2NjkwNjg=,size_16,color_FFFFFF,t_70#pic_center)
  + `dequeue()`：从队列头取出元素，内部构建了环形队列，并维护了当前任务数
  ![dequeue](https://img-blog.csdnimg.cn/20210418222256283.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2NjkwNjg=,size_8,color_FFFFFF,t_70#pic_center)

```java
    private static void checkNotNull(Object v) {
        if (v == null)
            throw new NullPointerException();
    }
```
```java
    private void enqueue(E x) {
        // 断言当前线程持有锁且队列尾指针为空
        final Object[] items = this.items;
        items[putIndex] = x; // 将对象插入尾指针指向位置
        if (++putIndex == items.length) // 此处构建环形队列
            putIndex = 0;
        count++; // 任务数 +1
        notEmpty.signal(); // 归还锁对象，并唤醒阻塞的线程
    }
```
```java
    private E dequeue() {
        // 断言当前线程持有锁且队列头指针不为空
        final Object[] items = this.items;
        @SuppressWarnings("unchecked")
        E x = (E) items[takeIndex];
        items[takeIndex] = null; // 获取当前头指针对应对象，并将指针位置置空
        if (++takeIndex == items.length) // 此处构建环形队列
            takeIndex = 0;
        count--; // 任务数 -1
        if (itrs != null)
            itrs.elementDequeued();
        notFull.signal(); // 归还锁对象，并唤醒阻塞的线程
        return x;
    }
```
+ #### 构造方法
  + `ArrayBlockingQueue(int capacity)`：默认非公平可重入锁实现
  + `ArrayBlockingQueue(int capacity, boolean fair)`：可通过fair参数选择是否使用公平锁
  + `ArrayBlockingQueue(int capacity, boolean fair,Collection<? extends E> c)`：构造时添加集合中的对象到队列中
```java
    /**
    *  capacity:队列容量
    *  默认非公平锁
    */
    public ArrayBlockingQueue(int capacity) {
        this(capacity, false);
    }
```
```java
    /**
    *  capacity:队列容量    fair：是否为公平加锁
    */
    public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(fair); // 获取可重入锁（fair为true表示公平锁）
        notEmpty = lock.newCondition(); //从锁对象获取读阻塞的线程通信对象
        notFull =  lock.newCondition(); // 从锁对象获取写阻塞的线程通信对象
    }
```
```java
    /**
    *  capacity：队列容量    fair：是否为公平加锁    c：将集合中的元素放入队列
    */
    public ArrayBlockingQueue(int capacity, boolean fair,
                              Collection<? extends E> c) {
        this(capacity, fair);// 调用构造方法

        final ReentrantLock lock = this.lock;
        lock.lock(); // 加锁仅为了可见性，不为互斥性
        try {
            int i = 0;
            try {
                for (E e : c) { // 将集合元素写入队列
                    checkNotNull(e);
                    items[i++] = e;
                }
            } catch (ArrayIndexOutOfBoundsException ex) {
                throw new IllegalArgumentException();
            }
            count = i; // 初始化元素数量为i
            putIndex = (i == capacity) ? 0 : i; //初始化队列尾指针
        } finally {
            lock.unlock(); //解锁
        }
    }
```
+ #### 入队列方法 
  + `put(E e)`：在阻塞时触发wait使线程等待，适用于并发量较小的情形
  + `offer(E e)`：若队列满直接false，适用于并发量极大的情形
  + `offer(E e, long timeout, TimeUnit unit)`：会在超时后直接false，适用于并发量较大的情形
```java
    public void put(E e) throws InterruptedException {
        checkNotNull(e); // 检查 e 不为 null，若为 null 报空指针异常
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly(); // 可中断加锁
        try {
            while (count == items.length)
                notFull.await(); // 若队列满，线程 wait()
            enqueue(e); // 若队列不满，放入对象，并唤醒读阻塞的线程
        } finally {
            lock.unlock();
        }
    }
```
```java
    public boolean offer(E e) {
        checkNotNull(e); // 确保 e 不为 null
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            if (count == items.length) //若队列满直接放弃 false
                return false;
            else {
                enqueue(e); // 队列不满直接插入
                return true;
            }
        } finally {
            lock.unlock();
        }
    }
```
```java
    public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {

        checkNotNull(e);
        long nanos = unit.toNanos(timeout); // 获取 nano 等待时间
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == items.length) {
                if (nanos <= 0)
                    return false;
                nanos = notFull.awaitNanos(nanos); // 定时等待
            }
            enqueue(e);
            return true;
        } finally {
            lock.unlock();
        }
    }
```
+ #### 出队列方法
  + `take()`：会在阻塞时触发wait使线程等待，适用于并发量较小的情形
  + `poll()`：若队列空直接false，适用于并发量极大的情形
  + `poll(long timeout, TimeUnit unit)`：会在超时后直接false，适用于并发量较大的情形
```java
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly(); // 可中断加锁
        try {
            while (count == 0) //若队列空，线程 wait()
                notEmpty.await(); 
            return dequeue(); // 若队列不空，取出元素，并唤醒写阻塞的线程（可能当前取出后队列才不满）
        } finally {
            lock.unlock();
        }
    }
```
```java
    public E poll() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return (count == 0) ? null : dequeue(); // 取出队列头元素，若队列为空返回 null
        } finally {
            lock.unlock();
        }
    }
```
```java
    public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        long nanos = unit.toNanos(timeout); // 获取 nano 时间
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0) {
                if (nanos <= 0)
                    return null;
                nanos = notEmpty.awaitNanos(nanos); // 定时等待，若 nanos 时间耗尽，则返回 null 
            }
            return dequeue(); // 若队列不空，取出元素，并唤醒写阻塞的线程（可能当前取出后队列才不满）
        } finally {
            lock.unlock();
        }
    }
```

