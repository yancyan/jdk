

## Java 并发基础之内存模型

### 重排序

重排序由一下几种机制引起：
- 编译器的优化： 对于没有数据依赖关系的操作，编译器在编译过程中会进行一定程度的重排
- 指令重排序：CPU 优化行为，也是会对不存在数据依赖关系的指令进行一定程度的重排。
- 内存系统重排序：内存系统没有重排序，但是由于有缓存的存在，使得程序整体上会表现出乱序的行为。


### 内存可见性

现代多核 CPU 中每个核心拥有自己的一级缓存或一级缓存加上二级缓存等，问题就发生在每个核心的独占缓存上。每个核心都会将自己需要的数据读到独占缓存中，数据修改后也是写入到缓存中，然后等待刷入到主存中。所以会导致有些核心读取的值是一个过期的值。

Java 作为高级语言，屏蔽了这些底层细节，用 JMM 定义了一套读写内存数据的规范，虽然我们不再需要关心一级缓存和二级缓存的问题，但是，JMM 抽象了主内存和本地内存的概念。

所有的共享变量存在于主内存中，每个线程有自己的本地内存，线程读写共享数据也是通过本地内存交换的，所以可见性问题依然是存在的。这里说的本地内存并不是真的是一块给每个线程分配的内存，而是 JMM 的一个抽象，是对于寄存器、一级缓存、二级缓存等的抽象。


### 原子性

Java 编程语言规范中提到，对于 64 位的值的写入，可以分为两个 32 位的操作进行写入。本来一个整体的赋值操作，被拆分为低 32 位赋值和高 32 位赋值两个操作，中间如果发生了其他线程对于这个值的读操作，必然就会读到一个奇怪的值。

这个时候我们要使用 volatile 关键字进行控制了，JMM 规定了对于 volatile long 和 volatile double，JVM 需要保证写入操作的原子性（在 64 位的 JVM 中，不加 volatile 也是可以的）。

另外，对于引用的读写操作始终是原子的，不管是 32 位的机器还是 64 位的机器。
Java 编程语言规范同样提到，鼓励 JVM 的开发者能保证 64 位值操作的原子性，也鼓励使用者尽量使用 volatile 或使用正确的同步方式。关键词是”鼓励“


## Java 对于并发的规范约束

### Synchronization Order
```text
https://docs.oracle.com/javase/specs/jls/se8/html/jls-17.html#jls-17.4.4
```
1. 对于监视器 m 的解锁与所有后续操作对于 m 的加锁同步
2. 对 volatile 变量 v 的写入，与所有其他线程后续对 v 的读同步
3. 启动线程的操作与线程中的第一个操作同步
4. 对于每个属性写入默认值（0， false，null）与每个线程对其进行的操作同步(从概念上来说，每个对象都是在程序启动时用默认值初始化来创建的)
5. 线程 T1 的最后操作与线程 T2 发现线程 T1 已经结束同步(线程 T2 可以通过 T1.isAlive() 或 T1.join() 方法来判断 T1 是否已经终结)
6. 如果线程 T1 中断了 T2，那么线程 T1 的中断操作与其他所有线程发现 T2 被中断了同步（通过抛出 InterruptedException 异常，或者调用 Thread.interrupted 或 Thread.isInterrupted ）

### Happens-before Order
两个操作可以用 happens-before 来确定它们的执行顺序，如果一个操作 happens-before 于另一个操作，那么我们说第一个操作对于第二个操作是可见的。
如果我们分别有操作 x 和操作 y，我们写成 hb(x, y) 来表示 x happens-before y。
以下几个规则也是来自于 Java 8 语言规范 Happens-before Order：
- 如果操作 x 和操作 y 是同一个线程的两个操作，并且在代码上操作 x 先于操作 y 出现，那么有 hb(x, y)
- 对象构造方法的最后一行指令 happens-before 于 finalize() 方法的第一行指令
- 如果操作 x 与随后的操作 y 构成同步，那么 hb(x, y)
- hb(x, y) 和 hb(y, z)，那么可以推断出 hb(x, z)

这里再提一点，x happens-before y，并不是说 x 操作一定要在 y 操作之前被执行，而是说 x 的执行结果对于 y 是可见的，只要满足可见性，发生了重排序也是可以的。


### synchronized
一个线程在获取到监视器锁以后才能进入 synchronized 控制的代码块，

进入代码块，该线程对于共享变量的缓存就会失效，因此 synchronized 代码块中对于共享变量的读取需要从主内存中重新获取，也就能获取到最新的值。 退出代码块的时候的，会将该线程写缓冲区中的数据刷到主内存中，

所以在 synchronized 代码块之前或 synchronized 代码块中对于共享变量的操作随着该线程退出 synchronized 块，会立即对其他线程可见（这句话的前提是其他读取共享变量的线程会从主内存读取最新值）。

因此，我们可以总结一下：线程 a 对于进入 synchronized 块之前或在 synchronized 中对于共享变量的操作，对于后续的持有同一个监视器锁的线程 b 可见

注意一点，在进入 synchronized 的时候，并不会保证之前的写操作刷入到主内存中，synchronized 主要是保证退出的时候能将本地内存的数据刷入到主内存。



## 单例模式中的双重检查
```text
http://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html
```

### volatile 关键字


**volatile 的内存可见性**

读一个 volatile 变量之前，需要先使相应的本地缓存失效，这样就必须到主内存读取最新值，写一个 volatile 属性会立即刷入到主内存。
所以，volatile 读和 monitorenter 有相同的语义，volatile 写和 monitorexit 有相同的语义。

**volatile 的禁止重排序**

volatile 的禁止重排序并不局限于两个 volatile 的属性操作不能重排序，而且是 volatile 属性操作和它周围的普通属性的操作也不能重排序。

根据 volatile 的内存可见性和禁止重排序，那么我们不难得出一个推论：线程 a 如果写入一个 volatile 变量，此时线程 b 再读取这个变量，那么此时对于线程 a 可见的所有属性对于线程 b 都是可见的。

**volatile 小结**
- volatile 修饰符适用于以下场景：某个属性被多个线程共享，其中有一个线程修改了此属性，其他线程可以立即得到修改后的值
- volatile 属性的读写操作都是无锁的，它不能替代 synchronized，因为它没有提供原子性和互斥性。因为无锁，不需要花费时间在获取锁和释放锁上，所以说它是低成本的。
- volatile 只能作用于属性，我们用 volatile 修饰属性，这样 compilers 就不会对这个属性做指令重排序。
- volatile 提供了可见性，任何一个线程对其的修改将立马对其他线程可见
- volatile 提供了 happens-before 保证，对 volatile 变量 v 的写入 happens-before 所有其他线程后续对 v 的读操作。
- volatile 可以使得 long 和 double 的赋值是原子的，前面在说原子性的时候提到过。

### final 关键字

重排序规则
- 在构造函数内对一个final域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。（构造函数return之前，插入一个StoreStore屏障）
- 初次读一个包含final域的对象的引用，与随后初次读这个final域，这两个操作之间不能重排序。（编译器会在读final域操作的前面插入一个LoadLoad屏障）
 
即上面提到，写final域的重排序规则会要求译编器在final域的写之后，构造函数return之前，插入一个StoreStore障屏。读final域的重排序规则要求编译器在读final域的操作前面插入一个LoadLoad屏障。




















