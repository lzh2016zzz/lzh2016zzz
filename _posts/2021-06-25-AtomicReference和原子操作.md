---
tags: 
  java
article_header:
  type: overlay
  theme: dark
  background_color: '#203028'
  background_image:
    gradient: 'linear-gradient(135deg, rgba(34, 139, 87 , .4), rgba(139, 34, 139, .4))'
    src: /assets/images/cover6.jpg
---

`AtomicReference`类提供了一个可以将普通对象包装为原子对象的工具,它可以保证你修改对象的操作具有原子性.

原子性意味着在多个线程同时更新一个`AtomicReference<T>`引用的对象时可以保证线程安全.

<!--more-->

一般来说,在多个线程访问共享资源时,需要通过加锁控制并发:

```java
var shared_object = new SharedObject();
Lock lock = new ReentrantLock();
try{
  lock.lock();
  //操作共享资源shared_object
}finally{
  lock.unlock();
}
```

上面的访问方式其实是对共享资源加悲观锁的访问方式.而`AtomicReference`提供了`无锁访问共享资源`的能力:

```java
public class AtomicRefTest {
    public static void main(String[] args) throws InterruptedException {
        AtomicReference<Integer> ref = new AtomicReference<>(new Integer(1000));

        List<Thread> list = new ArrayList<>();
        for (int i = 0; i < 1000; i++) {
            Thread t = new Thread(new Task(ref), "Thread-" + i);
            list.add(t);
            t.start();
        }

        for (Thread t : list) {
            t.join();
        }

        System.out.println(ref.get());    // 打印2000
    }

}

class Task implements Runnable {
    private AtomicReference<Integer> ref;

    Task(AtomicReference<Integer> ref) {
        this.ref = ref;
    }
    
    @Override
    public void run() {
        for (; ; ) {    //自旋操作
            Integer oldV = ref.get();   
            if (ref.compareAndSet(oldV, oldV + 1))  // CAS操作 
                break;
        }
    }
}
```

上述示例，最终打印“2000”。

该示例并没有使用锁，而是使用**自旋+CAS**的无锁操作保证共享变量的线程安全。1000个线程，每个线程对金额增加1，最终结果为2000，如果线程不安全，最终结果应该会小于2000.

## compareAndSet方法

`AtomicReference#compareAndSet()`方法会将入参变量指向的对象和`AtomicReference`引用的对象做比较,如果两者指向的对象是同一个地址,则执行更新操作.有趣的是,因为是通过 `==`进行比较,所以不能用这个方法保证对同一个对象的多次修改操作的原子性.

## set方法

`=`赋值操作本身就具备原子性,所以这个方法意义不明.本人能想到的好处是不加`volatile`情况下也能保证可见性.