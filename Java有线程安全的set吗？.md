# Java有线程安全的set吗？

在多线程环境下，要使用线程安全的集合，比如，ConcurrentHashMap是线程安全的HashMap，CopyOnWriteArrayList是线程安全的ArrayList。那么HashSet对应的线程安全集合，是什么呢？java有没有提供默认实现呢？

在java的concurrent包中，我找到了CopyOnWriteArraySet，那么它是线程安全的吗？下面是测试代码。

```
public static void main(String[] args) {
        Set<String> set = new CopyOnWriteArraySet<>();
        ExecutorService service = Executors.newFixedThreadPool(12);
        int times = 10000;
        AtomicInteger flag = new AtomicInteger(0);
        for(int i = 0; i < times; i ++){
            service.execute(()->{
                set.add("a" + flag.getAndAdd(1));
            });
        }
        service.shutdown();
        try {
            service.awaitTermination(Long.MAX_VALUE, TimeUnit.DAYS);
        }catch (Exception e){
            e.printStackTrace();
        }
        System.out.println(set.size());
    }
```

经过多次执行，结果都是10000。可以说明，CopyOnWriteArraySet是线程安全的Set。

那么CopyOnWriteArraySet是如何保证写入时的线程安全呢？以下是CopyOnWriteArraySet的add源码。

```
    public boolean add(E e) {
        return al.addIfAbsent(e);
    }
    public boolean addIfAbsent(E e) {
        Object[] snapshot = getArray();
        return indexOf(e, snapshot, 0, snapshot.length) >= 0 ? false :
            addIfAbsent(e, snapshot);
    }
    private static int indexOf(Object o, Object[] elements,
                               int index, int fence) {
        if (o == null) {
            for (int i = index; i < fence; i++)
                if (elements[i] == null)
                    return i;
        } else {
            for (int i = index; i < fence; i++)
                if (o.equals(elements[i]))
                    return i;
        }
        return -1;
    }
    private boolean addIfAbsent(E e, Object[] snapshot) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] current = getArray();
            int len = current.length;
            if (snapshot != current) {
                // Optimize for lost race to another addXXX operation
                int common = Math.min(snapshot.length, len);
                for (int i = 0; i < common; i++)
                    if (current[i] != snapshot[i] && eq(e, current[i]))
                        return false;
                if (indexOf(e, current, common, len) >= 0)
                        return false;
            }
            Object[] newElements = Arrays.copyOf(current, len + 1);
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }
```

从源码可以看出，CopyOnWriteArraySet底层采用了CopyOnWriteArrayList数据结构来实现。在add元素时，采用的是可重入锁来实现线程安全。