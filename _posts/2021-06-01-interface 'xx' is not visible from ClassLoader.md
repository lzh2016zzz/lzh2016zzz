---
title: interface 'xx' is not visible from ClassLoader
tags: 
   - java
   - 经验
---

## interface 'xx' is not visible from ClassLoader问题处理


之前的代码里,在factoryBean中使用如下方式创建代理对象:

```java
        @SuppressWarnings("unchecked")
	@Override
	public T getObject() throws Exception {

		return (T) Proxy.newProxyInstance(
                        this.getClass().getClassLoader(),//错误的方式
                        new Class[] { serviceClientInterface },
                        new ServiceProxy<>(interfaceConfig, methodConfigs));

	}
```

如果是web应用程序，那么在创建动态代理时应该使用web应用程序类加载器.类似这样的:

```java
                Proxy.newProxyInstance(
                  ClassLoader.getSystemClassLoader(),
                  new Class < ? >[] {MyInterface.class},
                  new InvocationHandler() {
                    // (...)
                });
```


## 原因分析

tomcat classloader的层次结构（其他web容器也有类似的结构）如下所示：

```
      Bootstrap
          |
       System
          |
       Common
       /     \
  Webapp1   Webapp2 ... 
```

 WebappClassloader在最底层，它包含WEB应用程序的/WEB-INF/classes目录中的类和资源，以及WEB应用程序的/WEB-INF/lib目录下JAR文件中的类和资源.

若使用 WebappClassloader加载类,可能会出现同一个类被加载多次的情况.此时代理类无法确认哪一个类是正确的.就会导致上面的问题.修改类加载器为 `ClassLoader.getSystemClassLoader()`可以解决这个问题.
