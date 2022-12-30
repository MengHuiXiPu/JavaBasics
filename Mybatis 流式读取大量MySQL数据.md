# Mybatis 流式读取大量MySQL数据

#### JDBC三种读取方式：

**1、** 一次全部（默认）：一次获取全部；
**2、** 流式：多次获取，一次一行；
**3、** 游标：多次获取，一次多行；

> mybatis默认采取第一种。

#### 开发环境：

jdk1.8 、intellij IDEA 2018

mybatis 3 、 springMVC 、Spring 4

## 实现步骤：

实现流式读取的方式不止一种，但是我只能说我解决的这种，对不起，我不是大神级的。

这里采用的 controller、service、dao分层开发

- 在service层调用dao接口是，增加一个回调参数 ResultHandler`<>`。
- 对应的dao接口返回值为void
- mapper 填写 parameterType、resultMap、 resultSetType=“FORWARD_ONLY”、 fetchSize="-2147483648"

为什么dao返回值为void还要在mapper写resultMap?因为回调要用你的resultMap处理，但是不应该返回给service，因为回调处理好了

## 示例代码

controller层：

```
@RequestMapping("/export")
  public void export(Vo vo, String props,
      HttpServletResponse response) {

    //.......
      list = ossVipCustomService.selectForwardOnly(vo, Order.build());
    //......
  }
```

> [我推荐一套，架构师视频 155G 太全了
> https://mp.weixin.qq.com/s/PoWsCNFAe_MlV_oANsfBT](https://mp.weixin.qq.com/s?__biz=MzI4NTM1NDgwNw==&mid=2247532851&idx=1&sn=59739b7a60aa5855a4a4c22bd94cbd47&scene=21#wechat_redirect)

service层：(重点)

```
public List<Bo> selectForwardOnly(Vo vo, Order order) {
     final List<Bo> list = new ArrayList<>();
    mapper.selectForwardOnly(vo, order, new ResultHandler<Bo>() {
      @Override
      public void handleResult(ResultContext<? extends Bo> resultContext) {
        /**回调处理逻辑 */
        list.add(resultContext.getResultObject());
      }
  });
    return list;
  }
```

dao层：(重点)

```
  /**
   * 流式读取数据
   * @param vo 查询对象
   * @param order 排序
   * @param ossVipCustomerBoResultHandler 回调处理
   */
  void selectForwardOnly(@Param("record") Vo vo, @Param("order") Order order,
      ResultHandler<Bo> handler);
```

mapper:(重点)

```
<select id="selectForwardOnly"
  parameterType="com.*.Vo" resultMap="GetListBo"
    resultSetType="FORWARD_ONLY" fetchSize="-2147483648">
    SELECT
    *
    FROM
    customer
  </select>
```

个人原因：删除非关键部分代码。你肯定看的懂得。

