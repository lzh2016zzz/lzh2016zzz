---
tags: 
   - java
   - bug排查
article_header:
  type: overlay
  theme: dark
  background_color: '#203028'
  background_image:
    gradient: 'linear-gradient(135deg, rgba(34, 139, 87 , .4), rgba(139, 34, 139, .4))'
    src: /assets/images/cover4.jpg
---

`ArrayList.add()`报`java.lang.UnsupportedOperationException`的问题.

<!--more-->

看看`Arrays.asList()`的实现:

```java
  @SafeVarargs
  @SuppressWarnings("varargs")
  public static <T> List<T> asList(T... a) {
      return new ArrayList<>(a);
  }
```

看起来好像没啥问题.但是这个`ArrayList`不是`java.util`下的,而是`Arrays`的一个内部类.只实现了部分获取内部元素的方法.

想要要执行添加或者删除方法的话,还是自己new一个`ArrayList`比较好吧.

