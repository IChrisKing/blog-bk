---
title: Java并发
date: 2016-09-19 09:26:34
category:
	- Java
	- Android开发工程师
tags:
	- Android开发工程师
	- Android
	- Java
description: "Java并发相关的内容。"
---

## Thread与Runable
继承Thread类和实现Runable接口都可以实现多线程。两种方式都要通过重写run()方法来定义线程的行为，推荐使用后者，因为Java中的继承是单继承，一个类有一个父类，如果继承了Thread类就无法再继承其他类了，显然使用Runnable接口更为灵活。

实现Runnable接口相比继承Thread类有如下优势：

1. 可以避免由于Java的单继承特性而带来的局限
2. 增强程序的健壮性，代码能够被多个程序共享，代码与数据是独立的
3. 适合多个相同程序代码的线程区处理同一资源的情况

[Thread和Runnable实现多线程的区别](https://github.com/GeniusVJR/LearningNotes/blob/master/Part2/JavaConcurrent/Thread%E5%92%8CRunnable%E5%AE%9E%E7%8E%B0%E5%A4%9A%E7%BA%BF%E7%A8%8B%E7%9A%84%E5%8C%BA%E5%88%AB.md)

## Callable接口
Java 5以后创建线程还有第三种方式：实现Callable接口，该接口中的call方法可以在线程执行结束时产生一个返回值，代码如下所示：
```
class MyTask implements Callable<Integer> {  
    private int upperBounds;  

    public MyTask(int upperBounds) {  
        this.upperBounds = upperBounds;  
    }  

    @Override  
    public Integer call() throws Exception {  
        int sum = 0;   
        for(int i = 1; i <= upperBounds; i++) {  
            sum += i;  
        }  
        return sum;  
    }  

}  

public class Test {  

    public static void main(String[] args) throws Exception {  
        List<Future<Integer>> list = new ArrayList<>();  
        ExecutorService service = Executors.newFixedThreadPool(10);  
        for(int i = 0; i < 10; i++) {  
            list.add(service.submit(new MyTask((int) (Math.random() * 100))));  
        }  

        int sum = 0;  
        for(Future<Integer> future : list) {  
            while(!future.isDone()) ;  
            sum += future.get();  
        }  

        System.out.println(sum);  
    }  
}  
```

## 生产者消费者的例子
```

public class ProducerConsumerTest {

    public static void main(String[] args) {
        PublicResource resource = new PublicResource();
        new Thread(new ProducerThread(resource)).start();
        new Thread(new ConsumerThread(resource)).start();
        new Thread(new ProducerThread(resource)).start();
        new Thread(new ConsumerThread(resource)).start();
        new Thread(new ProducerThread(resource)).start();
        new Thread(new ConsumerThread(resource)).start();
    }
}
package 生产者消费者;
/**
 * 生产者线程，负责生产公共资源
 * @author dream
 *
 */
public class ProducerThread implements Runnable{

    private PublicResource resource;


    public ProducerThread(PublicResource resource) {
        this.resource = resource;
    }


    @Override
    public void run() {
        while (true) {
            try {
                Thread.sleep((long) (Math.random() * 1000));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            resource.increase();
        }
    }


}
package 生产者消费者;

/**
 * 消费者线程，负责消费公共资源
 * @author dream
 *
 */
public class ConsumerThread implements Runnable{

    private PublicResource resource;


    public ConsumerThread(PublicResource resource) {
        this.resource = resource;
    }


    @Override
    public void run() {
        while (true) {
            try {
                Thread.sleep((long) (Math.random() * 1000));
            } catch (InterruptedException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }
            resource.decrease();
        }

    }


}
package 生产者消费者;

/**
 * 公共资源类
 * @author dream
 *
 */
public class PublicResource {

    private int number = 0;
    private int size = 10;

    /**
     * 增加公共资源
     */
    public synchronized void increase()
    {
        while (number >= size) {
            try {
                wait();
            } catch (InterruptedException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }
        }
        number++;
        System.out.println("生产了1个，总共有" + number);
        notifyAll();
    }


    /**
     * 减少公共资源
     */
    public synchronized void decrease()
    {
        while (number <= 0) {
            try {
                wait();
            } catch (InterruptedException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }
        }
        number--;
        System.out.println("消费了1个，总共有" + number);
        notifyAll();
    }
}
```

## 线程中断
[线程中断](https://github.com/GeniusVJR/LearningNotes/blob/master/Part2/JavaConcurrent/%E7%BA%BF%E7%A8%8B%E4%B8%AD%E6%96%AD.md)

## 线程同步

### synchronized
[synchronized](https://github.com/GeniusVJR/LearningNotes/blob/master/Part2/JavaConcurrent/Synchronized.md)

[多线程环境中安全使用集合API](https://github.com/GeniusVJR/LearningNotes/blob/master/Part2/JavaConcurrent/%E5%A4%9A%E7%BA%BF%E7%A8%8B%E7%8E%AF%E5%A2%83%E4%B8%AD%E5%AE%89%E5%85%A8%E4%BD%BF%E7%94%A8%E9%9B%86%E5%90%88API.md)

### 锁
[死锁](https://github.com/GeniusVJR/LearningNotes/blob/master/Part2/JavaConcurrent/%E6%AD%BB%E9%94%81.md)

[可重入内置锁](https://github.com/GeniusVJR/LearningNotes/blob/master/Part2/JavaConcurrent/%E5%8F%AF%E9%87%8D%E5%85%A5%E5%86%85%E7%BD%AE%E9%94%81.md)
