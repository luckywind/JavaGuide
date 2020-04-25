# monitor的概念

管程，英文是 Monitor，也常被翻译为“监视器”，monitor 不管是翻译为“管程”还是“监视器”，都是比较晦涩的，通过翻译后的中文，并无法对 monitor 达到一个直观的描述。
 在[《操作系统同步原语》](https://links.jianshu.com/go?to=https%3A%2F%2Fbeanlam.me%2Fsyncprimitive%2F) 这篇文章中，介绍了操作系统在面对  进程/线程 间同步的时候，所支持的一些同步原语，其中 semaphore 信号量 和 mutex 互斥量是最重要的同步原语。
 在使用基本的 mutex 进行并发控制时，

需要程序员非常小心地控制 mutex 的 down 和 up 操作，否则很容易引起死锁等问题。为了更容易地编写出正确的并发程序，所以在 mutex 和 semaphore 的基础上，提出了更高层次的同步原语 monitor，不过需要注意的是，<font color=red>操作系统本身并不支持 monitor 机制，实际上，monitor 是属于编程语言的范畴，当你想要使用 monitor 时，先了解一下语言本身是否支持 monitor 原语，例如 C 语言它就不支持 monitor，Java 语言支持 monitor。
 一般的 monitor 实现模式是编程语言在语法上提供语法糖，而如何实现 monitor 机制，则属于编译器的工作，Java 就是这么干的。</font>

monitor 的重要特点是，<font color=red>同一个时刻，只有一个 进程/线程 能进入 monitor 中定义的临界区，这使得 monitor 能够达到互斥的效果</font>。但仅仅有互斥的作用是不够的，无法进入 monitor 临界区的 进程/线程，它们应该被阻塞，并且在必要的时候会被唤醒。显然，monitor 作为一个同步工具，也应该提供这样的管理 进程/线程 状态的机制。想想我们为什么觉得 semaphore 和 mutex 在编程上容易出错，因为我们需要去亲自操作变量以及对 进程/线程 进行阻塞和唤醒。monitor 这个机制之所以被称为“更高级的原语”，那么它就不可避免地需要对外屏蔽掉这些机制，并且在内部实现这些机制，使得使用 monitor 的人看到的是一个简洁易用的接口。

# monitor 基本元素

monitor 机制需要几个元素来配合，分别是：

1. 临界区
2. monitor 对象及锁
3. 条件变量以及定义在 monitor 对象上的 wait，signal 操作。

<font color=red>使用 monitor 机制的目的主要是为了**互斥进入临界区**，为了做到能够阻塞无法进入临界区的 进程/线程，还需要一个 monitor object 来协助，这个 monitor object 内部会有相应的数据结构，例如列表，**来保存被阻塞的线程**；同时由于 monitor 机制本质上是基于 mutex 这种基本原语的，所以 monitor object 还必须维护一个**基于 mutex 的锁**。</font>
 此外，为了在适当的时候能够阻塞和唤醒 进程/线程，还需要引入一个条件变量，这个条件变量用来决定什么时候是“适当的时候”，这个条件可以来自程序代码的逻辑，也可以是在 monitor object 的内部，总而言之，程序员对条件变量的定义有很大的自主性。不过，**由于 monitor object 内部采用了数据结构来保存被阻塞的队列，因此它也必须对外提供两个 API 来让线程进入阻塞状态以及之后被唤醒，分别是 wait 和 notify**。

# Java 语言对 monitor 的支持

monitor 是操作系统提出来的一种高级原语，但其具体的实现模式，不同的编程语言都有可能不一样。以下以 Java 的 monitor 为例子，来讲解 monitor 在 Java 中的实现方式。

## 临界区的圈定

在 Java 中，可以采用 synchronized 关键字来修饰实例方法、类方法以及代码块，如下所示：



```java
/**
 * @author beanlam
 * @version 1.0
 * @date 2018/9/12
 */
public class Monitor {

    private Object ANOTHER_LOCK = new Object();

    private synchronized void fun1() {
    }

    public static synchronized void fun2() {
    }

    public void fun3() {
        synchronized (this) {
        }
    }

    public void fun4() {
        synchronized (ANOTHER_LOCK) {
        }
    }
}
```

被 synchronized 关键字修饰的方法、代码块，就是 monitor 机制的临界区。

## monitor object

可以发现，上述的 synchronized 关键字在使用的时候，往往需要指定一个对象与之关联，例如 synchronized(this)，或者 synchronized(ANOTHER_LOCK)，synchronized 如果修饰的是实例方法，那么其关联的对象实际上是 this，如果修饰的是类方法，那么其关联的对象是 this.class。总之，synchronzied 需要关联一个对象，而这个对象就是 monitor object。
 monitor 的机制中，monitor object 充当着维护 mutex以及定义 wait/signal API 来管理线程的阻塞和唤醒的角色。
 Java 语言中的 java.lang.Object 类，便是满足这个要求的对象，任何一个 Java 对象都可以作为 monitor 机制的 monitor object。
 Java 对象存储在内存中，分别分为三个部分，即对象头、实例数据和对齐填充，而在其对象头中，保存了锁标识；同时，java.lang.Object 类定义了 wait()，notify()，notifyAll() 方法，这些方法的具体实现，依赖于一个叫 ObjectMonitor 模式的实现，这是 JVM 内部基于 C++ 实现的一套机制，基本原理如下所示：

![image-20191218103013572](../../../../../../Library/Application Support/typora-user-images/image-20191218103013572.png)

MonitorObject.png

当一个线程需要获取 Object 的锁时，会被放入 EntrySet 中进行等待，如果该线程获取到了锁，成为当前锁的 owner。如果根据程序逻辑，一个已经获得了锁的线程缺少某些外部条件，而无法继续进行下去（例如生产者发现队列已满或者消费者发现队列为空），那么该线程可以通过调用 wait 方法将锁释放，进入 wait set 中阻塞进行等待，其它线程在这个时候有机会获得锁，去干其它的事情，从而使得之前不成立的外部条件成立，这样先前被阻塞的线程就可以重新进入 EntrySet 去竞争锁。这个外部条件在 monitor 机制中称为条件变量。

## synchronized 关键字

<font color=red>synchronized 关键字是 Java 在语法层面上，用来让开发者方便地进行多线程同步的重要工具。要进入一个 synchronized 方法修饰的方法或者代码块，会先获取与 synchronized 关键字绑定在一起的 Object 的对象锁(获取锁：成功操作其MarkWord)，这个锁也限定了其它线程无法进入与这个锁相关的其它 synchronized 代码区域。</font>

网上很多文章以及资料，在分析 synchronized 的原理时，基本上都会说 synchronized 是基于 monitor 机制实现的，但很少有文章说清楚，都是模糊带过。
 参照前面提到的 Monitor 的几个基本元素，如果 synchronized 是基于 monitor 机制实现的，那么对应的元素分别是什么？
 它必须要有临界区，这里的**临界区我们可以认为是对对象头 mutex 的 P 或者 V 操作，这是个临界区**
 那 monitor object 对应哪个呢？mutex？总之无法找到真正的 monitor object。
 所以我认为“synchronized 是基于 monitor 机制实现的”这样的说法是不正确的，是模棱两可的。
 **Java 提供的 monitor 机制，其实是 Object，synchronized 等元素合作形成的，甚至说外部的条件变量也是个组成部分。**JVM 底层的 ObjectMonitor **只是用来辅助实现** monitor 机制的一种常用模式，但大多数文章把 ObjectMonitor 直接当成了 monitor 机制。
 我觉得应该这么理解：**Java 对 monitor 的支持，是以机制的粒度提供给开发者使用的，也就是说，开发者要结合使用 synchronized 关键字，以及 Object 的 wait / notify 等元素**，才能说自己利用 monitor 的机制去解决了一个生产者消费者的问题。

