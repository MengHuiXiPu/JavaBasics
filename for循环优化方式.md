# for循环优化方式

我们初始化一个集合 list，如下：

```
List<String> list = new ArrayList<String>();
```

### 方法一：最常规的不加思考的写法

```
for (int i = 0; i < list.size(); i++) {
 System.out.println(list.get(i));
}
```

- 优点：较常见，易于理解
- 缺点：每次都要计算`list.size()`

### 方法二：数组长度提取出来

```
int m = list.size();
for (int i = 0; i < m; i++) {
      System.out.println(list.get(i));
}
```

- 优点：不必每次都计算

- 缺点：

- 1. m的作用域不够小，违反了最小作用域原则
  2. 不能在for循环中操作list的大小，比如除去或新加一个元素

### 方法三：数组长度提取出来

```
for (int i = 0, n = list.size(); i < n; i++) {
    System.out.println(list.get(i));
}
```

- 优点：不必每次都计算 ，变量的作用域遵循最小范围原则

- 缺点：

- 1. m的作用域不够小，违反了最小作用域原则
  2. 不能在for循环中操作list的大小，比如除去或新加一个元素

### 方法四：采用倒序的写法

```
for (int i = list.size() - 1; i >= 0; i--) {
System.out.println(list.get(i));
}
```

- 优点：不必每次都计算 ，变量的作用域遵循最小范围原则
- 缺点：1、结果的顺序会反 2、看起来不习惯，不易读懂
- 适用场合：与显示结果顺序无关的地方：比如保存之前数据的校验

### 方法五：Iterator 遍历

```
for (Iterator<String> it = list.iterator(); it.hasNext();) {
      System.out.println(it.next());
}
```

- 优点：简洁

### 方法六：jdk1.5后的写法

```
for (Object o : list) {
     System.out.println(o);
}
```

- 优点：简洁[结合泛型](http://mp.weixin.qq.com/s?__biz=MzI4Njc5NjM1NQ==&mid=2247490546&idx=2&sn=aede21695ffcc58a41ebda03e63c8b8e&chksm=ebd624dedca1adc8a2122b273139201434b38064d5b4db01729468f104f7b0d97e31f7d099e1&scene=21#wechat_redirect)使用更简洁
- 缺点：jdk1.4向下不兼容



### 方法七：循环嵌套外小内大原则

```
for (int i = 0; i < 10; i++) {
   for (int j = 0; j < 10000; j++) {
   }
}
```

原因

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

### 方法八：循环嵌套提取不需要循环的逻辑

```
 //前：
 int a = 10, b = 11;
  for (int i = 0; i < 10; i++) {
               i = i * a * b;
   } 
 
 
//后：
 int c = a * b;
 for (int i = 0; i < 10; i++) {
         i = i * c;
  }
```

### 方法九：异常处理写在循环外面

反例

```
for (int i = 0; i < 10; i++) {
     try {
 
     } catch (Exception e) {
 
     }
}
```

正例

```
try {
   for (int i = 0; i < 10; i++) {
   }
} catch (Exception e) {
 
}
```

