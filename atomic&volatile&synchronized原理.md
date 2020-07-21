# atomic&volatile&synchronized原理

## 多线程共享变量

### 原子性

一个操作要么全都执行成功,要么全都失败.

### 可见性

多个线程操作一个变量,其中一个线程修改后,对其他线程可见.

### 有序性

程序的执行,按照代码的顺序执行.jvm会对一些不满足happen-before原则的代码进行指令重排.

#### happen-before

- 程序次序规则：一个线程内，按照代码顺序，书写在前面的操作先行发生于书写在后面的操作
- 锁定规则：一个unLock操作先行发生于后面对同一个锁额lock操作
- volatile变量规则：对一个变量的写操作先行发生于后面对这个变量的读操作
- 传递规则：如果操作A先行发生于操作B，而操作B又先行发生于操作C，则可以得出操作A先行发生于操作C
- 线程启动规则：Thread对象的start()方法先行发生于此线程的每个一个动作
- 线程中断规则：对线程interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生
- 线程终结规则：线程中所有的操作都先行发生于线程的终止检测，我们可以通过Thread.join()方法结束、Thread.isAlive()的返回值手段检测到线程已经终止执行
- 对象终结规则：一个对象的初始化完成先行发生于他的finalize()方法的开始

## Atomic

### 简介

在多线程场景下,在对于共享变量操作时,通常会使用synchronize关键字或者lock进行加锁,但是这种操作会涉及到用户态和内核态的转换,极大的占用资源.因此,后面出现了轻量级锁:atomic.

atomic是什么?它是JDK1.5之后,为多线程操作共享变量提供原子性的一系列数据结构.主要是使用Unsafe类去完成一系列核心操作.这个核心操作就是CAS(compare and swap).

#### Unsafe

Unsafe类使java像C语言一样操作内存空间的能力,也带来了指针的问题.它通过调用底层C语言,生成一个CPU指令,因此是原子性的.

#### CAS

由3个操作数组成,变量内存值V,变量期望值E,变量更新后的值U.在对V进行修改时,先判断E是否一致,如果一致则修改为U.不一致的情况下会堵塞进行自旋操作.直到修改成功.

### 组成

#### 基本类型

- AtomicBoolean：布尔型
- AtomicInteger：整型
- AtomicLong：长整型

#### 数组

- AtomicIntegerArray：数组里的整型
- AtomicLongArray：数组里的长整型
- AtomicReferenceArray：数组里的引用类型

#### 引用类型

- AtomicReference：引用类型
- AtomicStampedReference：带有版本号的引用类型
- AtomicMarkableReference：带有标记位的引用类型

#### 对象的属性

- AtomicIntegerFieldUpdater：对象的属性是整型
- AtomicLongFieldUpdater：对象的属性是长整型
- AtomicReferenceFieldUpdater：对象的属性是引用类型

#### JDK8新增

atomic等类都是使用的**自旋+CAS**来进行原子类操作.同一时间只能有一个线程进行修改,其它的需要自旋等待,在并发很高的情况下,自旋会极大,对CPU资源浪费也极大.在JDK8后,引入了新的适用于高并发下的原子性操作.

DoubleAccumulator、LongAccumulator、DoubleAdder、LongAdder

是对AtomicLong等类的改进。比如LongAccumulator与LongAdder在高并发环境下比AtomicLong更高效。

那这些新增的类又是如何对atomic进行的升级优化呢?

原理:这些类都继承了一个Striped64类.这个类具有**分段CAS操作**的能力.

##### Striped64

```java
  //当前CPU的数量,决定了cells数组的最大容量.
    static final int NCPU = Runtime.getRuntime().availableProcessors();
	//cells数组,其元素是操作的值,下标是线程的hash值.每次扩容为2的幂次方.
    transient volatile Cell[] cells;
//基础值,类似于atomic类中cas的value值.
    transient volatile long base;
//当cell创建或者扩容时,进行cas加锁.
    transient volatile int cellsBusy;
```

在Atomic类中,只有一个value值,每次通过cas竞争获取value,高并发下自旋,资源消耗严重.在Striped64中,有2个值,一个是base,没有竞争的情况下,base的值和value作用一样.在竞争时,cells就会使用,每次扩容2次幂,把竞争的线程进行hash计算.然后hash值为数组下标放进cells数组.即如果有2个线程竞争,则这2个线程分别操作cells数组里面的不同的元素.最后在进行获取值的时候,将base和cells数组里面的值进行累加.

### CAS的ABA问题

#### 场景

1. 假设有3个线程ABC.即将要对V修改.V初始值为100.
2. AB线程拿到V,Ab内存值和预期值都是100.
3. 此时线程A对比后将V+100,此时V=200.
4. 线程C拿到V,C的内存中和预期值是200.线程C将V-100.此时V=100.
5. 线程B通过竞争拿到了V,发现预期值是100,则进行了后续操作.

这种情况下,线程B无法感知到在它执行操作之前,其实V值已经进行了一轮变化.比如,你刚买的牙刷,被别人用了又放回原位,而你却无法得知这种情况.

#### 解决

添加版本号.为变量增加一个版本号,每次进行修改,版本号跟着变化.

atomic已经封装好了相应的方法:AtomicStampedReference和AtomicMarkableReference.在这里面维护了一个Pair对象,cas时比较两个pair值.

```java
// Pair对象
    private static class Pair<T> {
        final T reference;
        final int stamp;
        private Pair(T reference, int stamp) {
            this.reference = reference;
            this.stamp = stamp;
        }
        static <T> Pair<T> of(T reference, int stamp) {
            return new Pair<T>(reference, stamp);
        }
    }

    private volatile Pair<V> pair;

    // 比较的是Pari对象的引用和stamp.
    public boolean compareAndSet(V   expectedReference,
                                 V   newReference,
                                 int expectedStamp,
                                 int newStamp) {
        Pair<V> current = pair;
        return
            expectedReference == current.reference &&
            expectedStamp == current.stamp &&
            ((newReference == current.reference &&
              newStamp == current.stamp) ||
             casPair(current, Pair.of(newReference, newStamp)));
    }
```

## Volatile

### CPU与内存模型

计算机在执行程序时,每条指令都是在CPU中执行,但是运行时的数据都存在内存中.就是说需要CPU不断的从内存中拿取数据,执行操作,然后返回给内存.

问题是,CPU执行的速度比从内存中存取数据快得多,如果每次都等存取到内存完成,会极大的降低执行效率.因此,在内存和CPU之间就产生了高速缓存.在程序运行时,将内存中的数据放到高速缓存,这样CPU执行指令时,从高度缓存中拿数据,执行完后放进高速缓存,然后在更新到内存.这样效率有了极大提高.

但是在多线程条件下,就出现了问题:2个线程同时执行i=i+1,2个线程同时拿到数据,同时执行,放进内存,然后最后i还是只加了1.

为了解决这问题,出现了缓存一致性协议,即,当CPU写数据时,如果发现是共享变量,则写完后,更新到主存,同时会通知所有CPU,告知他们拿到的共享变量已经失效了,在使用时,需要重新从内存拿.

![image-20200713235405630](atomic&volatile&synchronized%E5%8E%9F%E7%90%86/image-20200713235405630.png)

### java内存模型

在java内存中,所有的数据变量存放在主存中(相当于上面的物理内存),每个线程都有各自的工作内存(类似于高速缓存).每个线程的变量都在各自的工作内存中执行,不能再主存中执行,各自的工作内存不能访问其他的工作内存.

为了实现缓存一致性的功能,java添加了Volatile关键字.

#### 保证可见性

规定,用Volatile修饰的变量,在工作内存中进行修改后,会立即刷新到主存,并且通知其它工作内存此变量需要重新从主存获取.从而保证了变量的线程间可见性.

#### 保证有序性

禁止指令重排.根据happen-before原则,用volatile关键字修饰的关键字,多线程并发操作时,写操作肯定优先于读操作.

```java
//线程1
boolean stop = false;
while(!stop){
    doSomething();
}
 
//线程2
stop = true;
```

比如以上代码,线程1会进入循环中.线程2的修改在大多数情况下,不会让循环终止.

但是如果stop在用volatile关键字修饰后,线程2对stop修改后,会将此次修改直接刷入主存.同时通知线程1中的变量stop无效,需要重新获取.

