# 玩转Java注解：元注解、内置注解、自定义注解的原理和实现



#### 1、@Override 重写

概念：检查该方法是否是重写方法。如果发现其父类，或者是引用的接口中并没有该方法时，会报编译错误。

```
//这个extends 不要在意，我写上去只是为了更加方便直观的去理解，Object是万物之源，不写也会默认是其子类，不用解释过多吧？
public class Annotation1 extends Object{

    @Override
    public String toString (){
        return "我是重新定义过的toString方法";
    }
}
```

@Override（重写），这个大家应该很熟悉，重写父类的方法。我们可以看下Object类中toString()是什么样子的。

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbud9C0eVziaLd0OF93vvfW790iajnxys5LibynbLUQoPfDC9jFvjfSjuD7bskX48cSLQOSmC8lCHk24KA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

那么显而易见，使用了@Override（重写）注解，方法名、方法参数必须得和父类保持一致，否则会报错。如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbud9C0eVziaLd0OF93vvfW790hDdZAJJw7C8CWPn64ooiagyB1WricicR6lo2Hy8nUr4dibHnsh9WsMteqA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

如果不加@Override（重写）注解，则正常编译。

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbud9C0eVziaLd0OF93vvfW7905CRaHlqk1cWFjSWG5u2pVGDickjuA1EoHicEvwlYOIibrah7lprcKga6A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

#### 2、@Deprecated 过期警告

概念：标记过时方法。如果使用该方法，会报编译警告。在开发中，我们经常能遇到这样的情况，如下图：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbud9C0eVziaLd0OF93vvfW790hJSq1bMHbhcrtGvdKcUzWEntLOE76qLia28XIn6IdXzA92xUFAAs03w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

在jdk中有大量这样的方法，我就不举例了，自己写一个可能会更加方便理解。

```
public class Annotation1 extends Object{
    public static void main(String[] args) {
        testDeprecated.toString1();
    }
}
class testDeprecated {
    @Deprecated
    public static String toString1(){
        return "我是重新定义过的toString方法";
    }
}
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbud9C0eVziaLd0OF93vvfW790XRGBSElzQmmSXfic7dDNAD5NACALyaJrTm7OjswBSYwJH98c2DMgL6Q/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

注意点：这个不是报错，只是警告，提醒我们这个方法可能会有问题，可能有更好的方法来实现！

#### 3、@SuppressWarnings 忽略警告

概念：指示编译器去忽略注解中声明的警告。

平时开发中，我们会遇到这样的情况，如下图：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

这也不是错误，这是提醒我们，该方法没有使用到，警告提醒的作用。加上@SuppressWarnings注解后。

```
public class Annotation1 extends Object{
    public static void main(String[] args) {

    }
    @SuppressWarnings("all")
    public static void testSuppressWarnings(){
        System.out.println("测试+testSuppressWarnings忽略警告！");
    }
}
```

方法成功高亮起来，并且没有警告提示了！

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbud9C0eVziaLd0OF93vvfW790PIxJXicTUJkW4TWZklUPnOj0ErFHF0rv52JG3xh3a5nKG6NgkaLMeCA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

我们可以点进去看下这个注解为什么需要参数？

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbud9C0eVziaLd0OF93vvfW790xiawpjU4ogibunW2GtyUOE9Q5PzOCp7IzUyOFJAFMs48mJOQYiaI3BpGA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

看这里，这个不是方法哦，这是参数。

在注解中的参数格式：calss + 参数名 + ()！这个需要强行记忆哦，回头我们自定义注解时也需要用到。换一种写法加深理解！如下图：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbud9C0eVziaLd0OF93vvfW790wZDvOpMqzGPCAh8xTdWicrjgvpAWMApj24grJmK5zPpcV7WMyV9LiavA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

注意点：当注解中只有一个参数时，我们无需加上参数名，注解会自动帮我们匹配的。

### 二、元注解

概念：顾名思义，元注解就是给注解使用的注解！

#### 1、@Retention 作用域-（常用）

概念：表示在什么级别保存该注解信息。在实际开发中，我们一般都写RUNTIME，除非项目有特殊需求！我们看下@Retention的源码。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

可以看到，需要一个参数，进参数瞅瞅。

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbud9C0eVziaLd0OF93vvfW790OaAOnIQexOGSVulF8HT5m7yBx3lbhiaWBteJCGNYFZSO4iaKRdkZEQfg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

- SOURCE：源代码时有用。
- CLASS：class文件中有用，但会被jvm丢弃。
- RUNTIME：运行时有用。
- 关系：RUNTIME>CLASS>SOURCE

后面我们自定义注解时，每个都需要用该注解！

#### 2、@Documented 作用文档

概念：将此注解包含在 javadoc 中 ，它代表着此注解会被javadoc工具提取成文档。

老规矩看下源码：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbud9C0eVziaLd0OF93vvfW7909Qk1sbCR1SzQCMp0bJAsbEkrEYfUDhIexJnZlqK49Xw7KdDhfYOvxQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

无参的注解，作用域为`RetentionPolicy.RUNTIME`，运行时有用！这个只是用来作为标记，了解即可，在实际运行后会将该注解写入javadoc中，方便查看。

#### 3、@Target 目标-（常用）

概念：标记这个注解应该是使用在哪种 Java 成员上面！

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbud9C0eVziaLd0OF93vvfW790ukkonfvpUzkPeq2lGD5iaxEFAibXibU1PQbGV33nNJD7iaVfg7UmI1r1WQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

参数源码：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbud9C0eVziaLd0OF93vvfW790KlSYic3YnolpiaDEqicXTGObDLEcgKgXnicMeyr4mfDSVAsiaEB7DhVaKRQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

注意这里是数组格式的参数，证明可以传多个值。

- `@Target(ElementType.TYPE)`——接口、类、枚举、注解
- `@Target(ElementType.FIELD)`——字段、枚举的常量
- `@Target(ElementType.METHOD)`——方法
- `@Target(ElementType.PARAMETER)`——方法参数
- `@Target(ElementType.CONSTRUCTOR)` ——构造函数
- `@Target(ElementType.LOCAL_VARIABLE)`——局部变量
- `@Target(ElementType.ANNOTATION_TYPE)`——注解
- `@Target(ElementType.PACKAGE)`——包

我们来试一下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbud9C0eVziaLd0OF93vvfW790tic22ib0H80W5kTCjSjkibvcDL9PXWzibvYT74W5LEQQb8UibBpWF6PGNiaw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

目标不对会报错的哦！我们将其改成方法上！编译即正常通过。

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbud9C0eVziaLd0OF93vvfW790IkaiaTWDnu9tviac6RjRNkRY36XTC4ic55Qn6UL2G5xLW3tj3RibZf6D2w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

其他的作用域大家可以去自行尝试，篇幅问题，无法做到每个都去试一遍！

#### 4、@Inherited 继承

概念：标记这个注解是继承于哪个注解类(默认 注解并没有继承于任何子类)。

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbud9C0eVziaLd0OF93vvfW790p11KfIbnDzqiabYpO9VWeHyn70kCkE2jpjm5QTSEE93ibhACfmiaVRL1g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

这个很简单，就是当@InheritedAnno注解加在某个类A上时，假如类B继承了A，则B也会带上该注解。

#### 5、新注解-（了解即可）

从 Java 7 开始，额外添加了 3 个注解:

- `@SafeVarargs` - Java 7 开始支持，忽略任何使用参数为泛型变量的方法或构造函数调用产生的警告。
- `@FunctionalInterface` - Java 8 开始支持，标识一个匿名函数或函数式接口。
- `@Repeatable` - Java 8 开始支持，标识某注解可以在同一个声明上使用多次。

#### 三、自定义注解

我们来定义一个属于自己的注解。

```
@Retention(value = RetentionPolicy.RUNTIME)
@Target(value = ElementType.METHOD)
@Inherited
@interface myAnnotation {
    String name() default "";
    int age() default 18;
    String like();
    String IDCard() default "";
}
```

格式：修饰符（pulic）+ @interface +注解名+ {参数等}

可利用default 设置默认值，设定了默认值后使用注解时不传值也不会报错，反之报错！

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbud9C0eVziaLd0OF93vvfW790icjhxzHXzA7UC9hgiaMwq08cnNvQibGPic1BkNxibJFdk3tUqw5zwLhmHgA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

我们只需要传没有默认值的参数即可。

如果不传则报错：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbud9C0eVziaLd0OF93vvfW790NiaBwricYpz2F2rYwOn1xDFfjJlk9Qog3HkWNOocnFAhJxib15so4wm3A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

### 总结

主要就是要注意元注解的使用，因为我们自定义注解时必须得用到！其实注解主要配合反射来用，在此就不展开来叙述了。



