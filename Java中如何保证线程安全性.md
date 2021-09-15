# Java中如何保证线程安全性

## **一、线程安全在三个方面体现**

1.原子性：提供互斥访问，同一时刻只能有一个线程对数据进行操作，（atomic,synchronized）；

2.可见性：一个线程对主内存的修改可以及时地被其他线程看到，（synchronized,volatile）；

3.有序性：一个线程观察其他线程中的指令执行顺序，由于指令重排序，该观察结果一般杂乱无序，（happens-before原则）。

接下来，依次分析。

## **二、原子性---atomic**

JDK里面提供了很多atomic类，`AtomicInteger`,`AtomicLong`,`AtomicBoolean`等等。

它们是通过CAS完成原子性。

我们一次来看`AtomicInteger`，`AtomicStampedReference`，`AtomicLongArray`，`AtomicBoolean`。

#### （1）AtomicInteger

先来看一个AtomicInteger例子：

```
public class AtomicIntegerExample1 {
    // 请求总数
    public static int clientTotal = 5000;
    // 同时并发执行的线程数
    public static int threadTotal = 200;
 
    public static AtomicInteger count = new AtomicInteger(0);
 
    public static void main(String[] args) throws Exception {
        ExecutorService executorService = Executors.newCachedThreadPool();//获取线程池
        final Semaphore semaphore = new Semaphore(threadTotal);//定义信号量
        final CountDownLatch countDownLatch = new CountDownLatch(clientTotal);
        for (int i = 0; i < clientTotal ; i++) {
            executorService.execute(() -> {
                try {
                    semaphore.acquire();
                    add();
                    semaphore.release();
                } catch (Exception e) {
                    log.error("exception", e);
                }
                countDownLatch.countDown();
            });
        }
        countDownLatch.await();
        executorService.shutdown();
        log.info("count:{}", count.get());
    }
 
    private static void add() {
        count.incrementAndGet();
    }
}
```

我们可以执行看到最后结果是5000是线程安全的。

那么看AtomicInteger的`incrementAndGet()`方法：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

再看`getAndAddInt()`方法：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

这里面调用了`compareAndSwapInt()`方法：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

它是native修饰的，代表是java底层的方法，不是通过java实现的 。

再重新看`getAndAddInt()`，传来第一个值是当前的一个对象 ，比如是count.incrementAndGet()，那么在`getAndAddInt()`中，var1就是count，而var2第二个值是当前的值，比如想执行的是2+1=3操作，那么第二个参数是2，第三个参数是1 。

变量5（var5）是我们调用底层的方法而得到的底层当前的值，如果没有别的线程过来处理我们count变量的时候，那么它正常返回值是2。

因此传到compareAndSwapInt方法里的参数是（count对象，当前值2，当前从底层传过来的2，从底层取出来的值加上改变量var4）。

compareAndSwapInt()希望达到的目标是对于var1对象，如果当前的值var2和底层的值var5相等，那么把它更新成后面的值（var5+var4）.

compareAndSwapInt核心就是CAS核心。

关于count值为什么和底层值不一样：**count里面的值相当于存在于工作内存的值，底层就是主内存。**

#### （2）AtomicStampedReference

接下来我们看一下`AtomicStampedReference`。

关于CAS有一个ABA问题：开始是A，后来改为B，现在又改为A。解决办法就是：每次变量改变的时候，把变量的版本号加1。

这就用到了`AtomicStampedReference`。

我们来看AtomicStampedReference里的`compareAndSet()`实现：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

而在AtomicInteger里`compareAndSet()`实现：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

可以看到AtomicStampedReference里的`compareAndSet()`中多了 一个stamp比较（也就是版本），这个值是由每次更新时来维护的。

#### （3）AtomicLongArray

这种维护数组的atomic类，我们可以选择性地更新其中某一个索引对应的值，也是进行原子性操作。这种对数组的操作的各种方法，会多处一个索引。

比如，我们看一下`compareAndSet()`：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

#### （4）AtomicBoolean

看一段代码：

```
public class AtomicBooleanExample {
 
    private static AtomicBoolean isHappened = new AtomicBoolean(false);
 
    // 请求总数
    public static int clientTotal = 5000;
    // 同时并发执行的线程数
    public static int threadTotal = 200;
    public static void main(String[] args) throws Exception {
        ExecutorService executorService = Executors.newCachedThreadPool();
        final Semaphore semaphore = new Semaphore(threadTotal);
        final CountDownLatch countDownLatch = new CountDownLatch(clientTotal);
        for (int i = 0; i < clientTotal ; i++) {
            executorService.execute(() -> {
                try {
                    semaphore.acquire();
                    test();
                    semaphore.release();
                } catch (Exception e) {
                    log.error("exception", e);
                }
                countDownLatch.countDown();
            });
        }
        countDownLatch.await();
        executorService.shutdown();
        log.info("isHappened:{}", isHappened.get());
    }
    private static void test() {
        if (isHappened.compareAndSet(false, true)) {
            log.info("execute");
        }
    }
}
```

执行之后发现，`log.info("execute");`只执行了一次，且isHappend值为true。

原因就是当它第一次`compareAndSet()`之后，isHappend变为true，没有别的线程干扰。

通过使用AtomicBoolean，我们可以使某段代码只执行一次。

## **三、原子性---synchronized**

synchronized是一种同步锁，通过锁实现原子操作。

JDK提供锁分两种：一种是synchronized，依赖JVM实现锁，因此在这个关键字作用对象的作用范围内是同一时刻只能有一个线程进行操作；另一种是LOCK，是JDK提供的代码层面的锁，依赖CPU指令，代表性的是ReentrantLock。

synchronized修饰的对象有四种：

- 修饰代码块，作用于调用的对象；
- 修饰方法，作用于调用的对象；
- 修饰静态方法，作用于所有对象；
- 修饰类，作用于所有对象。

修饰代码块和方法：

```
@Slf4j
public class SynchronizedExample1 {
 
    // 修饰一个代码块
    public void test1(int j) {
        synchronized (this) {
            for (int i = 0; i < 10; i++) {
                log.info("test1 {} - {}", j, i);
            }
        }
    }
 
    // 修饰一个方法
    public synchronized void test2(int j) {
        for (int i = 0; i < 10; i++) {
            log.info("test2 {} - {}", j, i);
        }
    }
 
    public static void main(String[] args) {
        SynchronizedExample1 example1 = new SynchronizedExample1();
        SynchronizedExample1 example2 = new SynchronizedExample1();
        ExecutorService executorService = Executors.newCachedThreadPool();
        //一
        executorService.execute(() -> {
            example1.test1(1);
        });
        executorService.execute(() -> {
            example1.test1(2);
        });
        //二
        executorService.execute(() -> {
            example2.test2(1);
        });
        executorService.execute(() -> {
            example2.test2(2);
        });
        //三
        executorService.execute(() -> {
            example1.test1(1);
        });
        executorService.execute(() -> {
            example2.test1(2);
        });
    }
}
```

执行后可以看到对于情况一，test1内部方法块作用于example1，先执行完一次0-9输出，再执行下一次0-9输出；情况二，同情况一类似，作用于example2；情况三，可以看到交叉执行，test1分别独立作用于example1和example2，互不影响。

修饰静态方法和类：

```
@Slf4j
public class SynchronizedExample2 {
 
    // 修饰一个类
    public static void test1(int j) {
        synchronized (SynchronizedExample2.class) {
            for (int i = 0; i < 10; i++) {
                log.info("test1 {} - {}", j, i);
            }
        }
    }
 
    // 修饰一个静态方法
    public static synchronized void test2(int j) {
        for (int i = 0; i < 10; i++) {
            log.info("test2 {} - {}", j, i);
        }
    }
 
    public static void main(String[] args) {
        SynchronizedExample2 example1 = new SynchronizedExample2();
        SynchronizedExample2 example2 = new SynchronizedExample2();
        ExecutorService executorService = Executors.newCachedThreadPool();
        executorService.execute(() -> {
            example1.test1(1);
        });
        executorService.execute(() -> {
            example2.test1(2);
        });
    }
}
```

test1和test2会锁定调用它们的对象所属的类，同一个时间只有一个对象在执行。

## **四、可见性---volatile**

对于可见性，JVM提供了synchronized和volatile。这里我们看volatile。

#### （1）volatile的可见性是通过内存屏障和禁止重排序实现的

volatile会在写操作时，会在写操作后加一条store屏障指令，将本地内存中的共享变量值刷新到主内存：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

volatile在进行读操作时，会在读操作前加一条load指令，从内存中读取共享变量：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

#### （2）但是volatile不是原子性的，进行++操作不是安全的

```
@Slf4j
public class VolatileExample {
 
    // 请求总数
    public static int clientTotal = 5000;
 
    // 同时并发执行的线程数
    public static int threadTotal = 200;
 
    public static volatile int count = 0;
 
    public static void main(String[] args) throws Exception {
        ExecutorService executorService = Executors.newCachedThreadPool();
        final Semaphore semaphore = new Semaphore(threadTotal);
        final CountDownLatch countDownLatch = new CountDownLatch(clientTotal);
        for (int i = 0; i < clientTotal ; i++) {
            executorService.execute(() -> {
                try {
                    semaphore.acquire();
                    add();
                    semaphore.release();
                } catch (Exception e) {
                    log.error("exception", e);
                }
                countDownLatch.countDown();
            });
        }
        countDownLatch.await();
        executorService.shutdown();
        log.info("count:{}", count);
    }
 
    private static void add() {
        count++;
    }
}
```

执行后发现线程不安全，原因是 执行conut++ 时分成了三步，第一步是取出当前内存 count 值，这时 count 值时最新的，接下来执行了两步操作，分别是 +1 和重新写回主存。假设有两个线程同时在执行 count++ ，两个内存都执行了第一步，比如当前 count 值为 5 ，它们都读到了，然后两个线程分别执行了 +1 ，并写回主存，这样就丢掉了一次加一的操作。

#### （3）volatile适用的场景

既然volatile不适用于计数，那么volatile适用于哪些场景呢：

1. 对变量的写操作不依赖于当前值
2. 该变量没有包含在具有其他变量不变的式子中

因此，volatile适用于状态标记量：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

线程1负责初始化，线程2不断查询inited值，当线程1初始化完成后，线程2就可以检测到inited为true了。

## **五、有序性**

有序性是指，在JMM中，允许编译器和处理器对指令进行重排序，但是重排序过程不会影响到单线程程序的执行，却会影响到多线程并发执行的正确性。

可以通过volatile、synchronized、lock保证有序性。

另外，JMM具有先天的有序性，即不需要通过任何手段就可以得到保证的有序性。这称为happens-before原则。

如果两个操作的执行次序无法从happens-before原则推导出来，那么它们就不能保证它们的有序性。虚拟机可以随意地对它们进行重排序。

happens-before原则：

1. 程序次序规则：在一个单独的线程中，按照程序代码书写的顺序执行。
2. 锁定规则：一个unlock操作happen—before后面对同一个锁的lock操作。
3. volatile变量规则：对一个volatile变量的写操作happen—before后面对该变量的读操作。
4. 线程启动规则：Thread对象的start()方法happen—before此线程的每一个动作。
5. 线程终止规则：线程的所有操作都happen—before对此线程的终止检测，可以通过Thread.join()方法结束、Thread.isAlive()的返回值等手段检测到线程已经终止执行。
6. 线程中断规则：对线程interrupt()方法的调用happen—before发生于被中断线程的代码检测到中断时事件的发生。
7. 对象终结规则：一个对象的初始化完成（构造函数执行结束）happen—before它的finalize()方法的开始。
8. 传递性：如果操作A happen—before操作B，操作B happen—before操作C，那么可以得出A happen—before操作C。

- 