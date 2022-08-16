# Java四大引用详解：强引用、软引用、弱引用、虚引用

从JDK 1.2版本开始，对象的引用被划分为4种级别，从而使程序能更加灵活地控制对象的生命周期，这4种级别由高到低依次为：强引用、软引用、弱引用和虚引用。

## 强引用

强引用是最普遍的引用，一般把一个对象赋给一个引用变量，这个引用变量就是强引用。

比如：

```ini
//  强引用
MikeChen mikechen=new MikeChen();
复制代码
```

在一个方法的内部有一个强引用，这个引用保存在Java栈中，而真正的引用内容(MikeChen)保存在Java堆中。

如果一个对象具有强引用，垃圾回收器不会回收该对象，当内存空间不足时，JVM 宁愿抛出 OutOfMemoryError异常。

如果强引用对象不使用时，需要弱化从而使GC能够回收，如下：

```ini
//帮助垃圾收集器回收此对象
mikechen=null;
```

显式地设置mikechen对象为null，或让其超出对象的生命周期范围，则GC认为该对象不存在引用，这时就可以回收这个对象，具体什么时候收集这要取决于GC算法。

举例：

```typescript
package com.mikechen.java.refenence;

/**
* 强引用举例
*
* @author mikechen
*/
public class StrongRefenenceDemo {

    public static void main(String[] args) {
        Object o1 = new Object();
        Object o2 = o1;
        o1 = null;
        System.gc();
        System.out.println(o1);  //null
        System.out.println(o2);  //java.lang.Object@2503dbd3
    }
}
复制代码
```

StrongRefenenceDemo 中尽管 o1已经被回收，但是 o2 强引用 o1，一直存在，所以不会被GC回收。

## **软引用**

软引用是一种相对强引用弱化了一些的引用，需要用java.lang.ref.SoftReference 类来实现。

比如：

```javascript
String str=new String("abc");                                     // 强引用
SoftReference<String> softRef=new SoftReference<String>(str);     // 软引用
复制代码
```

如果一个对象只具有软引用，则内存空间足够，垃圾回收器就不会回收它，如果内存空间不足了，就会回收这些对象的内存。

先通过一个例子来了解一下软引用：

```javascript
/**
* 弱引用举例
*
* @author mikechen
*/
Object obj = new Object();
SoftReference softRef = new SoftReference<Object>(obj);//删除强引用
obj = null;//调用gc

// 对象依然存在
System.gc();System.out.println("gc之后的值：" + softRef.get());
复制代码
```

软引用可以和一个引用队列(ReferenceQueue)联合使用,如果软引用所引用对象被垃圾回收，Java虚拟机就会把这个软引用加入到与之关联的引用队列中。

```ini
ReferenceQueue<Object> queue = new ReferenceQueue<>();
Object obj = new Object();
SoftReference softRef = new SoftReference<Object>(obj,queue);//删除强引用
obj = null;//调用gc
System.gc();
System.out.println("gc之后的值: " + softRef.get()); // 对象依然存在
//申请较大内存使内存空间使用率达到阈值，强迫gc
byte[] bytes = new byte[100 * 1024 * 1024];//如果obj被回收，则软引用会进入引用队列
Reference<?> reference = queue.remove();if (reference != null){
    System.out.println("对象已被回收: "+ reference.get());  // 对象为null
}
复制代码
```

软引用通常用在对内存敏感的程序中，比如高速缓存就有用到软引用，内存够用的时候就保留，不够用就回收。

我们看下 Mybatis 缓存类 SoftCache 用到的软引用：

```kotlin
public Object getObject(Object key) {
    Object result = null;
    SoftReference<Object> softReference = (SoftReference)this.delegate.getObject(key);
    if (softReference != null) {
        result = softReference.get();
        if (result == null) {
            this.delegate.removeObject(key);
        } else {
            synchronized(this.hardLinksToAvoidGarbageCollection) {
                this.hardLinksToAvoidGarbageCollection.addFirst(result);
                if (this.hardLinksToAvoidGarbageCollection.size() > this.numberOfHardLinks) {
                    this.hardLinksToAvoidGarbageCollection.removeLast();
                }
            }
        }
    }
    return result;}
复制代码
```

注意：软引用对象是在jvm内存不够的时候才会被回收，我们调用System.gc()方法只是起通知作用，JVM什么时候扫描回收对象是JVM自己的状态决定的，就算扫描到软引用对象也不一定会回收它，只有内存不够的时候才会回收。

 **弱引用**

弱引用的使用和软引用类似，只是关键字变成了 WeakReference：

```ini
MikeChen mikechen = new MikeChen();
WeakReference<MikeChen> wr = new WeakReference<MikeChen>(mikechen );
复制代码
```

弱引用的特点是不管内存是否足够，只要发生 GC，都会被回收。

举例说明：

```csharp
package com.mikechen.java.refenence;

import java.lang.ref.WeakReference;

/**
* 弱引用
*
* @author mikechen
*/
public class WeakReferenceDemo {
    public static void main(String[] args) {
        Object o1 = new Object();
        WeakReference<Object> w1 = new WeakReference<Object>(o1);

        System.out.println(o1);
        System.out.println(w1.get());

        o1 = null;
        System.gc();

        System.out.println(o1);
        System.out.println(w1.get());
    }
}
复制代码
```

 

**弱引用的应用**

**WeakHashMap**

```ini
public class WeakHashMapDemo {

    public static void main(String[] args) throws InterruptedException {
        myHashMap();
        myWeakHashMap();
    }

    public static void myHashMap() {
        HashMap<String, String> map = new HashMap<String, String>();
        String key = new String("k1");
        String value = "v1";
        map.put(key, value);
        System.out.println(map);

        key = null;
        System.gc();

        System.out.println(map);
    }

    public static void myWeakHashMap() throws InterruptedException {
        WeakHashMap<String, String> map = new WeakHashMap<String, String>();
        //String key = "weak";
        // 刚开始写成了上边的代码
        //思考一下，写成上边那样会怎么样？ 那可不是引用了
        String key = new String("weak");
        String value = "map";
        map.put(key, value);
        System.out.println(map);
        //去掉强引用
        key = null;
        System.gc();
        Thread.sleep(1000);
        System.out.println(map);
    }}
复制代码
```

当key只有弱引用时，GC发现后会自动清理键和值，作为简单的缓存表解决方案。

**ThreadLocal**

```scala
static class ThreadLocalMap {

    static class Entry extends WeakReference<ThreadLocal<?>> {
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
    //......}
复制代码
```

ThreadLocal.ThreadLocalMap.Entry 继承了弱引用，key为当前线程实例，和WeakHashMap基本相同。

 

## **虚引用**

虚引用”顾名思义，就是形同虚设，与其他几种引用都不同，虚引用并不会决定对象的生命周期。如果一个对象仅持有虚引用，那么它就和没有任何引用一样，在任何时候都可能被垃圾回收器回收。

虚引用也称为“幽灵引用”或者“幻影引用”，它是最弱的一种引用关系。

虚引用需要java.lang.ref.PhantomReference 来实现：

```ini
A a = new A();
ReferenceQueue<A> rq = new ReferenceQueue<A>();
PhantomReference<A> prA = new PhantomReference<A>(a, rq);
复制代码
```

虚引用主要用来跟踪对象被垃圾回收器回收的活动。

虚引用与软引用和弱引用的一个区别在于：虚引用必须和引用队列 （ReferenceQueue）联合使用，当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收对象的内存之前，把这个虚引用加入到与之 关联的引用队列中。

## **Java引用总结**

java4种引用的级别由高到低依次为：强引用 > 软引用 > 弱引用 > 虚引用。



| 引用类型 | 被垃圾回收时间 | 用途           | 生存时间       |
| -------- | -------------- | -------------- | -------------- |
| 强引用   | 从来不会       | 对象的一般状态 | JVM停止运行时  |
| 软引用   | 在内存不足时   | 对象的缓存     | 内存不足时终止 |
| 弱引用   | 在垃圾回收时   | 对象缓存       | gc运行后终止   |
| 虚引用   | unknown        | unknown        | unknown        |

