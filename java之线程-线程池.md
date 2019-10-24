---
title: 'java之线程,线程池'
date: 2019-03-03 22:34:55
tags: 
- 线程
- 并发
categories: 
- java基础
- java之线程,线程池
---
## 1.线程与进程
&nbsp;&nbsp; 进程是系统进行资源分配和调度的独立单位,一个进程下可以包括多个线程.一个进程在执行时,总需要多个子任务去执行,这就需要多线程.最大的利用CPU的空闲时间去执行更多的任务.
## 2 .创建线程
### 2.1继承Thread类

```
public class StringUtil extends Thread{
	
		@Override
		public void run() {
			System.out.println(this.getName());
			super.run();
		}
		public static void main(String[] args) {
			for(int i=0;i<100;i++) {
				new StringUtil().start();
			}
		}
	}
```

### 2.2 实现runnable接口
```
	public class Aa implements Runnable{
	
		@Override
		public void run() {
			System.out.println(Thread.currentThread().getName());
		}
		public static void main(String[] args) {
			for(int i=0;i<100;i++) {
				new Thread(new Aa()).start();
			}
		}
	}
```
### 2.3 两种方法对比
1. 看源码
```
public
class Thread implements Runnable {
    /* Make sure registerNatives is the first thing <clinit> does. */
    private static native void registerNatives();
    static {
        registerNatives();
    }

    private volatile String name;
    private int priority;

    /* Whether or not the thread is a daemon thread. */
    private boolean daemon = false;

    /* Fields reserved for exclusive use by the JVM */
    private boolean stillborn = false;
    private long eetop;
   
```

在Thread源码中,Thread是Runnable的实现.因此可以得出,
	a) 我们如果使用继承Thread类,系统已经给我们经过继承,封装好了.可以直接使用.
	b) 我们如果直接使用实现Runnable接口方式,需要将我们的类放进Thread方法中,去生成一个Thread类,从而具有Thread的所有功能.
2. 由于两种方法基本上是一样的,但是java是单继承的,为了不会影响到我们写的类去继承其他类,所以推荐使用第二种方法.
3.  从demo可知,run()方法是线程中我们需要实现的业务.start()方法是开启一个线程.
4. start()方法源码:
```
 public synchronized void start() {
        /**
         * This method is not invoked for the main method thread or "system"
         * group threads created/set up by the VM. Any new functionality added
         * to this method in the future may have to also be added to the VM.
         *
         * A zero status value corresponds to state "NEW".
         */
        if (threadStatus != 0)
            throw new IllegalThreadStateException();

        /* Notify the group that this thread is about to be started
         * so that it can be added to the group's list of threads
         * and the group's unstarted count can be decremented. */
        group.add(this);
        
        boolean started = false;
        try {
           //开启线程
            start0();
            started = true;
        } finally {
            try {
                if (!started) {
                    group.threadStartFailed(this);
                }
            } catch (Throwable ignore) {
                /* do nothing. If start0 threw a Throwable then
                  it will be passed up the call stack */
            }
        }
    }
```

&nbsp;&nbsp;可知真正开启一个线程的方法是start0(),并且在创建的时候,用synchronized关键字,进行锁住;同时,可知为了进行线程的管理,将创建的线程加入了线程组.
5. Thread类
	线程的方法（Method）、属性（Property）
		1）优先级（priority）
		&nbsp;&nbsp;每个类都有自己的优先级，一般property用1-10的整数表示，默认优先级是5，优先级最高是10；优先级高的线程并不一定比优先级低的线程执行的机会高，只是执行的机率高；默认一个线程的优先级和创建他的线程优先级相同；
		2）Thread.sleep()/sleep(long millis)
		&nbsp;&nbsp;当前线程睡眠/millis的时间（millis指定睡眠时间是其最小的不执行时间，因为sleep(millis)休眠到达后，无法保证会被JVM立即调度）；sleep()是一个静态方法(static method) ，所以他不会停止其他的线程也处于休眠状态；线程sleep()时不会失去拥有的对象锁。 作用：保持对象锁，让出CPU，调用目的是不让当前线程独自霸占该进程所获取的CPU资源，以留一定的时间给其他线程执行的机会；
		3）Thread.yield()
		 &nbsp;&nbsp; 让出CPU的使用权，给其他线程执行机会、让同等优先权的线程运行（但并不保证当前线程会被JVM再次调度、使该线程重新进入Running状态），如果没有同等优先权的线程，那么yield()方法将不会起作用。
		4）thread.join()
		 使用该方法的线程会在此之间执行完毕后再往下继续执行。
		5）object.wait()
		  当一个线程执行到wait()方法时，他就进入到一个和该对象相关的等待池(Waiting Pool)中，同时失去了对象的机锁—暂时的，wait后还要返还对象锁。当前线程必须拥有当前对象的锁，如果当前线程不是此锁的拥有者，会抛出IllegalMonitorStateException异常,所以wait()必须在synchronized block中调用。
		6）object.notify()/notifyAll()
		  唤醒在当前对象等待池中等待的第一个线程/所有线程。notify()/notifyAll()也必须拥有相同对象
		锁，否则也会抛出IllegalMonitorStateException异常。
		7）Synchronizing Block
		 Synchronized Block/方法控制对类成员变量的访问；Java中的每一个对象都有唯一的一个内置的锁，每个Synchronized Block/方法只有持有调用该方法被锁定对象的锁才可以访问，否则所属线程阻塞；机锁具有独占性、一旦被一个Thread持有，其他的Thread就不能再拥有（不能访问其他同步方法），方法一旦执行，就独占该锁，直到从该方法返回时才将锁释放，此后被阻塞的线程方能获得该锁，重新进入可执行状态。
	注: 
	线程组是一个进程下所有的线程管理类,线程以树形的结构存储
	线程池是为了管理线程的生命周期，复用线程，减少创建销毁线程的开销
	
6. 继续查看:
源码:
	```
	 private native void start0();
	```
	可知最终是调用的原生C/C++方法去开启的一个线程.
## 3. 线程池: ThreadPoolExecutor
&nbsp;&nbsp;ThreadPoolExecutor是jdk1.5之后package java.util.concurrent;官方提供的线程池创建类.主要负责线程的调度,任务的执行,线程池的管理等.
	构造方法:
	
```
	    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }
```
|参数|说明
|--|--|
|corePoolSize|核心线程池大小|
|maximumPoolSize	|最大线程池大小|
|keepAliveTime	|线程池中超过corePoolSize数目的空闲线程最大存活时间；可以allowCoreThreadTimeOut(true)使得核心线程有效时间|
|unit	|keepAliveTime时间单位|
|workQueue	|阻塞任务队列|
|threadFactory	|线程工厂|
|handler|	当提交任务数超过maxmumPoolSize+workQueue之和时，任务会交给RejectedExecutionHandler来处理|
重点讲解： 
其中比较容易让人误解的是：corePoolSize，maximumPoolSize，workQueue之间关系。 
1. 当线程池小于corePoolSize时，新提交任务将创建一个新线程执行任务，即使此时线程池中存在空闲线程。 
2. 当线程池达到corePoolSize时，新提交任务将被放入workQueue中，等待线程池中任务调度执行 
3. 当workQueue已满，且maximumPoolSize>corePoolSize时，新提交任务会创建新线程执行任务 
4. 当提交任务数超过maximumPoolSize时，新提交任务由RejectedExecutionHandler处理 
5. 当线程池中超过corePoolSize线程，空闲时间达到keepAliveTime时，关闭空闲线程 
6. 当设置allowCoreThreadTimeOut(true)时，线程池中corePoolSize线程空闲时间达到keepAliveTime也将关闭

线程池图解:
	
	
	
	屏幕剪辑的捕获时间: 2019/2/26 22:23
其中,官方给出了工具类Executors,package java.util.concurrent;,用来快速创建一些线程池方案.
1. 构造一个固定线程数目的线程池，配置的corePoolSize与maximumPoolSize大小相同，同时使用了一个无界LinkedBlockingQueue存放阻塞任务，因此多余的任务将存在再阻塞队列，不会由
	RejectedExecutionHandler处理 
	```
	public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
   ```
2. 构造一个缓冲功能的线程池，配置corePoolSize=0，maximumPoolSize=Integer.MAX_VALUE，keepAliveTime=60s,以及一个无容量的阻塞队列 SynchronousQueue，因此任务提交之后，将会创建新的
	线程执行；线程空闲超过60s将会销毁 
	```
	public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
   ```
3. 构造一个只支持一个线程的线程池，配置corePoolSize=maximumPoolSize=1，无界阻塞队列
	LinkedBlockingQueue；保证任务由一个线程串行执行 
	```
	public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
   ```
4. 构造有定时功能的线程池，配置corePoolSize，无界延迟阻塞队列DelayedWorkQueue；有意思的是：
	maximumPoolSize=Integer.MAX_VALUE，由于DelayedWorkQueue是无界队列，所以这个值是没有意义的 
```
	public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }
	public static ScheduledExecutorService newScheduledThreadPool(
            int corePoolSize, ThreadFactory threadFactory) {
        return new ScheduledThreadPoolExecutor(corePoolSize, threadFactory);
    }
	public ScheduledThreadPoolExecutor(int corePoolSize,
                             ThreadFactory threadFactory) {
        super(corePoolSize, Integer.MAX_VALUE, 0, TimeUnit.NANOSECONDS,
              new DelayedWorkQueue(), threadFactory);
    }
```
注:
	工具类中提供的创建线程池,所用的队列都是无界队列,因此会存在OOM问题,当然,也可以自定制自己的线程池方案,自定义拒绝策略:
```
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.RejectedExecutionHandler;
import java.util.concurrent.ThreadFactory;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;
	public class CustomThreadPoolExecutor {
	private ThreadPoolExecutor pool = null;
	
	
	/**
	 * 线程池初始化方法
	 * 
	 * corePoolSize 核心线程池大小----10
	 * maximumPoolSize 最大线程池大小----30
	 * keepAliveTime 线程池中超过corePoolSize数目的空闲线程最大存活时间----30+单位TimeUnit
	 * TimeUnit keepAliveTime时间单位----TimeUnit.MINUTES
	 * workQueue 阻塞队列----new ArrayBlockingQueue<Runnable>(10)====10容量的阻塞队列
	 * threadFactory 新建线程工厂----new CustomThreadFactory()====定制的线程工厂
	 * rejectedExecutionHandler 当提交任务数超过maxmumPoolSize+workQueue之和时,
	 * 							即当提交第41个任务时(前面线程都没有执行完,此测试方法中用sleep(100)),
	 * 						          任务会交给RejectedExecutionHandler来处理
	 */
	public void init() {
		pool = new ThreadPoolExecutor(
				10,
				30,
				30,
				TimeUnit.MINUTES,
				new ArrayBlockingQueue<Runnable>(10),
				new CustomThreadFactory(),
				new CustomRejectedExecutionHandler());
	}
	public void destory() {
		if(pool != null) {
			pool.shutdownNow();
		}
	}
	
	
	public ExecutorService getCustomThreadPoolExecutor() {
		return this.pool;
	}
	
	private class CustomThreadFactory implements ThreadFactory {
	private AtomicInteger count = new AtomicInteger(0);
		
		@Override
		public Thread newThread(Runnable r) {
			Thread t = new Thread(r);
			String threadName = CustomThreadPoolExecutor.class.getSimpleName() + count.addAndGet(1);
			System.out.println(threadName);
			t.setName(threadName);
			return t;
		}
	}
	
	
	private class CustomRejectedExecutionHandler implements RejectedExecutionHandler {
	@Override
		public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
			// 记录异常
			// 报警处理等
			System.out.println("error.............");
		}
	}
	
	
	
	// 测试构造的线程池
	public static void main(String[] args) {
		CustomThreadPoolExecutor exec = new CustomThreadPoolExecutor();
		// 1.初始化
		exec.init();
		
		ExecutorService pool = exec.getCustomThreadPoolExecutor();
		for(int i=1; i<100; i++) {
			System.out.println("提交第" + i + "个任务!");
			pool.execute(new Runnable() {
				@Override
				public void run() {
					try {
						Thread.sleep(3000);
					} catch (InterruptedException e) {
						e.printStackTrace();
					}
					System.out.println("running=====");
				}
			});
		}
		
		
		
		// 2.销毁----此处不能销毁,因为任务没有提交执行完,如果销毁线程池,任务也就无法执行了
		// exec.destory();
		
		try {
			Thread.sleep(10000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}
}
```
&nbsp;&nbsp;&nbsp;&nbsp;方法中建立一个核心线程数为30个，缓冲队列有10个的线程池。每个线程任务，执行时会先睡眠3秒，保证提交10任务时，线程数目被占用完，再提交30任务时，阻塞队列被占用完，，这样提交第41个任务是，会交给CustomRejectedExecutionHandler 异常处理类来处理。 
&nbsp;提交任务的代码如下：
```
	public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }
```
注意：41以后提交的任务就不能正常处理了，因为，execute中提交到任务队列是用的offer方法，如上面代码，这个方法是非阻塞的，所以就会交给CustomRejectedExecutionHandler 来处理，所以对于大数据量的任务来说，这种线程池，如果不设置队列长度会OOM，设置队列长度，会有任务得不到处理，接下来我们构建一个阻塞的自定义线程池 
	
定制属于自己的阻塞线程池 :
```
package com.tongbanjie.trade.test.commons;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.RejectedExecutionHandler;
import java.util.concurrent.ThreadFactory;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;
	public class CustomThreadPoolExecutor {  
	  
      
    private ThreadPoolExecutor pool = null;  
      
      
    /** 
     * 线程池初始化方法 
     *  
     * corePoolSize 核心线程池大小----1 
     * maximumPoolSize 最大线程池大小----3 
     * keepAliveTime 线程池中超过corePoolSize数目的空闲线程最大存活时间----30+单位TimeUnit 
     * TimeUnit keepAliveTime时间单位----TimeUnit.MINUTES 
     * workQueue 阻塞队列----new ArrayBlockingQueue<Runnable>(5)====5容量的阻塞队列 
     * threadFactory 新建线程工厂----new CustomThreadFactory()====定制的线程工厂 
     * rejectedExecutionHandler 当提交任务数超过maxmumPoolSize+workQueue之和时, 
     *                          即当提交第41个任务时(前面线程都没有执行完,此测试方法中用sleep(100)), 
     *                                任务会交给RejectedExecutionHandler来处理 
     */  
    public void init() {  
        pool = new ThreadPoolExecutor(  
                1,  
                3,  
                30,  
                TimeUnit.MINUTES,  
                new ArrayBlockingQueue<Runnable>(5),  
                new CustomThreadFactory(),  
                new CustomRejectedExecutionHandler());  
    }  
  
      
    public void destory() {  
        if(pool != null) {  
            pool.shutdownNow();  
        }  
    }  
      
      
    public ExecutorService getCustomThreadPoolExecutor() {  
        return this.pool;  
    }  
      
    private class CustomThreadFactory implements ThreadFactory {  
  
        private AtomicInteger count = new AtomicInteger(0);  
          
        @Override  
        public Thread newThread(Runnable r) {  
            Thread t = new Thread(r);  
            String threadName = CustomThreadPoolExecutor.class.getSimpleName() + count.addAndGet(1);  
            System.out.println(threadName);  
            t.setName(threadName);  
            return t;  
        }  
    }  
      
      
    private class CustomRejectedExecutionHandler implements RejectedExecutionHandler {  
  
        @Override  
        public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {  
        	try {
                                // 核心改造点，由blockingqueue的offer改成put阻塞方法
				executor.getQueue().put(r);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
        }  
    }  
      
      
      
    // 测试构造的线程池  
    public static void main(String[] args) {  
    	
        CustomThreadPoolExecutor exec = new CustomThreadPoolExecutor();  
        // 1.初始化  
        exec.init();  
          
        ExecutorService pool = exec.getCustomThreadPoolExecutor();  
        for(int i=1; i<100; i++) {  
            System.out.println("提交第" + i + "个任务!");  
            pool.execute(new Runnable() {  
                @Override  
                public void run() {  
                    try {  
                    	System.out.println(">>>task is running====="); 
                        TimeUnit.SECONDS.sleep(10);
                    } catch (InterruptedException e) {  
                        e.printStackTrace();  
                    }  
                }  
            });  
        }  
          
          
        // 2.销毁----此处不能销毁,因为任务没有提交执行完,如果销毁线程池,任务也就无法执行了  
        // exec.destory();  
          
        try {  
            Thread.sleep(10000);  
        } catch (InterruptedException e) {  
            e.printStackTrace();  
        }  
    }  
}  
	
	解释：当提交任务被拒绝时，进入拒绝机制，我们实现拒绝方法，把任务重新用阻塞提交方法put提交，实现阻塞提交任务功能，防止队列过大，OOM，提交被拒绝方法在下面 
	
	   
	public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
	int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            // 进入拒绝机制， 我们把runnable任务拿出来，重新用阻塞操作put，来实现提交阻塞功能
            reject(command);
    }
```

总结： 
1. 用ThreadPoolExecutor自定义线程池，看线程是的用途，如果任务量不大，可以用无界队列，如果任务量非常大，要用有界队列，防止OOM 
2. 如果任务量很大，还要求每个任务都处理成功，要对提交的任务进行阻塞提交，重写拒绝机制，改为阻塞提交。保证不抛弃一个任务 
3. 最大线程数一般设为2N+1最好，N是CPU核数 
4. 核心线程数，看应用，如果是任务，一天跑一次，设置为0，合适，因为跑完就停掉了，如果是常用线程池，看任务量，是保留一个核心还是几个核心线程数 
5. 如果要获取任务执行结果，用CompletionService，但是注意，获取任务的结果的要重新开一个线程获取，如果在主线程获取，就要等任务都提交后才获取，就会阻塞大量任务结果，队列过大OOM，所以最好异步开个线程获取结果
	
	
	
	
	
	
	
	
	
	
	

