---
title: 自己动手写把”锁”之---锁的作用
categories:
  - 技术
  - Java
tags:
  - synchronized
  - 同步
  - 多线程
date: 2017-12-31 15:38:42
---
### 前序
这是一个系列文章，前边几篇比较基础，主要为了后续做准备。熟悉的朋友可以直接跳过看后续的文章。

本主题很重要，学完这个系列，你将会对Java并包有一个透彻的原理性的认识。线程池技术、阻塞队列、信号量、原子性操作等等所用的基础技术都会在这系列的文章中讲到。大家可以提前学习下CountDownLatch的使用，在学完这系列文章后，我将会布置一个作业：自己动手实现一个CountDownLatch。
<!--more-->
### 正文
都知道，现在处理器的核数越来越多，为充分利用其计算资源，服务端编程通常会用上多线程技术。利用多线程技术可以同时进行计算任务，从而提高的服务的并发度。

但是，当多线程对同一块内存资源进行操作时，如果不对线程进行“排队”，其结果将是混乱不堪预测额。

这里说的“排队”就是通常说的线程同步。当一个线程在操作这个资源时，后来的线程等待上一个线程操作完才能开始。

举个代码栗子：

```
public class TestMyLock {
    private static  List<Integer> list = new ArrayList<>();
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
        list.add(i);
    }
    private static void print(){
        Iterator<Integer> iterator = list.iterator();
        while (iterator.hasNext()){
            System.out.println(iterator.next());
        }
    }
}
```
以上我们创建了两个线程：t1和t2。t1循环一万次向list里边添加数字，t2遍历list并将内容打印到控制台。
执行后得到如下错误：

原因很简单，就是两个线程同时操作list而没有进行线程同步导致的报错。我们修改下代码再来看一下。
add和print方法修改为如下：

```
private synchronized static void add(int i){
    list.add(i);
}
private synchronized static void print(){
    Iterator<Integer> iterator = list.iterator();
    while (iterator.hasNext()){
        System.out.println(iterator.next());
    }
}
```
我们仅仅是在add和print方法前加入了synchronized修饰词，程序便可以正常执行了

synchronized关键字是Java内置的同步锁。两个线程会竞争对synchronized绑定的同步对象加锁，加锁失败的线程会阻塞等待加锁成功的线程执行完毕。由于add和print方法都是静态方法，这里synchronized绑定的同步对象就是TestMyLock.class。


