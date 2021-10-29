# PO,VO,DAO,BO,POJO 之间的区别

**VO**

value object：值对象。

通常用于业务层之间的数据传递，由new创建，由GC回收。

## **PO**

persistant object：持久层对象。

对应数据库中表的字段。VO 和 PO 都是属性加上属性的 get 和 set 方法；表面看没什么不同，但代表的含义是完全不同的。

## **DTO**

data transfer object：数据传输对象。

表里面有十几个字段：id,name,gender(M/F),age,conmpanyId(如001)...

页面需要展示四个字段：name,gender(男/女),age,conmpanyName(如今日头条股份有限公司)。

DTO由此产生，一是能提高数据传输的速度（减少了传输字段），二能隐藏后端表结构。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/SJm51egHPPE3IMYibAuD0dBUWA7JGib3tIMgpT14FBUFicbP41NnTHPsGcVDq0FUyKMnKk4jI52FEtVYMP84tIKkw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## **BO**

business object：业务对象。

BO 把业务逻辑封装为一个对象。我理解是 PO 的组合，比如投保人是一个 PO，被保险人是一个 PO，险种信息是一个 PO 等等，他们组合起来是第一张保单的 BO。

## **POJO**

plain ordinary java object：简单无规则 java 对象。

纯的传统意义的 java 对象，最基本的 Java Bean 只有属性加上属性的 get 和 set 方法。可以转化为 PO、DTO、VO；比如 POJO 在传输过程中就是 DTO。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/iarMRfJHmKibWEIvNrxicJibwXWKLibicFSWDhBt6OoKxAb1Infmq4c7RthUMdhzbn40gOb0giaoMnPGeaHzYsXHrxstA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## **DAO**

data access object：数据访问对象。

主要用来封装对数据的访问，注意，是对数据的访问，不是对数据库的访问。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/iarMRfJHmKibWEIvNrxicJibwXWKLibicFSWDhfl0BoCTndgBv2e2XgicCOYtYVCibqXgicbdoI0OLkEGdOV2O7iaIcnMslg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

