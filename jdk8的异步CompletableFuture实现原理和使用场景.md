# jdk8的异步CompletableFuture实现原理和使用场景，具体讲下



![图片](https://mmbiz.qpic.cn/mmbiz_png/obDoO79MTFGwvf4xfxX24InPWQFlGxAQXVqVwIxa4UlJuhfpL6RLib4d6QHUb8yfZR4qNgCiajuIAd2HIKVBemicw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

面试中关于JDK中异步的API能说清楚，谈薪资时也比较有底气，现在并发多线程是面试官最爱问的题型

### 1.概述

CompletableFuture是jdk1.8引入的实现类。扩展了Future和CompletionStage，是一个可以在任务完成阶段触发一些操作Future。简单的来讲就是可以实现异步回调。

### 2.为什么引入CompletableFuture

对于jdk1.5的Future，虽然提供了异步处理任务的能力，但是获取结果的方式很不优雅，还是需要通过阻塞（或者轮训）的方式。如何避免阻塞呢？其实就是注册回调。

业界结合观察者模式实现异步回调。也就是当任务执行完成后去通知观察者。比如Netty的ChannelFuture，可以通过注册监听实现异步结果的处理。

##### Netty的ChannelFuture

```
public Promise<V> addListener(GenericFutureListener<? extends Future<? super V>> listener) {  
    checkNotNull(listener, "listener");  
    synchronized (this) {  
        addListener0(listener);  
    }  
    if (isDone()) {  
        notifyListeners();  
    }  
    return this;  
}  
private boolean setValue0(Object objResult) {  
    if (RESULT_UPDATER.compareAndSet(this, null, objResult) ||  
        RESULT_UPDATER.compareAndSet(this, UNCANCELLABLE, objResult)) {  
        if (checkNotifyWaiters()) {  
            notifyListeners();  
        }  
        return true;  
    }  
    return false;  
}  
```

通过addListener方法注册监听。如果任务完成，会调用notifyListeners通知。

CompletableFuture通过扩展Future，引入函数式编程，通过回调的方式去处理结果。

### 3.功能

CompletableFuture的功能主要体现在他的CompletionStage。

可以实现如下等功能

- 转换（thenCompose）
- 组合（thenCombine）
- 消费（thenAccept）
- 运行（thenRun）。
- 带返回的消费（thenApply）

消费和运行的区别：

消费使用执行结果。运行则只是运行特定任务。具体其他功能大家可以根据需求自行查看。

> CompletableFuture借助CompletionStage的方法可以实现链式调用。并且可以选择同步或者异步两种方式。

这里举个简单的例子来体验一下他的功能。

```
public static void thenApply() {  
    ExecutorService executorService = Executors.newFixedThreadPool(2);  
    CompletableFuture cf = CompletableFuture.supplyAsync(() -> {  
        try {  
            //  Thread.sleep(2000);  
        } catch (Exception e) {  
            e.printStackTrace();  
        }  
        System.out.println("supplyAsync " + Thread.currentThread().getName());  
        return "hello";  
    }, executorService).thenApplyAsync(s -> {  
        System.out.println(s + "world");  
        return "hhh";  
    }, executorService);  
    cf.thenRunAsync(() -> {  
        System.out.println("ddddd");  
    });  
    cf.thenRun(() -> {  
        System.out.println("ddddsd");  
    });  
    cf.thenRun(() -> {  
        System.out.println(Thread.currentThread());  
        System.out.println("dddaewdd");  
    });  
}  
```

执行结果

```
supplyAsync pool-1-thread-1  
helloworld  
ddddd  
ddddsd  
Thread[main,5,main]  
dddaewdd  
```

根据结果我们可以看到会有序执行对应任务。

注意：

> 如果是同步执行cf.thenRun。他的执行线程可能main线程，也可能是执行源任务的线程。如果执行源任务的线程在main调用之前执行完了任务。那么cf.thenRun方法会由main线程调用。

这里说明一下，如果是同一任务的依赖任务有多个：

- 如果这些依赖任务都是同步执行。那么假如这些任务被当前调用线程（main）执行，则是有序执行，假如被执行源任务的线程执行，那么会是倒序执行。因为内部任务数据结构为LIFO。
- 如果这些依赖任务都是异步执行，那么他会通过异步线程池去执行任务。不能保证任务的执行顺序。

上面的结论是通过阅读源代码得到的。下面我们深入源代码。

### 4.源码追踪

##### 创建CompletableFuture

创建的方法有很多，甚至可以直接new一个。我们来看一下supplyAsync异步创建的方法。

```
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier,  
                                                   Executor executor) {  
    return asyncSupplyStage(screenExecutor(executor), supplier);  
}  
static Executor screenExecutor(Executor e) {  
    if (!useCommonPool && e == ForkJoinPool.commonPool())  
        return asyncPool;  
    if (e == null) throw new NullPointerException();  
    return e;  
}  
```

入参Supplier，带返回值的函数。如果是异步方法，并且传递了执行器，那么会使用传入的执行器去执行任务。否则采用公共的ForkJoin并行线程池，如果不支持并行，新建一个线程去执行。[关注Java项目分享](http://mp.weixin.qq.com/s?__biz=Mzg2ODU0NTA2Mw==&mid=2247488067&idx=2&sn=bc20c4f449d5cada1335b24ea6210687&chksm=ceabf50bf9dc7c1d8486a0c1954d658778d5c652355560b65543b39aabee3cdce47732198b33&scene=21#wechat_redirect)

这里我们需要注意ForkJoin是通过守护线程去执行任务的。所以必须有非守护线程的存在才行。

##### asyncSupplyStage方法

```
static <U> CompletableFuture<U> asyncSupplyStage(Executor e,  
                                                 Supplier<U> f) {  
    if (f == null) throw new NullPointerException();  
    CompletableFuture<U> d = new CompletableFuture<U>();  
    e.execute(new AsyncSupply<U>(d, f));  
    return d;  
}  
```

这里会创建一个用于返回的CompletableFuture。

然后构造一个AsyncSupply，并将创建的CompletableFuture作为构造参数传入。

那么，任务的执行完全依赖AsyncSupply。

##### AsyncSupply#run

```
public void run() {  
    CompletableFuture<T> d; Supplier<T> f;  
    if ((d = dep) != null && (f = fn) != null) {  
        dep = null; fn = null;  
        if (d.result == null) {  
            try {  
                d.completeValue(f.get());  
            } catch (Throwable ex) {  
                d.completeThrowable(ex);  
            }  
        }  
        d.postComplete();  
    }  
}  
```

1. 该方法会调用Supplier的get方法。并将结果设置到CompletableFuture中。我们应该清楚这些操作都是在异步线程中调用的。
2. `d.postComplete`方法就是通知任务执行完成。触发后续依赖任务的执行，也就是实现CompletionStage的关键点。[关注Java项目分享](http://mp.weixin.qq.com/s?__biz=Mzg2ODU0NTA2Mw==&mid=2247488067&idx=2&sn=bc20c4f449d5cada1335b24ea6210687&chksm=ceabf50bf9dc7c1d8486a0c1954d658778d5c652355560b65543b39aabee3cdce47732198b33&scene=21#wechat_redirect)

在看postComplete方法之前我们先来看一下创建依赖任务的逻辑。

##### thenAcceptAsync方法

```
public CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> action) {  
    return uniAcceptStage(asyncPool, action);  
}  
private CompletableFuture<Void> uniAcceptStage(Executor e,  
                                               Consumer<? super T> f) {  
    if (f == null) throw new NullPointerException();  
    CompletableFuture<Void> d = new CompletableFuture<Void>();  
    if (e != null || !d.uniAccept(this, f, null)) {  
        # 1  
        UniAccept<T> c = new UniAccept<T>(e, d, this, f);  
        push(c);  
        c.tryFire(SYNC);  
    }  
    return d;  
}  
```

上面提到过。thenAcceptAsync是用来消费CompletableFuture的。该方法调用uniAcceptStage。

**uniAcceptStage逻辑：**

1. 构造一个CompletableFuture，主要是为了链式调用。
2. 如果为异步任务，直接返回。因为源任务结束后会触发异步线程执行对应逻辑。
3. 如果为同步任务（e==null），会调用d.uniAccept方法。这个方法在这里逻辑：如果源任务完成，调用f，返回true。否则进入if代码块（Mark 1）。
4. 如果是异步任务直接进入if（Mark 1）。

**Mark1逻辑：**

1. 构造一个UniAccept，将其push入栈。这里通过CAS实现乐观锁实现。
2. 调用c.tryFire方法。

```
final CompletableFuture<Void> tryFire(int mode) {  
    CompletableFuture<Void> d; CompletableFuture<T> a;  
    if ((d = dep) == null ||  
        !d.uniAccept(a = src, fn, mode > 0 ? null : this))  
        return null;  
    dep = null; src = null; fn = null;  
    return d.postFire(a, mode);  
}  
```

1. 会调用d.uniAccept方法。其实该方法判断源任务是否完成，如果完成则执行依赖任务，否则返回false。
2. 如果依赖任务已经执行，调用d.postFire，主要就是Fire的后续处理。根据不同模式逻辑不同。

这里简单说一下，其实mode有同步异步，和迭代。迭代为了避免无限递归。

**这里强调一下d.uniAccept方法的第三个参数。**

如果是异步调用（mode>0），传入null。否则传入this。

区别看下面代码。c不为null会调用c.claim方法。[关注Java项目分享](http://mp.weixin.qq.com/s?__biz=Mzg2ODU0NTA2Mw==&mid=2247488067&idx=2&sn=bc20c4f449d5cada1335b24ea6210687&chksm=ceabf50bf9dc7c1d8486a0c1954d658778d5c652355560b65543b39aabee3cdce47732198b33&scene=21#wechat_redirect)

```
try {  
    if (c != null && !c.claim())  
        return false;  
    @SuppressWarnings("unchecked") S s = (S) r;  
    f.accept(s);  
    completeNull();  
} catch (Throwable ex) {  
    completeThrowable(ex);  
}  
  
final boolean claim() {  
    Executor e = executor;  
    if (compareAndSetForkJoinTaskTag((short)0, (short)1)) {  
        if (e == null)  
            return true;  
        executor = null; // disable  
        e.execute(this);  
    }  
    return false;  
}  
```

**claim方法是逻辑：**

- 如果异步线程为null。说明同步，那么直接返回true。最后上层函数会调用f.accept(s)同步执行任务。
- 如果异步线程不为null，那么使用异步线程去执行this。

this的run任务如下。也就是在异步线程同步调用tryFire方法。达到其被异步线程执行的目的。

```
public final void run(){   
   tryFire(ASYNC);   
}  
```

看完上面的逻辑，我们基本理解依赖任务的逻辑。

其实就是先判断源任务是否完成，如果完成，直接在对应线程执行以来任务（如果是同步，则在当前线程处理，否则在异步线程处理）

如果任务没有完成，直接返回，因为等任务完成之后会通过postComplete去触发调用依赖任务。

##### postComplete方法

```
final void postComplete() {  
    /*  
     * On each step, variable f holds current dependents to pop  
     * and run.  It is extended along only one path at a time,  
     * pushing others to avoid unbounded recursion.  
     */  
    CompletableFuture<?> f = this; Completion h;  
    while ((h = f.stack) != null ||  
           (f != this && (h = (f = this).stack) != null)) {  
        CompletableFuture<?> d; Completion t;  
        if (f.casStack(h, t = h.next)) {  
            if (t != null) {  
                if (f != this) {  
                    pushStack(h);  
                    continue;  
                }  
                h.next = null;    // detach  
            }  
            f = (d = h.tryFire(NESTED)) == null ? this : d;  
        }  
    }  
}  
```

在源任务完成之后会调用。

其实逻辑很简单，就是迭代堆栈的依赖任务。调用h.tryFire方法。NESTED就是为了避免递归死循环。因为FirePost会调用postComplete。如果是NESTED，则不调用。

堆栈的内容其实就是在依赖任务创建的时候加入进去的。上面我们已经提到过。[关注Java项目分享](http://mp.weixin.qq.com/s?__biz=Mzg2ODU0NTA2Mw==&mid=2247488067&idx=2&sn=bc20c4f449d5cada1335b24ea6210687&chksm=ceabf50bf9dc7c1d8486a0c1954d658778d5c652355560b65543b39aabee3cdce47732198b33&scene=21#wechat_redirect)

### 4.总结

基本上述源码已经分析了逻辑。

因为涉及异步等操作，我们需要理一下（这里针对全异步任务）：

1. 创建CompletableFuture成功之后会通过异步线程去执行对应任务。
2. 如果CompletableFuture还有依赖任务（异步），会将任务加入到CompletableFuture的堆栈保存起来。以供后续完成后执行依赖任务。

> 当然，创建依赖任务并不只是将其加入堆栈。如果源任务在创建依赖任务的时候已经执行完成，那么当前线程会触发依赖任务的异步线程直接处理依赖任务。并且会告诉堆栈其他的依赖任务源任务已经完成。

主要是考虑代码的复用。所以逻辑相对难理解。

postComplete方法会被源任务线程执行完源任务后调用。同样也可能被依赖任务线程后调用。

执行依赖任务的方法主要就是靠tryFire方法。因为这个方法可能会被多种不同类型线程触发，所以逻辑也绕一点。（其他依赖任务线程、源任务线程、当前依赖任务线程）

- 如果是当前依赖任务线程，那么会执行依赖任务，并且会通知其他依赖任务。
- 如果是源任务线程，和其他依赖任务线程，则将任务转换给依赖线程去执行。不需要通知其他依赖任务，避免死递归。

不得不说Doug Lea的编码，真的是艺术。代码的复用性全体现在逻辑上了。