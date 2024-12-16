## 一、数据类型



## 二、集合

### 2.1 线程不安全集合

#### 是否支持 Null 值

**答案解析：**

通常而言，支持自动排序或者线程安全的集合不允许 Null 值。

不支持的情况：

- `TreeSet` 和 `TreeMap` 使用 `Comparator` 或对象的 `compareTo` 方法进行自动排序，因此它们不允许 `null` 值。因为在比较过程中，`null` 无法与其他元素进行有意义的比较，会导致 `NullPointerException`。
- `ConcurrentHashMap` 是线程安全的 `Map` 实现，它不允许 `null` 键或 `null` 值。由于该类实现的高并发性，`null` 键或值会影响其工作机制，可能引发不期望的行为或异常。如果在 `ConcurrentHashMap` 中插入 `null` 键或值，会抛出 `NullPointerException`。
- `PriorityQueue` 是基于优先级排序的队列实现，不允许 `null` 元素。因为它必须对元素进行排序或优先级处理，`null` 值无法参与其中。

支持的情况：

- `HashSet` 和 `HashMap` 允许存储 `null` 值。`HashMap` 允许一个 `null` 键和多个 `null` 值。`HashSet` 只允许一个 `null` 值。
- `ArrayList` 和 `LinkedList` 都允许 `null` 元素，因为它们基于索引或链表存储元素，不对值的比较或排序做限制。注意，`ArrayList` 可以进行排序，但 `ArrayList` 自身不会自动排序。



### 2.2 线程安全集合

#### 什么是 ConcurrentHashMap ？

**答案解析：**

**(1)存储结构**

在 JDK 17， 数据结构为 <font color="red">**数组 + 链表 / 红黑树**</font>。其中，

- 数组(哈希表)： 存放 HashMap 里面的键值对。数组里面的每个元素要么是 `null`，要么是哈希桶的头节点（一个 `Node<K, V>` 对象），指向链表 / 红黑树的起始。
- 链表 / 红黑树(哈希桶)：解决哈希冲突。当链表达到一定长度时，链表将会转化为红黑树。 在 链表 / 红黑树 中，所有元素的键不相同，但键的哈希值相同。

**(2) 线程安全**

线程安全的实现策略为 `Node + CAS + synchronized`，具体优化如下：

- **细粒度锁与 CAS 的结合使用**：使用更细粒度的锁（`synchronized`）和 CAS (Compare-And-Swap) 算法来保证线程安全，根据锁竞争操作来判断使用 `synchronized`  或者 CAS 。
- **锁粒度进一步优化到节点级别**：锁的粒度被细化到每个节点，最大限度地降低锁的范围，从而实现更高的并发度。JDK 17 最大并发度是 Node 数组的大小。

- **链表或树节点操作中的自旋机制**：在某些操作（如链表或树节点的插入、删除）中，CAS 失败后，采用自旋的方式重试。在高并发环境下，自旋可以避免不必要的上下文切换，从而提高性能。如果自旋锁仍然无法解决冲突，`ConcurrentHashMap` 会回退到更传统的锁机制，即使用内置的 `synchronized` 来进行更粗粒度的锁保护。



#### ConcurrentHashMap 操作的安全实现有什么区别？

**(1) initTable() 方法**

- **CAS**：使用 CAS 尝试将 `sizeCtl` 设置为 -1，表示当前线程开始初始化
- **`volatile` 可见性**：`table` 是一个 `volatile` 变量，保证多线程对其操作的可见性。
- **细粒度控制**：通过 `sizeCtl` 的状态（正数/负数）协调线程，避免多个线程同时初始化或扩容。

**(2) put() 方法**

- 当桶为空时，使用 CAS 更新。
- 当目标桶不为空时（存在链表或红黑树结构），操作逻辑复杂，无法通过 CAS 一步完成，需要 `synchronized` 来保护一致性。

**(3) get() 方法**

- 只读操作无需加锁
- 基于 `volatile` 保证可见性：`ConcurrentHashMap` 内部使用了 `volatile` 修饰 `table` 和桶中的节点（如 `Node` 数组）。



#### ConcurrentHashMap 为什么 key 和 value 不能为 null？

null 具备二义性：

- 不存在
- 存在，但本身为 null

在多线程的环境下，`containsKey(key)` **无法完全准确地判断一个 `key` 是否存在**。因为 `containsKey` 本身并不会加锁，也没有保证在它执行时，其他线程不会修改 `Map`。举个例子，

- 线程 A 执行 `containsKey(key)`，判断 `key` 是否存在。

- 在查询过程中，线程 B 执行了 `remove(key)`，将该 `key` 从 `Map` 中移除。

- 如果线程 A 得到 `containsKey(key)` 为 `true`，而实际上 `key` 已经被线程 B 删除了，结果就不准确了。







#### ConcurrentHashMap 和 HashTable 的区别？

- **`HashTable` 使用 `synchronized` 来保证线程安全**，包含 `get()/put()` 在内的所有相关需要进行同步执行的方法都加上了 `synchronized`  关键字，以锁定这个哈希表。`HashTable` 线程安全策略的代价非常大，这相当于给整个哈希表加了一把大锁。

- ConcurrentHashMap 则使用 `Node + CAS + synchronized` 保证线程安全，锁的粒度降到来 Node。







## 三、并发

### 3.1 线程

#### 创建线程的方式

Java 提供了三种创建线程的方法：

- 通过实现 Runnable 接口；
- 通过继承 Thread 类本身；
- 通过 Callable 和 Future 创建线程。

<br>



 **Thread 类 和  Runnable 接口 的区别**

 Thread 类  实现 Runnable 接口

<br>



####  wait()方法和 sleep()方法的区别是什么?

**答案解析：**

- 功能： `wait()` 是 `Object` 类中的一个方法, 使线程进入 **`WAITING`** 状态，等待被其他线程唤醒。。而 `sleep()`是 `Thread` 类中的一个静态方法, 使线程进入 **`TIMED_WAITING`** 状态，直到指定的休眠时间结束。
- 底层实现： `wait()` 会释放锁，让其他线程有机会获得锁。`sleep()` 不会释放锁，当前线程仍然持有它所占有的锁。

<br>



#### **启动一个线程是run()还是start()?有什么区别？**

启动一个线程是调用start(）方法，这并不意味着线程就会立即运行，只是进入了可运行状态。

start() 方法用于启动线程，run() 方法用于执行线程的运行时代码。run() 可以重复调用，而 start() 只能调用一次。

直接调用run()方法不会产生线程，而是把它当作普通的方法调用，马上执行。

<br>





#### 线程同步通信

使用同步的方式，通过notifyAll和wait方法实现线程间的通信。

<br>





## 四、Java 虚拟机

### 4.1 Java 虚拟机

#### 简单阐述一下 JVM 的体系结构？





1. 说下 JVM 内存结构
2. 什么是 JMM Java 内存模型？
3. 如何判断对象已经死亡？
4. 说说常见的垃圾回收器
5. 有没有看过 GC 垃圾回收日志？
6. 什么情况下会触发 JVM 内存溢出？如何解决？





### 4.2 内存泄漏

#### JVM 常见的内存泄漏原因？

**答案解析：**

[Java 中发生内存泄漏 5 个场景以及解决方法_java内存泄漏-CSDN博客](https://blog.csdn.net/wuhuayangs/article/details/122594327)

- **连接没有正常关闭**（如数据库连接、网络连接等）。如果在使用数据库连接、文件流、网络连接等资源时，没有在使用完毕后及时关闭，JVM 会将这些连接对象保留在内存中，造成资源无法释放，导致内存泄漏。为了避免这类问题，我们可以使用 `try-with-resources` 或在 `finally` 代码块中显式关闭连接。
- **静态集合类（如 `Map`, `List` 等）持有大量数据**。这些静态变量的生命周期和应用程序一致，则容器中的对象在程序结束之前将不能被释放，从而造成内存泄漏，简单而言，长生命周期的对象持有短生命周期对象的引用，尽管短生命周期的对象不再使用，但是因为长生命周期对象持有它的引用而导致不能被回收。
-  **线程未停止或未关闭**。
- **使用 ThreadLocal 造成内存泄露**。使用到 ThreadLocal 来保留线程池中线程的变量副本时，ThreadLocal 没有显式地删除时，就会一直保留在内存中，不会被垃圾回收。解决办法是不在使用 ThreadLocal 时，调用 remove() 方法。

<br>

#### 内存泄漏的常见错误?

**答案解析：**

`java.lang.OutOfMemoryError` 错误的种类有：

+ 堆内存溢出  `java.lang.OutOfMemoryError: Java heap space`
+ 元空间溢出  `java.lang.OutOfMemoryError: Metaspace`
+ 直接内存溢出  `java.lang.OutOfMemoryError: Direct buffer memory`
+ 线程内存溢出（ `java.lang.OutOfMemoryError: unable to create new native thread`

比较特殊地，栈溢出 `java.lang.StackOverflowError` 耗尽了栈空间，但并没有导致内存资源耗尽。

