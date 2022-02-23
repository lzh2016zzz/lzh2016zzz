---
tags: 
   - java
---

Java8引入了Stream API，它使得将集合作为数据流进行迭代变非常很容易。创建并行执行并利用多个CPU的流也非常容易.
我们可能会认为，并行执行任务一定比串行执行快,但情况往往并非如此.这篇文章分析了常见情况下使用流对性能的影响.以及什么情况下使用流能提升性能.

## java中的流
Java中的流(`stream`)只是数据源的包装器，它封装了一系列对数据管道上的函数式操作的支持,允许我们以方便的方式对数据执行批量操作.
流的特性: 
 - 不存储数据，也不对数据源进行任何更改.每次调用`stream()`或者`parallelStream()`都会生成一份新的数据.不会影响到原有的数据集合.不需要担心操作原有对象带来的副作用.
 - 延迟执行.只有调用最终操作时,才会调用中间流程的代码.比如这段代码是不会执行的:
     ```java
     public void testJavaStream() {
        List<Integer> rs = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 0);
        //改为 : rs.parallelStream().map(String::valueOf).collect(Collectors.toList()) 才能执行
        //最终输出List<String>
        rs.parallelStream().map(String::valueOf);
     }
     ```

### 顺序流

默认情况下，Java中的任何流操作都是按顺序处理的，除非明确指定为并行流(`parallelStream()`)
顺序流使用单个线程来处理数据:
```java
List<Integer> listOfNumbers = Arrays.asList(1, 2, 3, 4);
listOfNumbers.stream().forEach(number ->
    System.out.println(number + " " + Thread.currentThread().getName())
);
```
这个连续流会按照顺序输出'1 main','2 main','3 main','4 main':

```bash
1 main
2 main
3 main
4 main
```

### 并行流

java中的任何流都可以简单的转化为并行流.通过调用`parallelStream()`就能实现:
```java
List<Integer> listOfNumbers = Arrays.asList(1, 2, 3, 4);
listOfNumbers.parallelStream().forEach(number ->
    System.out.println(number + " " + Thread.currentThread().getName())
);
```
并行流可以让我们简单的将流的执行过程并行化.然而,执行顺序是乱序的.每次运行程序时,它都有可能发生变化:
```bash
4 ForkJoinPool.commonPool-worker-3
2 ForkJoinPool.commonPool-worker-5
1 ForkJoinPool.commonPool-worker-7
3 main
```

### fork-join 线程池
并行流使用了fork-join 线程池作为它的工作线程.相对于普通线程池而言它使用了工作窃取算法.空闲的线程会去其他的线程的任务队列`窃取`任务.防止线程池最大负载之前耗时的任务全部堆积给了单个线程.默认情况下全局共用一个fork-join线程池.线程池的线程数量 = CPU核心数量.

可以通过JVM参数指定工作线程的数量:
```
-D java.util.concurrent.ForkJoinPool.common.parallelism=4
```

实际上,调用者自己算一个额外的工作线程.所以实际并行数 = 配置的工作线程数量 +1 

### 自定义线程池

除了默认的公共线程池外，还可以在自定义线程池中运行并行流:

```java
List<Integer> listOfNumbers = Arrays.asList(1, 2, 3, 4);
ForkJoinPool customThreadPool = new ForkJoinPool(5);
int sum = customThreadPool.submit(
    () -> listOfNumbers.parallelStream().reduce(0, Integer::sum)).get();
customThreadPool.shutdown();
assertThat(sum).isEqualTo(10);
```

这个代码使用了独立的自定义的`ForkJoinPool`.线程数量 = 5.

使用完线程池以后记得通过customThreadPool.shutdown()关闭.否则可能导致线程泄露
{:.warning}

### 分割数据源的副作用
fork-join框架负责在工作线程之间拆分源数据，并在任务完成时处理回调.这个操作在特定情况下可能导致副作用.比如下面这个例子,我们使用了`reduce()`方法,初始值 = 5,迭代元素并且在初始值上加上每个元素的值:
```java
   public static void main(String[] args) {
        List<Integer> listOfNumbers = Arrays.asList(1, 2, 3, 4);
        int sum = listOfNumbers.stream().reduce(5, (a,b) -> a + b);
        System.out.println(sum);
    }
```
期望的输出值是1+2+3+4+5=15,但由于`reduce()`操作是并行处理的.它在每次迭代时都会把初始值拷贝一份过去,最后再累加起来,所以实际的输出值是15 + 5*n(n=流中的元素数量) = 30.因此,我们在使用并行流时需要特别小心哪些对象是可以并行操作的.


### 对性能的影响

并行处理可以充分利用多个CPU的性能对集合进行处理.但也要考虑线程同步,拷贝数据,合并结果的开销.

#### 简单操作的开销对比

```
IntStream.rangeClosed(1, 100).reduce(0, Integer::sum);
IntStream.rangeClosed(1, 100).parallel().reduce(0, Integer::sum);
```
运行结果:
```
Benchmark                         Mode  Cnt        Score        Error  Units
Costs.parallelstream              avgt   25      35476,283 ±     204,446  ns/op
Costs.seqStream                   avgt   25         68,274 ±       0,963  ns/op
```
可以看出有些时候管理线程,源,合并结果的开销要比计算还要大.因此,对于计算量比较小的操作不应该使用并行流.

#### 数据结构对并行操作的性能影响

拆分数据源是并行执行的必要条件，但数据源的类型对性能也会造成很大的影响.通过`ArrayList`和`LinkedList`来演示这点:

```java
private static final List<Integer> arrayListOfNumbers = new ArrayList<>();
private static final List<Integer> linkedListOfNumbers = new LinkedList<>();
...
arrayListOfNumbers.stream().reduce(0, Integer::sum)
arrayListOfNumbers.parallelStream().reduce(0, Integer::sum);
linkedListOfNumbers.stream().reduce(0, Integer::sum);
linkedListOfNumbers.parallelStream().reduce(0, Integer::sum);
```
结果:
```
Benchmark                          Mode  Cnt        Score        Error  Units
Costs.arrayListParallelstream     avgt   25    2004849,711 ±    5289,437  ns/op
Costs.arrayListSeq                avgt   25    5437923,224 ±   37398,940  ns/op
Costs.linkedListParallelstream    avgt   25   13561609,611 ±  275658,633  ns/op
Costs.linkedListSeq               avgt   25   10664918,132 ±  254251,184  ns/op
```

可以看出.只有`ArrayList`在使用`parallelstream`时性能更好了,这是合理的.原因是`ArrayList`底层是数组,支持随机访问.可以在不遍历的情况下对其进行拆分.
此外造成性能差异的另一个因素是数组访问可以借助CPU的缓存机制进行加速,在访问连续的内存空间时提前读取下一个地址的数据使得访问更快.而多线程操作会导致缓存miss.从而导致访问性能下降.