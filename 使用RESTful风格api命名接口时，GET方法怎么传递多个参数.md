# 使用RESTful风格api命名接口时，GET方法怎么传递多个参数

在使用RESTful风格不同于普通借口命名的一点是，它规范使用/来表示资源之间的层级关系。

![图片](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqLNR52a3H5kppnKJvvUtEm3DWtxiaK85yASBzoibVnhKK0DPbOJ8kbs1447LWRibpTtbk2a9XdtMr5g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

对于普通形式命名的接口，假设需要传入lessonId、lessonType2个必选项，在controller传入参数时，可以写成下图的形式：

![图片](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqLNR52a3H5kppnKJvvUtEmd3qsKEeZoxKAkictFBxVVXn0VfqvV3cc68xDFfAYj91D0pogJMX0Fug/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

但是对于RESTful风格的url，传入参数是应该用到@PathVariable注解，示例解析如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqLNR52a3H5kppnKJvvUtEmp2PPRd8xTGu422qEFIiaz0qq5648pdicoaQH9LNnMwpLibAs085dnD9lQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

那么对于传入传入lessonId、lessonType2个必选项时，用`@PathVariable`注解的话，应该写成：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

调用之后的结果显示成功。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

但是，还有一种情况，假如传入的参数有的是非必选项，那么应该把可能出现的url在声明时全部列出来，并且把`@PathVariable`注解配置的参数设置为非必选（required=false），如下图：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

调用之后结果如下：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

由此可见，对于RESTful风格的接口，当查询接口需要传入一个或者两个参数的时候，编码起来较为简单，但是当传入3个以上参数的手，要列举出url的所有可能性还是比较复杂的。所以，RESTful风格的接口传入参数比较复杂时，还是尽量使用POST方法比较简便。

*（感谢阅读，希望对你所有帮助)*

*来源：blog.csdn.net/qq_35075909/article/details/94005211*

```
最近给大家找了  ssm框架外卖订餐系统
资源，怎么领取？
扫二维码，加我微信，回复：ssm框架外卖订餐系统 注意，不要乱回复 
没错，不是机器人记得一定要等待，等待才有好东西

```