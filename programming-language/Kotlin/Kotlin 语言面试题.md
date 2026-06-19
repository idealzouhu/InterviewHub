## Kotlin 基础

#### Kotlin 有哪些特点？它和Java有什么区别？

**特点**：简洁、空安全、函数式编程、协程支持、100%兼容Java。

**区别**：

| 方面     | Kotlin         | Java                  |
| -------- | -------------- | --------------------- |
| 空安全   | 内置空安全机制 | 需手动判空            |
| 扩展函数 | 支持           | 不支持                |
| 协程     | 原生支持       | 需第三方库            |
| 数据类   | `data class`   | 需手动写getter/setter |
| 类型推断 | 广泛支持       | 有限支持              |



#### val 和 var 有什么区别？各自的使用场景是什么？

- **val**：只读变量（不可重新赋值），类似 Java 的 `final`。用于常量、配置值等不变数据。
- **var**：可变变量。用于需要改变的值。

```kotlin
val name = "Tom"  // 不可修改
var count = 0     // 可修改
count++           // 允许
```



#### 基本数据类型有哪些？和 java 有什么不同？

**Kotlin基本类型**：Byte、Short、Int、Long、Float、Double、Char、Boolean。

**区别**：

- Kotlin所有类型都是对象，没有Java的原始类型
- Kotlin不区分包装类和基本类型
- 数字类型自动装箱/拆箱
- Kotlin没有隐式拓宽转换（如Int不能直接赋给Long）



#### 怎么使用字符串模板？

用`$`符号引用变量，`${}`引用表达式：

```kotlin
val name = "Kotlin"
println("Hello $name")        // Hello Kotlin
println("长度: ${name.length}") // 长度: 6
```



#### when 表达式相对于 Java 的 switch 有什么优势？

1. **更强大的匹配**：支持任意类型、范围、条件表达式
2. **不需要break**：不会穿透
3. **可作为表达式**：有返回值
4. **智能类型推断**：匹配后自动转型

```kotlin
when (x) {
    in 1..10 -> println("1到10")
    is String -> println("字符串")
    else -> println("其他")
}
```



#### 如何定义函数？有哪些特殊的函数类型？

**定义**：

```kotlin
fun add(a: Int, b: Int): Int = a + b
```

**特殊函数**：

- 单表达式函数：`fun square(x: Int) = x * x`
- 内联函数：`inline fun`
- 尾递归函数：`tailrec fun`
- 高阶函数：参数或返回值为函数
- 扩展函数：为已有类添加方法



#### 可见性修饰符有哪些？和 Java 有什么区别？

| **修饰符** | **Kotlin**           | **Java**         |
| :--------- | :------------------- | :--------------- |
| public     | 默认（任何地方可见） | 默认（包内可见） |
| private    | 类内部/文件内        | 类内部           |
| protected  | 类及其子类           | 类及其子类、同包 |
| internal   | 模块内               | 无对应           |

**关键区别**：Kotlin默认public，Java默认包级私有；Kotlin新增internal。



#### 扩展函数是什么？如何实现的？

**定义**：在不修改原类的情况下为其添加新函数。

```kotlin
fun String.addExclamation(): String = "$this!"
println("Hello".addExclamation()) // Hello!
```

**实现原理**：编译成静态工具函数，第一个参数是被扩展类的实例。本质是语法糖，不是真正修改了类。



## 协程

#### Kotlin 的协程是什么？在 Android 中如何使用？

**协程**：轻量级并发框架，可以在单线程上实现异步非阻塞编程，通过挂起而非阻塞来管理并发。

**Android中使用**：

```kotlin
// 添加依赖
implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3'

// 在Activity/Fragment中使用
lifecycleScope.launch {
    val result = withContext(Dispatchers.IO) { fetchData() }
    updateUI(result)
}
```



#### 在 Android 中 Kotlin 的创建过程是怎样的？

1. 定义协程作用域（CoroutineScope）
2. 使用`launch`或`async`启动协程
3. 指定调度器（Dispatchers.Main/IO/Default）
4. 在协程体内执行异步操作

```kotlin
viewModelScope.launch(Dispatchers.IO) {
    val data = repository.getData()
    withContext(Dispatchers.Main) {
        _uiState.value = data
    }
}
```



#### 在 Android 中 Kotlin 协程的挂起与恢复机制是怎样的？

**挂起**：遇到`suspend`函数时，协程保存当前状态并释放线程，不阻塞线程。

**恢复**：异步操作完成后，协程从挂起点继续执行。

**原理**：编译时将协程体转换为状态机，每个挂起点是一个状态，通过Continuation传递控制权。



#### 如何在 Kotlin 协程中配合使用 Okhttp？

```kotlin
suspend fun fetchData(): String = withContext(Dispatchers.IO) {
    val client = OkHttpClient()
    val request = Request.Builder().url("https://api.example.com").build()
    val response = client.newCall(request).execute()
    response.body?.string() ?: ""
}
```

或使用`suspendCancellableCoroutine`封装回调：

```kotlin
suspend fun fetchAsync() = suspendCancellableCoroutine<String> { cont ->
    client.newCall(request).enqueue(object : Callback {
        override fun onResponse(call: Call, response: Response) {
            cont.resume(response.body?.string() ?: "")
        }
        override fun onFailure(call: Call, e: IOException) {
            cont.resumeWithException(e)
        }
    })
}
```



#### 如何使用 Retrofit 配合使用 Kotlin 协程？

Retrofit 2.6+ 原生支持协程：

```kotlin
// API接口定义
interface ApiService {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: Int): User
}

// 调用
viewModelScope.launch {
    try {
        val user = apiService.getUser(1)
        // 更新UI
    } catch (e: Exception) {
        // 错误处理
    }
}
```



#### Kotlin 的协程与 Java 的线程有什么区别与联系？

| **对比项** | **协程**           | **线程**     |
| :--------- | :----------------- | :----------- |
| 调度单位   | 用户态             | 内核态       |
| 开销       | 极低（可百万级）   | 较高（千级） |
| 切换成本   | 函数调用级别       | 系统调用级别 |
| 阻塞       | 挂起不阻塞线程     | 阻塞线程     |
| 共享数据   | 无需锁（同一线程） | 需同步机制   |

**联系**：协程运行在线程之上，可以理解为轻量级线程。



#### launch 和 async 有什么区别?各自的应用场景是什么？

| **对比** | **launch**    | **async**          |
| :------- | :------------ | :----------------- |
| 返回值   | Job（无结果） | Deferred（有结果） |
| 获取结果 | 无            | await()            |
| 异常处理 | 直接抛出      | 在await时抛出      |

**场景**：

- `launch`：执行无需返回结果的任务（如日志、UI更新）

- `async`：需要并行计算并获取结果（如同时请求多个API）



#### Kotlin 协程的作用域 CoroutineScope 是什么？如何选择合适的作用域？

**CoroutineScope**：管理协程生命周期的上下文，控制协程的启动、取消。

**常见作用域**：

- `GlobalScope`：全局作用域，不推荐（生命周期与应用一致）
- `lifecycleScope`：Activity/Fragment生命周期绑定
- `viewModelScope`：ViewModel生命周期绑定
- `coroutineScope`：自定义作用域，等待所有子协程完成

**选择原则**：谁创建谁管理，协程生命周期不超过其宿主。



#### Kotlin 的 Flow 是什么？和 RxJava 有什么区别？

**Flow**：冷数据流，用于处理异步数据序列，支持背压。

**与RxJava区别**：

| 对比     | Flow       | RxJava     |
| -------- | ---------- | ---------- |
| 冷热     | 默认冷流   | 支持冷热   |
| 背压     | 内置支持   | 需显式策略 |
| 操作符   | 较少但够用 | 丰富       |
| 学习曲线 | 较平缓     | 较陡峭     |
| 协程集成 | 原生支持   | 需适配     |



## 空安全

#### 安全调用操作符 ?. 和非空断言 !! 有什么区别

- `?.`：安全调用，如果为null则返回null，不抛异常
- `!!`：非空断言，如果为null则抛NPE

```kotlin
val len = str?.length      // null安全
val len = str!!.length     // 可能抛NPE
```



#### Elvis 操作符 ?: 有什么作用？如何使用？

**作用**：提供默认值，左侧为空时使用右侧值。

```kotlin
val name: String? = null
val displayName = name ?: "Unknown"  // displayName = "Unknown"

// 结合return/throw
fun getLength(str: String?): Int {
    return str?.length ?: throw IllegalArgumentException("str不能为空")
}
```



#### 如何在 Kotlin 中处理 Java 代码的空安全问题？

1. **平台类型**：Java类型在Kotlin中为平台类型（Type!），可空性未知
2. **注解**：使用`@Nullable`/`@NonNull`注解，Kotlin会识别
3. **手动处理**：使用`?.`、`?:`、`!!`自行决定处理方式
4. **Optional**：将Java Optional转为Kotlin的可空类型



## 面向对象

#### 如何定义类？主构造函数和次构造函数有什么区别？

**定义类**：

```kotlin
class Person(val name: String, var age: Int)  // 主构造函数
```

**区别**：

- **主构造函数**：在类头声明，最简洁，可包含属性声明
- **次构造函数**：用`constructor`关键字，必须直接或间接委托给主构造函数

```kotlin
class Person(val name: String) {
    constructor(name: String, age: Int) : this(name) {
        // 次构造函数
    }
}
```





#### 数据类是什么？有什么作用？

**定义**：用`data`关键字修饰的类，自动生成`equals()`、`hashCode()`、`toString()`、`copy()`、`componentN()`。

**作用**：专门用于持有数据的POJO类，减少样板代码。

```kotlin
data class User(val id: Int, val name: String)
```



#### 密封类是什么？有什么作用？

**密封类**：用`sealed`修饰的抽象类，限制子类只能在同一文件中定义。

**作用**：

- 表示受限的类层次结构
- 在`when`表达式中穷举所有分支，无需else
- 常用于状态模式、结果封装

```kotlin
sealed class Result {
    data class Success(val data: String) : Result()
    data class Error(val msg: String) : Result()
}
```



#### object 关键字有哪些用法？

1. **对象声明**：单例模式

   ```kotlin
   object Singleton {
       fun doSomething() {}
   }
   ```

2. **伴生对象**：类级别的静态成员

3. **对象表达式**：匿名内部类

   ```kotlin
   val listener = object : ClickListener {
       override fun onClick() {}
   }
   ```



#### 伴生对象是什么？和 Java 的静态成员有什么区别？

# Kotlin 面试题整理与答案

------

## Kotlin 基础

### Kotlin 有哪些特点？它和Java有什么区别？

**特点**：简洁、空安全、函数式编程、协程支持、100%兼容Java。

**区别**：

| 方面     | Kotlin         | Java                  |
| -------- | -------------- | --------------------- |
| 空安全   | 内置空安全机制 | 需手动判空            |
| 扩展函数 | 支持           | 不支持                |
| 协程     | 原生支持       | 需第三方库            |
| 数据类   | `data class`   | 需手动写getter/setter |
| 类型推断 | 广泛支持       | 有限支持              |

------

### val 和 var 有什么区别？各自的使用场景是什么？

- 

  **val**：只读变量（不可重新赋值），类似Java的`final`。用于常量、配置值等不变数据。

- 

  **var**：可变变量。用于需要改变的值。

```kotlin
val name = "Tom"  // 不可修改
var count = 0     // 可修改
count++           // 允许
```

------

### 基本数据类型有哪些？和java有什么不同？

**Kotlin基本类型**：Byte、Short、Int、Long、Float、Double、Char、Boolean。

**区别**：

- 

  Kotlin所有类型都是对象，没有Java的原始类型

- 

  Kotlin不区分包装类和基本类型

- 

  数字类型自动装箱/拆箱

- 

  Kotlin没有隐式拓宽转换（如Int不能直接赋给Long）

------

### 怎么使用字符串模板？

用`$`符号引用变量，`${}`引用表达式：

```kotlin
val name = "Kotlin"
println("Hello $name")        // Hello Kotlin
println("长度: ${name.length}") // 长度: 6
```

------

### when 表达式相对于 Java 的 switch 有什么优势？

1. 

   **更强大的匹配**：支持任意类型、范围、条件表达式

2. 

   **不需要break**：不会穿透

3. 

   **可作为表达式**：有返回值

4. 

   **智能类型推断**：匹配后自动转型

```kotlin
when (x) {
    in 1..10 -> println("1到10")
    is String -> println("字符串")
    else -> println("其他")
}
```

------

### 如何定义函数？有哪些特殊的函数类型？

**定义**：

```kotlin
fun add(a: Int, b: Int): Int = a + b
```

**特殊函数**：

- 

  单表达式函数：`fun square(x: Int) = x * x`

- 

  内联函数：`inline fun`

- 

  尾递归函数：`tailrec fun`

- 

  高阶函数：参数或返回值为函数

- 

  扩展函数：为已有类添加方法

------

### 可见性修饰符有哪些？和Java有什么区别？

| 修饰符    | Kotlin               | Java             |
| --------- | -------------------- | ---------------- |
| public    | 默认（任何地方可见） | 默认（包内可见） |
| private   | 类内部/文件内        | 类内部           |
| protected | 类及其子类           | 类及其子类、同包 |
| internal  | 模块内               | 无对应           |

**关键区别**：Kotlin默认public，Java默认包级私有；Kotlin新增internal。

------

### 扩展函数是什么？如何实现的？

**定义**：在不修改原类的情况下为其添加新函数。

```kotlin
fun String.addExclamation(): String = "$this!"
println("Hello".addExclamation()) // Hello!
```

**实现原理**：编译成静态工具函数，第一个参数是被扩展类的实例。本质是语法糖，不是真正修改了类。

------

## 协程

### Kotlin 的协程是什么？在Android中如何使用？

**协程**：轻量级并发框架，可以在单线程上实现异步非阻塞编程，通过挂起而非阻塞来管理并发。

**Android中使用**：

```kotlin
// 添加依赖
implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3'

// 在Activity/Fragment中使用
lifecycleScope.launch {
    val result = withContext(Dispatchers.IO) { fetchData() }
    updateUI(result)
}
```

------

### 在Android中Kotlin协程的创建过程是怎样的？

1. 

   定义协程作用域（CoroutineScope）

2. 

   使用`launch`或`async`启动协程

3. 

   指定调度器（Dispatchers.Main/IO/Default）

4. 

   在协程体内执行异步操作

```kotlin
viewModelScope.launch(Dispatchers.IO) {
    val data = repository.getData()
    withContext(Dispatchers.Main) {
        _uiState.value = data
    }
}
```

------

### 在Android中Kotlin协程的挂起与恢复机制是怎样的？

**挂起**：遇到`suspend`函数时，协程保存当前状态并释放线程，不阻塞线程。

**恢复**：异步操作完成后，协程从挂起点继续执行。

**原理**：编译时将协程体转换为状态机，每个挂起点是一个状态，通过Continuation传递控制权。

------

### 如何在Kotlin协程中配合使用Okhttp？

```kotlin
suspend fun fetchData(): String = withContext(Dispatchers.IO) {
    val client = OkHttpClient()
    val request = Request.Builder().url("https://api.example.com").build()
    val response = client.newCall(request).execute()
    response.body?.string() ?: ""
}
```

或使用`suspendCancellableCoroutine`封装回调：

```kotlin
suspend fun fetchAsync() = suspendCancellableCoroutine<String> { cont ->
    client.newCall(request).enqueue(object : Callback {
        override fun onResponse(call: Call, response: Response) {
            cont.resume(response.body?.string() ?: "")
        }
        override fun onFailure(call: Call, e: IOException) {
            cont.resumeWithException(e)
        }
    })
}
```

------

### 如何使用Retrofit配合使用Kotlin协程？

Retrofit 2.6+ 原生支持协程：

```kotlin
// API接口定义
interface ApiService {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: Int): User
}

// 调用
viewModelScope.launch {
    try {
        val user = apiService.getUser(1)
        // 更新UI
    } catch (e: Exception) {
        // 错误处理
    }
}
```

------

### Kotlin的协程与Java的线程有什么区别与联系？

| 对比项   | 协程               | 线程         |
| -------- | ------------------ | ------------ |
| 调度单位 | 用户态             | 内核态       |
| 开销     | 极低（可百万级）   | 较高（千级） |
| 切换成本 | 函数调用级别       | 系统调用级别 |
| 阻塞     | 挂起不阻塞线程     | 阻塞线程     |
| 共享数据 | 无需锁（同一线程） | 需同步机制   |

**联系**：协程运行在线程之上，可以理解为轻量级线程。

------

### launch 和 async 有什么区别?各自的应用场景是什么？

| 对比     | launch        | async              |
| -------- | ------------- | ------------------ |
| 返回值   | Job（无结果） | Deferred（有结果） |
| 获取结果 | 无            | await()            |
| 异常处理 | 直接抛出      | 在await时抛出      |

**场景**：

- 

  `launch`：执行无需返回结果的任务（如日志、UI更新）

- 

  `async`：需要并行计算并获取结果（如同时请求多个API）

------

### Kotlin协程的作用域CoroutineScope是什么？如何选择合适的作用域？

**CoroutineScope**：管理协程生命周期的上下文，控制协程的启动、取消。

**常见作用域**：

- 

  `GlobalScope`：全局作用域，不推荐（生命周期与应用一致）

- 

  `lifecycleScope`：Activity/Fragment生命周期绑定

- 

  `viewModelScope`：ViewModel生命周期绑定

- 

  `coroutineScope`：自定义作用域，等待所有子协程完成

**选择原则**：谁创建谁管理，协程生命周期不超过其宿主。

------

### Kotlin的Flow是什么？和RxJava有什么区别？

**Flow**：冷数据流，用于处理异步数据序列，支持背压。

**与RxJava区别**：

| 对比     | Flow       | RxJava     |
| -------- | ---------- | ---------- |
| 冷热     | 默认冷流   | 支持冷热   |
| 背压     | 内置支持   | 需显式策略 |
| 操作符   | 较少但够用 | 丰富       |
| 学习曲线 | 较平缓     | 较陡峭     |
| 协程集成 | 原生支持   | 需适配     |

------

## 空安全

### 安全调用操作符 ?. 和非空断言 !! 有什么区别

- 

  `?.`：安全调用，如果为null则返回null，不抛异常

- 

  `!!`：非空断言，如果为null则抛NPE

```kotlin
val len = str?.length      // null安全
val len = str!!.length     // 可能抛NPE
```

------

### Elvis 操作符 ?: 有什么作用？如何使用？

**作用**：提供默认值，左侧为空时使用右侧值。

```kotlin
val name: String? = null
val displayName = name ?: "Unknown"  // displayName = "Unknown"

// 结合return/throw
fun getLength(str: String?): Int {
    return str?.length ?: throw IllegalArgumentException("str不能为空")
}
```

------

### 如何在Kotlin中处理Java代码的空安全问题？

1. 

   **平台类型**：Java类型在Kotlin中为平台类型（Type!），可空性未知

2. 

   **注解**：使用`@Nullable`/`@NonNull`注解，Kotlin会识别

3. 

   **手动处理**：使用`?.`、`?:`、`!!`自行决定处理方式

4. 

   **Optional**：将Java Optional转为Kotlin的可空类型

------

## 面向对象

### 如何定义类？主构造函数和次构造函数有什么区别？

**定义类**：

```kotlin
class Person(val name: String, var age: Int)  // 主构造函数
```

**区别**：

- 

  **主构造函数**：在类头声明，最简洁，可包含属性声明

- 

  **次构造函数**：用`constructor`关键字，必须直接或间接委托给主构造函数

```kotlin
class Person(val name: String) {
    constructor(name: String, age: Int) : this(name) {
        // 次构造函数
    }
}
```

------

### 数据类是什么？有什么作用？

**定义**：用`data`关键字修饰的类，自动生成`equals()`、`hashCode()`、`toString()`、`copy()`、`componentN()`。

**作用**：专门用于持有数据的POJO类，减少样板代码。

```kotlin
data class User(val id: Int, val name: String)
```

------

### 密封类是什么？有什么作用？

**密封类**：用`sealed`修饰的抽象类，限制子类只能在同一文件中定义。

**作用**：

- 

  表示受限的类层次结构

- 

  在`when`表达式中穷举所有分支，无需else

- 

  常用于状态模式、结果封装

```kotlin
sealed class Result {
    data class Success(val data: String) : Result()
    data class Error(val msg: String) : Result()
}
```

------

### object 关键字有哪些用法？

1. 

   **对象声明**：单例模式

   ```kotlin
   object Singleton {
       fun doSomething() {}
   }
   ```

2. 

   **伴生对象**：类级别的静态成员

3. 

   **对象表达式**：匿名内部类

   ```kotlin
   val listener = object : ClickListener {
       override fun onClick() {}
   }
   ```



### 伴生对象是什么？和Java的静态成员有什么区别？

**伴生对象**：用`companion object`定义的类内单例对象，替代Java的static成员。

**区别**：

- 伴生对象是真正的对象，可以实现接口、继承类
- 伴生对象可以有名字（默认为Companion）
- 伴生对象在运行时是真实存在的实例

```kotlin
class MyClass {
    companion object Factory {
        fun create() = MyClass()
    }
}
MyClass.create()  // 调用
```



#### 为什么 kotlin 的类默认是 final 的？

**设计理念**：组合优于继承。默认final鼓励使用扩展函数、委托等方式代替继承，避免脆弱的基类问题。如需继承，显式使用`open`关键字。



## Kotlin 高级特性

#### 高级函数是什么？有什么作用？

**定义**：参数或返回值是函数的函数。

**作用**：

- 简化代码，提高复用性
- 实现函数式编程模式
- 构建DSL

```kotlin
fun <T> List<T>.filter(predicate: (T) -> Boolean): List<T> {
    val result = mutableListOf<T>()
    for (item in this) if (predicate(item)) result.add(item)
    return result
}
list.filter { it > 5 }
```



#### Lambda 表达式和匿名函数有什么区别？

| **对比** | **Lambda**         | **匿名函数**               |
| :------- | :----------------- | :------------------------- |
| 语法     | `{ args -> body }` | `fun(args): Type { body }` |
| return   | 返回外层函数       | 返回自身                   |
| 参数类型 | 可省略             | 通常需指定                 |

```kotlin
val lambda: (Int, Int) -> Int = { a, b -> a + b }
val anonymous = fun(a: Int, b: Int): Int = a + b
```





#### 作用域函数有什么区别？

| **函数** | **上下文对象** | **返回值** | **适用场景**           |
| :------- | :------------- | :--------- | :--------------------- |
| let      | it             | lambda结果 | 非空检查、链式调用     |
| run      | this           | lambda结果 | 对象配置+计算结果      |
| with     | this           | lambda结果 | 多次调用同一对象方法   |
| apply    | this           | 上下文对象 | 对象初始化配置         |
| also     | it             | 上下文对象 | 附加操作（日志、验证） |



#### 委托是什么？有哪些应用场景？

**委托**：将某个操作交给另一个对象处理，Kotlin通过`by`关键字支持。

**应用场景**：

1. **类委托**：实现装饰器模式

   ```kotlin
   class MySet<T>(private val inner: Set<T>) : Set<T> by inner
   ```

2. **属性委托**：lazy、observable、vetoable

   ```kotlin
   val lazyValue: String by lazy { computeExpensiveValue() }
   ```



#### by lazy 延迟初始化是怎么工作的？

**工作原理**：

1. 第一次访问属性时执行lambda
2. 缓存结果
3. 后续访问直接返回缓存值
4. 默认线程安全（LazyThreadSafetyMode.SYNCHRONIZED）

```kotlin
val heavyObject: HeavyObject by lazy {
    HeavyObject()  // 只在首次使用时创建
}
```



#### Kotlin 的泛型和 Java 的泛型有什么区别？型变是什么？

**区别**：

| 对比       | Kotlin                | Java                  |
| ---------- | --------------------- | --------------------- |
| 通配符     | `out`/`in`            | `? extends`/`? super` |
| 声明处型变 | 支持                  | 不支持                |
| 具体化     | reified（inline函数） | 不支持                |

**型变**：泛型类型的子类型关系。

- **协变(out)**：`Producer<out T>`，只能读取
- **逆变(in)**：`Consumer<in T>`，只能写入
- **不变**：默认情况，既不能读也不能写不同类型





## 参考资料

[Kotlin面试题 - 面试鸭 | 2026最新企业真题+详细答案解析](https://www.mianshiya.com/tag/Kotlin?current=2&pageSize=20)











