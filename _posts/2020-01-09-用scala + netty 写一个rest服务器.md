---
title: 用scala + netty 写一个rest服务器
tags: 
   - java
   - 技术
---



<!--more-->
作为一个Programmer,要多多尝试新的技术.

老实说,一开始让我用scala写代码的时候,我是拒绝的.

因为之前听过的关于scala的传说里面,最让我印象深刻的就是它的难度.据说scala是"全宇宙唯一一个复杂度超过c++的预言".

听起来是很厉害的样子.但是,我又实在是忍不住想尝试一下传说中的函数式是一等公民的语言是什么样的.所以忍不住尝试用scala + netty 写了一个rest服务器..

写完以后,我的感想是...其实也没有也没那么复杂啦.抛开那些繁杂的特性,单纯的把scala当作lambda强化版的java,也是相当不错的.


核心类`Dispatcher.scala` 只用了七十多行代码就实现了url路由功能

```scala
@Singleton
class Dispatcher {


  private[this] val log = LoggerFactory.getLogger(this.getClass)

  private[this] val allocator = UnpooledByteBufAllocator.DEFAULT

  private[this] var mapping: mutable.Set[MappingCase] = _

  @Inject
  def this(scanner: MappingScanner) {
    this()
    mapping = scanner.routerMapping(Constant.mappingPackage)
  }


  def dispatch(req: FullHttpRequest): Option[DefaultFullHttpResponse] = {
    log.debug("dispatch uri :{}", req.uri())
    mapping
      .find(x => x.pattern.matches(req.uri()) && x.method == req.method.toString.toUpperCase)
      .map(_.mappings)
      .flatMap(routingTo(req, _))
      .orElse(strResponse(str = Constant.NOT_FOUND, status = HttpResponseStatus.NOT_FOUND))
  }

  private def routingTo(req: FullHttpRequest, mapping: Mappings): Option[DefaultFullHttpResponse] = {
     try {
       mapping.execute(req).flatMap {
         case str: String => strResponse(str)
         case bytes: Array[Byte] => response(allocator.heapBuffer(bytes.length).writeBytes(bytes), contentType = "application/octet-stream")
         case ref: AnyRef => Option(ref).map(JSON.toJSONString(_)).flatMap(strResponse(_, contentType = "application/json"))
         case _ => strResponse("null")
       }
     } catch {
       case e: Exception =>
         log.error("execute mapping error:\r\n", e)
         strResponse(str = Option(e.getMessage).getOrElse("exception with 417"), status = HttpResponseStatus.EXPECTATION_FAILED)
     }
   }
...
```



虽然scala可以直接使用java的库,但是,很多小细节都在提醒我,它是`scala`.

比如,`scala`的集合类是另外实现的一套,和java的集合类不兼容.

默认情况下 scala调用java方法 不支持协变.比如 调用`add(Collection<T>)`时不能用`List<T>`作为参数.

`scala`的所有算术操作符都是函数.可以任意重载.

`scala`支持元编程.可以自己实现一套基于scala的方言.同样是基于jvm的java却做不到这点.



总体来说,我认为scala的很多特性都很棒.用这些特性写代码会很舒服.能让代码更精简 和 减少很多低级错误.

scala比java更强大,更超前. 有很多特性 是scala 先有 然后 java 才跟进的.比如多行字符串.lambda表达式 等等.

说scala比java超前 5 -10 年我觉得一点都不过分. 



但是,scala不太适用于java web.强行使用scala写java web会比较痛苦.

比如当你遇到集合类的转换问题 和调用java方法不支持协变的问题的时候.为了处理好这些问题,写出来的代码一点都不优雅,甚至可以说很丑陋.



相对于scala而言, 能够和java无缝对接的 kotlin更适合用来写java web. kotlin是介于scala 和 java之间的语言.

按照编程语言能力来划分 kotlin的特性应该属于scala的子集.

有时间的话再来聊聊kotlin吧.



代码 : 

[https://gitee.com/minagamiyuki/my-scala-dispatcher](https://gitee.com/minagamiyuki/my-scala-dispatcher)


