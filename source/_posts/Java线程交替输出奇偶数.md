---
title: Java线程交替输出奇偶数
date: 2018-06-28 22:07:32
categories: Java 
tags: [Java]
---


## 直接上代码

```java


import java.util.concurrent.locks.LockSupport;

/**
 * 类名称是随便起的
 *
 * 这个类主要展示了两种交替输出奇数和偶数的实现
 *
 * 主要的实现思路都是直接启动 2 个线程分别输出奇数和偶数， 然后在通过 Java 的线程调度控制让他们每秒交替输出一次
 *
 * 第一种控制思路是利用 Object.wait() 和 Object.notify()
 * 第二种是使用在主线程中使用 LockSupport.park() 和 LockSupport.unpark()
 */
public class Evaluate{
    static class WaitNotify implements Runnable {
        private Object lock; //定义 锁 对象
        private int start; //线程的初始数字

        WaitNotify(int init, Object lockObj) {
            lock = lockObj;
            start = init;
        }

        public void run() {
            while(true) {
                synchronized (lock) {
                    System.out.println(start);
                    try {
                        Thread.sleep(1000);
                        lock.notify(); //这里 notify() 写在前面是因为第二次循环开始需要先唤醒线程
                        lock.wait(); // wait() 方法会释放锁， 并且让当前线程休眠
                    } catch (InterruptedException e) {
                        e. printStackTrace();
                    }
                }
                start += 2;
            }
        }
    }

    static class LockSupportParkUnpark implements Runnable {
        private int start;

        LockSupportParkUnpark(int init) {
            start = init;
        }

        public void run() {
            while(true) {
                //默认调用 park() 会阻塞线程
                LockSupport.park();
                System.out.println(start);
                start += 2;
            }
        }
    }

    public static void main(String[] args) {
        //这种实现方式主线程没有参与， 依靠子线程自己进行调度
        Object lock = new Object();
        //两个线程使用同一个锁
        Thread t1 = new Thread(new WaitNotify(0, lock));
        Thread t2 = new Thread(new WaitNotify(1, lock));
        t1.start();
        t2.start();

        //这种实现依靠主线程来进行调度， 但是没有使用锁
        Thread t3 = new Thread(new LockSupportParkUnpark(0));
        Thread t4 = new Thread(new LockSupportParkUnpark(1));
        t3.start();
        t4.start();

        //这个调度过程很简单
        for(int i = 0; ; i++) {
            if(i % 2 == 0 ) {
                LockSupport.unpark(t3);
            } else {
                LockSupport.unpark(t4);
            }
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}


```


<!--more-->