# 深入浅出Spring架构设计

为什么需要Spring? 什么是Spring?

对于这样的问题，大部分人都是处于一种朦朦胧胧的状态，说的出来，但又不是完全说的出来，今天我们就以架构设计的角度尝试解开Spring的神秘面纱。

本篇文章以由浅入深的方式进行介绍，大家不必惊慌，我可以保证，只要你会编程就能看懂。

> 本篇文章基于Spring 5.2.8，阅读时长大概需要20分钟

## 案例

我们先来看一个案例：有一个小伙，有一辆吉利车， 平常就开吉利车上班

![图片](https://mmbiz.qpic.cn/mmbiz_svg/X6Ucic5kYIBOGfYEz2saoxQLHVwXLVibOm3ocIv7O49og5qAibu4aIAhBsU01HEJCGDZS8sgDddPmNFEV1LNtXa6lubBOAhrE2g/640?wx_fmt=svg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

代码实现：

```
public class GeelyCar {

    public void run(){
        System.out.println("geely running");
    }
}
public class Boy {
  // 依赖GeelyCar
    private final GeelyCar geelyCar = new GeelyCar();

    public void drive(){
        geelyCar.run();
    }
}
```

有一天，小伙赚钱了，又买了辆红旗，想开新车。

简单，把依赖换成`HongQiCar`

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

代码实现：

```
public class HongQiCar {

    public void run(){
        System.out.println("hongqi running");
    }
}
public class Boy {
    // 修改依赖为HongQiCar
    private final HongQiCar hongQiCar = new HongQiCar();

    public void drive(){
        hongQiCar.run();
    }
}
```

新车开腻了，又想换回老车，这时候，就会出现一个问题：这个代码一直在改来改去

很显然，这个案例违背了我们的**依赖倒置原则(DIP)**:程序不应依赖于实现，而应依赖于抽象

### 优化后

现在我们对代码进行如下优化：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)img

`Boy`依赖于`Car`接口，而之前的`GeelyCar`与`HongQiCar`为`Car`接口实现

代码实现：

定义出Car接口

```
public interface Car {

    void run();
}
```

将之前的`GeelyCar`与`HongQiCar`改为`Car`的实现类

```
public class GeelyCar implements Car {

    @Override
    public void run(){
        System.out.println("geely running");
    }
}
```

> HongQiCar相同

Person此时依赖的为`Car`接口

```
public class Boy {
  // 依赖于接口
    private final Car car;
  
    public Person(Car car){
        this.car = car;
    }

    public void drive(){
        car.run();
    }
}
```

此时小伙想换什么车开，就传入什么参数即可，代码不再发生变化。

### 局限性

以上案例改造后看起来确实没有什么毛病了，但还是存在一定的局限性，如果此时增加新的场景：

有一天小伙喝酒了没法开车，需要找个代驾。代驾并不关心他给哪个小伙开车，也不关心开的是什么车，小伙就突然成了个抽象，这时代码又要进行改动了，代驾依赖小伙的代码可能会长这个样子：

```
private final Boy boy = new YoungBoy(new HongQiCar());
```

随着系统的复杂度增加，这样的问题就会越来越多，越来越难以维护，那么我们应当如何解决这个问题呢？

### 思考

首先，我们可以肯定：使用**依赖倒置原则**是没有问题的，它在一定程度上解决了我们的问题。

我们觉得出问题的地方是在传入参数的过程：程序需要什么我们就传入什么，一但系统中出现多重依赖的类关系，这个传入的参数就会变得极其复杂。

或许我们可以把思路**反转**一下：我们有什么，程序就用什么！

当我们只实现`HongQiCar`和`YoungBoy`时，代驾就使用的是开着`HongQiCar`的`YoungBoy`！

当我们只实现`GeelyCar`和`OldBoy`时，代驾自然而然就改变成了开着`GeelyCar`的`OldBoy`！

而如何反转，就是Spring所解决的一大难题。

## Spring介绍

Spring是一个一站式轻量级重量级的开发框架，目的是为了解决企业级应用开发的复杂性，它为开发Java应用程序提供全面的基础架构支持，让Java开发者不再需要关心类与类之间的依赖关系，可以专注的开发应用程序(crud)。

Spring为企业级开发提供给了丰富的功能，而这些功能的底层都依赖于它的两个核心特性:依赖注入(DI)和面向切面编程(AOP)。

### Spring的核心概念

#### IoC容器

IoC的全称为`Inversion of Control`，意为控制反转，IoC也被称为依赖性注入(DI)，这是一个通过依赖注入对象的过程：对象仅通过构造函数、工厂方法，或者在对象实例化在其上设置的属性来定义其依赖关系(即与它们组合的其他对象)，然后容器在创建bean时注入这些需要的依赖。这个过程从根本上说是Bean本身通过使用直接构建类或诸如服务定位模式的机制，来控制其依赖关系的实例化或位置的逆过程（因此被称为控制反转）。

> 依赖倒置原则是IoC的设计原理，依赖注入是IoC的实现方式。

#### 容器

在Spring中，我们可以使用XML、Java注解或Java代码的方式来编写配置信息，而通过配置信息，获取有关实例化、配置和组装对象的说明，进行实例化、配置和组装应用对象的称为容器。

一般情况下，我们只需要添加几个注解，这样容器进行创建和初始化后，我们就可以得到一个可配置的，可执行的系统或应用程序。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

#### Bean

在Spring中，由Spring IOC容器进行实例化—>组装管理—>构成程序骨架的对象称为Bean。Bean就是应用程序中众多对象之一。

> 以上三点串起来就是：Spring内部是一个放置Bean的IoC容器，通过依赖注入的方式处理Bean之间的依赖关系。

#### AOP

面向切面编程(Aspect-oriented Programming)，是相对面向对象编程(OOP)的一种功能补充,OOP面向的主要对象是类，而AOP则是切面。在处理日志、安全管理、事务管理等方面有非常重要的作用。AOP是Spring框架重要的组件，虽然IOC容器没有依赖AOP，但是AOP提供了非常强大的功能，用来对IOC做补充。

AOP可以让我们在不修改原有代码的情况下，对我们的业务功能进行增强：将一段功能切入到我们指定的位置，如在方法的调用链之间打印日志。

### Spring的优点

1、Spring通过DI、AOP来简化企业级Java开发

2、Spring的低侵入式设计，让代码的污染极低

3、Spring的IoC容器降低了业务对象之间的复杂性，让组件之间互相解耦

4、Spring的AOP支持允许将一些通用任务如安全、事务、日志等进行集中式处理，从而提高了更好的复用性

5、Spring的高度开放性，并不强制应用完全依赖于Spring，开发者可自由选用Spring框架的部分或全部

6、Spring的高度扩展性，让开发者可以轻易的让自己的框架在Spring上进行集成

7、Spring的生态极其完整，集成了各种优秀的框架，让开发者可以轻易的使用它们

> 我们可以没有Java，但是不能没有Spring~

## 用Spring改造案例

我们现在已经认识了什么是Spring，现在就尝试使用Spring对案例进行改造一下

原来的结构没有变化，只需在`GeelyCar`或`HongQiCar`上增加`@Component`注解，`Boy`在使用时加上`@Autowired`注解

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

代码样式：

```
@Componentpublic class GeelyCar implements Car { @Override public void run() {  System.out.println("geely car running"); }}
```

> HongQiCar相同
>
> 在Spring中，当类标识了@Component注解后就表示这是一个Bean，可以被IoC容器所管理

```
@Componentpublic class Boy {  // 使用Autowired注解表示car需要进行依赖注入 @Autowired private Car car; public void driver(){  car.run(); }}
```

> 我们之前所说的：我们实现什么，程序就使用什么，在这里就等同于我们在哪个类上标识了`Component`注解，哪个类就会是一个Bean，Spring就会使用它注入`Boy`的属性`Car`中
>
> 所以当我们给`GeelyCar`标识`Component`注解时，`Boy`开的车就是`GeelyCar`，当我们给`HongQiCar`标识`Component`注解时，`Boy`开的车就是`HongQiCar`
>
> 当然，我们不可以在`GeelyCar`和`HongQiCar`上同时标识`Component`注解，因为这样Spring就不知道用哪个`Car`进行注入了——Spring也有选择困难症（or 一boy不能开俩车？）

使用Spring启动程序

```
// 告诉Spring从哪个包下扫描Bean，不写就是当前包路径@ComponentScan(basePackages = "com.my.spring.test.demo")public class Main { public static void main(String[] args) {  // 将Main(配置信息)传入到ApplicationContext(IoC容器)中  ApplicationContext context = new AnnotationConfigApplicationContext(Main.class);  // 从(IoC容器)中获取到我们的boy  Boy boy = (Boy) context.getBean("boy");  // 开车  boy.driver(); }}
```

> 这里就可以把我们刚刚介绍Spring的知识进行解读了
>
> 把具有`ComponentScan`注解(配置信息)的`Main`类，传给`AnnotationConfigApplicationContext`(IoC容器)进行初始化，就等于：IoC容器通过获取配置信息进行实例化、管理和组装Bean。

而如何进行依赖注入则是在IoC容器内部完成的，这也是本文要讨论的重点

## 思考

我们通过一个改造案例完整的认识了Spring的基本功能，也对之前的概念有了一个具象化的体验，而我们还并不知道Spring的依赖注入这一内部动作是如何完成的，所谓知其然更要知其所以然，结合我们的现有知识，以及对Spring的理解，大胆猜想推测一下吧(这是很重要的能力哦)

其实猜测就是指：如果让我们自己实现，我们会如何实现这个过程？

首先，我们要清楚我们需要做的事情是什么：扫描指定包下面的类，进行实例化，并根据依赖关系组合好。

步骤分解：

扫描指定包下面的类 -> 如果这个类标识了Component注解(是个Bean) -> 把这个类的信息存起来

进行实例化 -> 遍历存好的类信息 -> 通过反射把这些类进行实例化

根据依赖关系组合 -> 解析类信息 -> 判断类中是否有需要进行依赖注入的字段 -> 对字段进行注入

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

## 方案实现

我们现在已经有了一个看起来像是那么一回事的解决方案，现在就尝试把这个方案实现出来

### 定义注解

首先我们需要定义出需要用到的注解：`ComponentScan`,`Component`,`Autowired`

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface ComponentScan {

    String basePackages() default "";
}
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Component {

    String value() default "";
}
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Autowired {
}
```

### 扫描指定包下面的类

扫描指定包下的所有类，听起来好像一时摸不着头脑，其实它等同于另一个问题：如何遍历文件目录？

那么存放类信息应该用什么呢？我们看看上面例子中`getBean`的方法，是不是像Map中的通过key获取value？而Map中还有很多实现，但线程安全的却只有一个，那就是`ConcurrentHashMap`(别跟我说HashTable)

定义存放类信息的map

```
private final Map<String, Class<?>> classMap = new ConcurrentHashMap<>(16);
```

具体流程，下面同样附上代码实现：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

代码实现，可以与流程图结合观看：

扫描类信息

```
private void scan(Class<?> configClass) {
  // 解析配置类，获取到扫描包路径
  String basePackages = this.getBasePackages(configClass);
  // 使用扫描包路径进行文件遍历操作
  this.doScan(basePackages);
}
private String getBasePackages(Class<?> configClass) {
  // 从ComponentScan注解中获取扫描包路径
  ComponentScan componentScan = configClass.getAnnotation(ComponentScan.class);
  return componentScan.basePackages();
}
private void doScan(String basePackages) {
  // 获取资源信息
  URI resource = this.getResource(basePackages);

  File dir = new File(resource.getPath());
  for (File file : dir.listFiles()) {
    if (file.isDirectory()) {
      // 递归扫描
      doScan(basePackages + "." + file.getName());
    }
    else {
      // com.my.spring.example + . + Boy.class -> com.my.spring.example.Boy
      String className = basePackages + "." + file.getName().replace(".class", "");
      // 将class存放到classMap中
      this.registerClass(className);
    }
  }
}
private void registerClass(String className){
  try {
    // 加载类信息
    Class<?> clazz = classLoader.loadClass(className);
    // 判断是否标识Component注解
    if(clazz.isAnnotationPresent(Component.class)){
      // 生成beanName com.my.spring.example.Boy -> boy
      String beanName = this.generateBeanName(clazz);
      // car: com.my.spring.example.Car
      classMap.put(beanName, clazz);
    }
  } catch (ClassNotFoundException ignore) {}
}
```

### 实例化

现在已经把所有适合的类都解析好了，接下来就是实例化的过程了

定义存放Bean的Map

```
private final Map<String, Object> beanMap = new ConcurrentHashMap<>(16);
```

具体流程，下面同样给出代码实现：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

代码实现，可以与流程图结合观看：

遍历`classMap`进行实例化Bean

```
public void instantiateBean() {
  for (String beanName : classMap.keySet()) {
    getBean(beanName);
  }
}
public Object getBean(String beanName){
  // 先从缓存中获取
  Object bean = beanMap.get(beanName);
  if(bean != null){
    return bean;
  }
  return this.createBean(beanName);
}
private Object createBean(String beanName){
  Class<?> clazz = classMap.get(beanName);
  try {
    // 创建bean
    Object bean = this.doCreateBean(clazz);
    // 将bean存到容器中
    beanMap.put(beanName, bean);
    return bean;
  } catch (IllegalAccessException e) {
    throw new RuntimeException(e);
  }
}
private Object doCreateBean(Class<?> clazz) throws IllegalAccessException {  // 实例化bean  Object bean = this.newInstance(clazz);  // 填充字段，将字段设值  this.populateBean(bean, clazz);  return bean;}
private Object newInstance(Class<?> clazz){  try {    // 这里只支持默认构造器    return clazz.getDeclaredConstructor().newInstance();  } catch (InstantiationException | IllegalAccessException | InvocationTargetException | NoSuchMethodException e) {    throw new RuntimeException(e);  }}
private void populateBean(Object bean, Class<?> clazz) throws IllegalAccessException {  // 解析class信息，判断类中是否有需要进行依赖注入的字段  final Field[] fields = clazz.getDeclaredFields();  for (Field field : fields) {    Autowired autowired = field.getAnnotation(Autowired.class);    if(autowired != null){      // 获取bean      Object value = this.resolveBean(field.getType());      field.setAccessible(true);      field.set(bean, value);    }  }}
private Object resolveBean(Class<?> clazz){  // 先判断clazz是否为一个接口，是则判断classMap中是否存在子类  if(clazz.isInterface()){    // 暂时只支持classMap只有一个子类的情况    for (Map.Entry<String, Class<?>> entry : classMap.entrySet()) {      if (clazz.isAssignableFrom(entry.getValue())) {        return getBean(entry.getValue());      }    }    throw new RuntimeException("找不到可以进行依赖注入的bean");  }else {    return getBean(clazz);  }}
public Object getBean(Class<?> clazz){
  // 生成bean的名称
  String beanName = this.generateBeanName(clazz);
  // 此处对应最开始的getBean方法
  return this.getBean(beanName);
}
```

### 组合

两个核心方法已经写好了，接下把它们组合起来，我把它们实现在自定义的`ApplicationContext`类中，构造方法如下：

```
public ApplicationContext(Class<?> configClass) {
  // 1.扫描配置信息中指定包下的类
  this.scan(configClass);
  // 2.实例化扫描到的类
  this.instantiateBean();
}
```

UML类图：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

### 测试

代码结构与案例相同，这里展示一下我们自己的Spring是否可以正常运行

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

> 运行正常，中国人不骗中国人
>
> 源码会在文末给出

## 回顾

现在，我们已经根据设想的方案进行了实现，运行的情况也达到了预期的效果。但如果仔细研究一下，再结合我们平常使用Spring的场景，就会发现这一份代码的不少问题：

1、无法支持构造器注入，当然也没有支持方法注入，这是属于功能上的缺失。

2、加载类信息的问题，加载类时我们使用的是`classLoader.loadClass`的方式，虽然这避免了类的初始化(可千万别用`Class.forName`的方式)，但还是不可避免的把类元信息加载到了元空间中，当我们扫描包下有不需要的类时，这就浪费了我们的内存。

3、无法解决bean之间的循环依赖，比如有一个A对象依赖了B对象， B对象又依赖了A对象，这个时候我们再来看看代码逻辑，就会发现此时会陷入死循环。

4、扩展性很差，我们把所有的功能都写在一个类里，当想要完善功能(比如以上3个问题)时，就需要频繁修改这个类，这个类也会变得越来越臃肿，别说迭代新功能，维护都会令人头疼。

## 优化方案

对于前三个问题都类似于功能上的问题，功能嘛，改一改就好了。

我们需要着重关注的是第四个问题，一款框架想要变得优秀，那么它的迭代能力一定要好，这样功能才能变得丰富，而迭代能力的影响因素有很多，其中之一就是它的扩展性。

那么应该如何提高我们的方案的扩展性呢，六大设计原则给了我们很好的指导作用。

在方案中，`ApplicationContext`做了很多事情， 主要可以分为两大块

1、扫描指定包下的类

2、实例化Bean

#### 借助单一职责原则的思想：一个类只做一种事，一个方法只做一件事。

我们把**扫描指定包下的类**这件事单独使用一个处理器进行处理，因为扫描配置是从配置类而来，那我们就叫他配置类处理器：ConfigurationCalssProcessor

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

实例化Bean这件事情也同样如此，实例化Bean又分为了两件事：实例化和依赖注入

实例化Bean就是相当于一个生产Bean的过程，我们就把这件事使用一个工厂类进行处理，它就叫做：BeanFactory，既然是在生产Bean，那就需要原料(Class)，所以我们把`classMap`和`beanMap`都定义到这里

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

而依赖注入的过程，其实就是在处理`Autowired`注解，那它就叫做: AutowiredAnnotationBeanProcessor

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

我们还在知道，在Spring中，不仅仅只有这种使用方式，还有xml，mvc，SpringBoot的方式，所以我们将`ApplicationContext`进行抽象，只实现主干流程，原来的注解方式交由`AnnotationApplicationContext`实现。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

#### 借助依赖倒置原则：程序应当依赖于抽象

在未来，类信息不仅仅可以从类信息来，也可以从配置文件而来，所以我们将ConfigurationCalssProcessor抽象

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

而依赖注入的方式不一定非得是用`Autowried`注解标识，也可以是别的注解标识，比如`Resource`，所以我们将AutowiredAnnotationBeanProcessor抽象

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

Bean的类型也可以有很多，可以是单例的，可以使多例的，也可以是个工厂Bean，所以我们将BeanFactory抽象

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

现在，我们借助两大设计原则对我们的方案进行了优化，相比于之前可谓是”脱胎换骨“。

## Spring的设计

在上一步，我们实现了自己的方案，并基于一些设想进行了扩展性优化，现在，我们就来认识一下实际上Spring的设计

那么，在Spring中又是由哪些"角色"构成的呢？

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

1、Bean: Spring作为一个IoC容器，最重要的当然是Bean咯

2、BeanFactory: 生产与管理Bean的工厂

3、BeanDefinition: Bean的定义，也就是我们方案中的Class，Spring对它进行了封装

4、BeanDefinitionRegistry: 类似于Bean与BeanFactory的关系，BeanDefinitionRegistry用于管理BeanDefinition

5、BeanDefinitionRegistryPostProcessor: 用于在解析配置类时的处理器，类似于我们方案中的ClassProcessor

6、BeanFactoryPostProcessor: BeanDefinitionRegistryPostProcessor父类，让我们可以再解析配置类之后进行后置处理

7、BeanPostProcessor: Bean的后置处理器，用于在生产Bean的过程中进行一些处理，比如依赖注入，类似我们的AutowiredAnnotationBeanProcessor

8、ApplicationContext: 如果说以上的角色都是在工厂中生产Bean的工人，那么ApplicationContext就是我们Spring的门面，ApplicationContext与BeanFactory是一种组合的关系，所以它完全扩展了BeanFactory的功能，并在其基础上添加了更多特定于企业的功能，比如我们熟知的ApplicationListener(事件监听器)

> 以上说的类似其实有一些本末倒置了，因为实际上应该是我们方案中的实现类似于Spring中的实现，这样说只是为了让大家更好的理解
>
> 我们在经历了自己方案的设计与优化后，对这些角色其实是非常容易理解的

接下来，我们就一个一个的详细了解一下

### BeanFactory

BeanFactory是Spring中的一个顶级接口，它定义了获取Bean的方式，Spring中还有另一个接口叫SingletonBeanRegistry，它定义的是操作单例Bean的方式，这里我将这两个放在一起进行介绍，因为它们大体相同，SingletonBeanRegistry的注释上也写了可以与BeanFactory接口一起实现，方便统一管理。

##### BeanFactory

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

1、ListableBeanFactory：接口，定义了获取Bean/BeanDefinition列表相关的方法，如`getBeansOfType(Class type)`

2、AutowireCapableBeanFactory：接口，定义了Bean生命周期相关的方法，如创建bean, 依赖注入，初始化

3、AbstractBeanFactory：抽象类，基本上实现了所有有关Bean操作的方法，定义了Bean生命周期相关的抽象方法

4、AbstractAutowireCapableBeanFactory：抽象类，继承了AbstractBeanFactory，实现了Bean生命周期相关的内容，虽然是个抽象类，但它没有抽象方法

5、DefaultListableBeanFactory：继承与实现以上所有类和接口，是为Spring中最底层的BeanFactory, 自身实现了ListableBeanFactory接口

6、ApplicationContext：也是一个接口，我们会在下面有专门对它的介绍

##### SingletonBeanRegistry

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

1、DefaultSingletonBeanRegistry： 定义了Bean的缓存池，类似于我们的BeanMap，实现了有关单例的操作，比如`getSingleton`(面试常问的三级缓存就在这里)

2、FactoryBeanRegistrySupport：提供了对FactoryBean的支持，比如从FactoryBean中获取Bean

### BeanDefinition

BeanDefinition其实也是个接口(想不到吧)，这里定义了许多和类信息相关的操作方法，方便在生产Bean的时候直接使用，比如`getBeanClassName`

它的大概结构如下(这里举例RootBeanDefinition子类)：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

> 里面的各种属性想必大家也绝不陌生

同样的，它也有许多实现类：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

1、AnnotatedGenericBeanDefinition：解析配置类与解析Import注解带入的类时，就会使用它进行封装

2、ScannedGenericBeanDefinition：封装通过@ComponentScan扫描包所得到的类信息

3、ConfigurationClassBeanDefinition：封装通过@Bean注解所得到的类信息

4、RootBeanDefinition：ConfigurationClassBeanDefinition父类，一般在Spring内部使用，将其他的BeanDefition转化成该类

### BeanDefinitionRegistry

定义了与BeanDefiniton相关的操作，如`registerBeanDefinition`，`getBeanDefinition`，在BeanFactory中，实现类就是DefaultListableBeanFactory

### BeanDefinitionRegistryPostProcessor

插话：讲到这里，有没有发现Spring的命名极其规范，Spring团队曾言Spring中的类名都是反复推敲才确认的，真是名副其实呀，所以看Spring源码真的是一件很舒服的事情，看看类名方法名就能猜出它们的功能了。

该接口只定义了一个功能：处理BeanDefinitonRegistry，也就是解析配置类中的`Import`、`Component`、`ComponentScan`等注解进行相应的处理，处理完毕后将这些类注册成对应的`BeanDefinition`

在Spring内部中，只有一个实现：ConfigurationClassPostProcessor

### BeanFactoryPostProcessor

所谓BeanFactory的后置处理器，它定义了在解析完配置类后可以调用的处理逻辑，类似于一个插槽，如果我们想在配置类解析完后做点什么，就可以实现该接口。

在Spring内部中，同样只有ConfigurationClassPostProcessor实现了它：用于专门处理加了`Configuration`注解的类

这里串场一个小问题，如知以下代码：

```
@Configuraiton
public class MyConfiguration{
  @Bean
  public Car car(){
      return new Car(wheel());
  }
  @Bean
  public Wheel wheel(){
      return new Wheel();
  }
}
```

问：Wheel对象在Spring启动时，被new了几次？为什么？

### BeanPostProcessor

江湖翻译：Bean的后置处理器

该后置处理器贯穿了Bean的生命周期整个过程，在Bean的创建过程中，一共被调用了9次，至于哪9次我们下次再来探究，以下介绍它的实现类以及作用

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

1、AutowiredAnnotationBeanPostProcessor：用于推断构造器进行实例化，以及处理Autowired和Value注解

2、CommonAnnotationBeanPostProcessor：处理Java规范中的注解，如Resource、PostConstruct

3、ApplicationListenerDetector: 在Bean的初始化后使用，将实现了ApplicationListener接口的bean添加到事件监听器列表中

4、ApplicationContextAwareProcessor：用于回调实现了Aware接口的Bean

5、ImportAwareBeanPostProcessor: 用于回调实现了ImportAware接口的Bean

### ApplicationContext

ApplicationContext作为Spring的核心，以门面模式隔离了BeanFactory，以模板方法模式定义了Spring启动流程的骨架，又以策略模式调用了各式各样的Processor......实在是错综复杂又精妙绝伦!

它的实现类如下：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

1、ConfigurableApplicationContext：接口，定义了配置与生命周期相关操作，如refresh

2、AbstractApplicationContext: 抽象类，实现了refresh方法，refresh方法作为Spring核心中的核心,可以说整个Spring皆在refresh之中，所有子类都通过refresh方法启动，在调用该方法之后，将实例化所有单例

3、AnnotationConfigApplicationContext: 在启动时使用相关的注解读取器与扫描器，往Spring容器中注册需要用的处理器，而后在refresh方法在被主流程调用即可

4、AnnotationConfigWebApplicationContext：实现loadBeanDefinitions方法，以期在refresh流程中被调用，从而加载BeanDefintion

5、ClassPathXmlApplicationContext: 同上

> 从子类的情况可以看出，子类的不同之处在于如何加载BeanDefiniton, AnnotationConfigApplicationContext是通过配置类处理器(ConfigurationClassPostProcessor)加载的，而AnnotationConfigWebApplicationContext与ClassPathXmlApplicationContext则是通过自己实现loadBeanDefinitions方法，其他流程则完全一致

## Spring的流程

以上，我们已经清楚了Spring中的主要角色以及作用，现在我们尝试把它们组合起来，构建一个Spring的启动流程

同样以我们常用的AnnotationConfigApplicationContext为例

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

> 图中只画出了Spring中的部分大概流程，详细内容我们会在后面的章节展开

## 小结

所谓万事开头难，本文初衷就是能让大家以由浅入深的方式认识Spring，初步建立Spring的认知体系，明白Spring的内部架构，对Spring的认知不再浮于表面。

现在头已经开了，相信后面内容的学习也将水到渠来。

本篇文章既讲是Spring的架构设计，也希望能成为我们以后复习Spring整体内容时使用的手册。

最后，看完文章之后，相信对以下面试常问的问题回答起来也是轻而易举

1、什么是BeanDefinition？

2、BeanFactory与ApplicationContext的关系？

3、后置处理器的分类与作用？

4、Spring的主要流程是怎么样的？