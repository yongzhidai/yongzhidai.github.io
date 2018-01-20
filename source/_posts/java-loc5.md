---
title: 自己动手写把”锁”之---终极篇
categories:
  - 技术
  - Java
tags:
  - 锁
date: 2018-01-10 16:05:41
---
锁是整个Java并发包的实现基础，通过学习本系列文章，将对你理解Java并发包的本质有很大的帮助。
<!--more-->

前边几篇中，我已经把实现锁用到的技术，进行了一一讲述。这其中有原子性、内存模型、LockSupport还有CAS，掌握了这些技术，即使没有本篇，你也完全有能力自己写一把锁出来。但为了本系列的完整性，我在这里还是把最后这一篇补上。

先说一下锁的运行流程：多个线程抢占同一把锁，只有一个线程能抢占成功，抢占成功的线程继续执行下边的逻辑，抢占失败的线程进入阻塞等待。抢占成功的线程执行完毕后，释放锁，并从等待的线程中挑一个唤醒，让它继续竞争锁。

转变成程序实现：我们首先定一个state变量，state=0表示未被加锁，state=1表示被加锁。多个线程在抢占锁时，竞争将state变量从0修改为1，修改成功的线程则加锁成功。state从0修改为1的过程，这里使用cas操作，以保证只有一个线程加锁成功，同时state需要用volatile修饰，已解决线程可见的问题。加锁成功的线程执行完业务逻辑后，将state从1修改回0，同时从等待的线程中选择一个线程唤醒。所以加锁失败的线程，在加锁失败时需要将自己放到一个集合中，以等待被唤醒。这个集合需要支持多线程并发安全，在这里我通过一个链表来实现，通过CAS操作来实现并发安全。

把思路说清楚了，咱们看下代码实现。

首先咱们实现一个ThreadList，这是一个链表结合，用来存放等待的处于等待唤醒的线程：

```
public class ThreadList{
    private volatile Node head = null;
    private static  long headOffset;
    private static Unsafe unsafe;
    static {
        try {
            Constructor<Unsafe> constructor = Unsafe.class.getDeclaredConstructor(new Class<?>[0]);
            constructor.setAccessible(true);
            unsafe = constructor.newInstance(new Object[0]);
            headOffset = unsafe.objectFieldOffset(ThreadList.class.getDeclaredField("head"));
        }catch (Exception e){
        }
    }
    /**
     *
     * @param thread
     * @return 是否只有当前一个线程在等待
     */
    public boolean insert(Thread thread){
        Node node = new Node(thread);
        for(;;){
            Node first = getHead();
            node.setNext(first);
            if(unsafe.compareAndSwapObject(this, headOffset,first,node)){
                return first==null?true:false;
            }
        }
    }
    public Thread pop(){
        Node first = null;
        for(;;){
            first = getHead();
            Node next = null;
            if(first!=null){
                next = first.getNext();
            }
            if(unsafe.compareAndSwapObject(this, headOffset,first,next)){
                break;
            }
        }
        return first==null?null:first.getThread();
    }
    private Node getHead(){
        return this.head;
    }
    private static class Node{
        volatile Node next;
        volatile Thread thread;
        public Node(Thread thread){
            this.thread = thread;
        }
        public void setNext(Node next){
            this.next = next;
        }
        public Node getNext(){
            return next;
        }
        public Thread getThread(){
            return this.thread;
        }
    }
}
```

加锁失败的线程，调用insert方法将自己放入这个集合中，insert方法里将线程封装到Node中，然后使用cas操作将node添加到列表的头部。同样为了线程可见的问题，Node里的thread和next都用volatile修饰。
加锁成功的线程，调用pop方法获得一个线程，进行唤醒，这里边同样使用了cas操作来保证线程安全。

接下来在看看锁的实现：

```
public class MyLock {
    private volatile int state = 0;
    private ThreadList threadList = new ThreadList();
    private static  long stateOffset;
    private static Unsafe unsafe;
    static {
       try {
           Constructor<Unsafe> constructor = Unsafe.class.getDeclaredConstructor(new Class<?>[0]);
           constructor.setAccessible(true);
           unsafe = constructor.newInstance(new Object[0]);
           stateOffset = unsafe.objectFieldOffset(MyLock.class.getDeclaredField("state"));
       }catch (Exception e){
       }

    }
    public void lock(){
        if(compareAndSetState(0,1)){
        }else{
            addNodeAndWait();
        }
    }
    public void unLock(){
        compareAndSetState(1,0);
        Thread thread = threadList.pop();
        if(thread != null){
            LockSupport.unpark(thread);
        }
    }
    private void addNodeAndWait(){
        //如果当前只有一个等待线程时，重新获取一下锁，防止永远不被唤醒。
        boolean isOnlyOne = threadList.insert(Thread.currentThread());
        if(isOnlyOne && compareAndSetState(0,1)){
            return;
        }
        LockSupport.park(this);//线程被挂起
        if(compareAndSetState(0,1)){//线程被唤醒后继续竞争锁
            return;
        }else{
            addNodeAndWait();
        }
    }
    private boolean compareAndSetState(int expect,int update){
        return unsafe.compareAndSwapInt(this,stateOffset,expect,update);
    }
}
```

线程调用lock方法进行加锁，cas将state从0修改1，修改成功则加锁成功，lock方法返回，否则调用addNodeAndWait方法将线程加入ThreadList队列，并使用LockSupport将线程挂起。(ThreadList的insert方法，返回一个boolean类型的值，用来处理一个特殊情况的，稍后再说。)

获得锁的线程执行完业务逻辑后，调用unLock方法释放锁，即通过cas操作将state修改回0，同时从ThreadList拿出一个等待线程，调用LockSupport的unpark方法，来将它唤醒。


将我们在《自己动手写把"锁"---锁的作用》的例子修改为如下，来测试下咱们的锁的效果：


```
public class TestMyLock {
    private static  List<Integer> list = new ArrayList<>();
    private static MyLock myLock = new MyLock();
    public static void main(String[] args){
        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                for(int i=0;i<10000;i++){
                    add(i);
                }
            }
        });
        Thread t2 = new Thread(new Runnable() {
            @Override
            public void run() {
                print();
            }
        });
        t1.start();
        t2.start();
    }
    private static void add(int i){
        myLock.lock();
        list.add(i);
        myLock.unLock();
    }
    private static void print(){
        myLock.lock();
        Iterator<Integer> iterator = list.iterator();
        while (iterator.hasNext()){
            System.out.println(iterator.next());
        }
        myLock.unLock();
    }
}
```

ok,正常运行了，不在报错。

到这里咱们的一个简单地锁已经实现了。接下来我再把上边的，一个没讲的细节说一下。即如下这段代码：

```
boolean isOnlyOne = threadList.insert(Thread.currentThread());
        if(isOnlyOne && compareAndSetState(0,1)){
            return;
        }
```

ThreadList的insert方法，在插入成功后，会判断当前链表中是否只有自己一个线程在等待，如果是则返回true。从而进入后边的if语句。这个逻辑的用意就是：如果只有自己一个线程在等待时，则试着通过cas操作重新获取锁，如果获取失败才进入阻塞等待。它是用来解决以下边界情况：

![image](java-loc5/imag1.png)

在只有线程A和线程B两个线程的时候，如果没有以上判断逻辑，线程B将有可能会永远处于阻塞不被唤醒。