## 一、基础概念与 JNI 机制

### 什么是 NDK？/ NDK 的作用是什么？

**考察点**：是否理解 NDK 的定位、能做什么、在项目中的使用场景（高性能计算、底层交互、复用 C/C++ 等）。

**答案**：NDK 是一套让 Android 应用能使用 C/C++ 编写 Native 代码并打包进 APK 的工具链。它提供交叉编译器、构建脚本（如 CMake）、头文件和库，使开发者可以编写和编译 so，并通过 JNI 与 Java/Kotlin 交互，常用于性能敏感逻辑、系统 API、音视频/图像处理、复用已有 C/C++ 库等场景。



### JNI（Java Native Interface）的作用是什么？

**考察点**：对 JNI 角色和“桥梁”概念的理解，能否说清 Java 与 Native 如何协作。

**答案**：JNI 是 Java 与 Native 代码（C/C++）之间的桥梁。它允许 Java 调用 Native 函数，也允许 Native 回调 Java 方法，实现跨语言交互。没有 JNI，Java 无法直接调用 so 里的 C/C++ 函数，Native 也无法访问 Java 对象或调用 Java 方法。



### JNI 中的 `JNIEnv*` 是什么？为什么不能跨线程使用？

**考察点**：是否理解 `JNIEnv` 的线程绑定特性，以及多线程下如何正确获取和使用 JNI。

**答案**：`JNIEnv*` 是指向 JNI 函数表的指针，提供所有 JNI 函数（如 `FindClass`、`CallVoidMethod`）的访问入口。它本质上是**线程局部**的，每个线程有各自的 `JNIEnv`。因此不能把在某个线程拿到的 `JNIEnv*` 传到另一线程使用。若在子线程中要用 JNI，必须通过 `JavaVM` 的 `AttachCurrentThread` 获取当前线程的 `JNIEnv`，用完后可调用 `DetachCurrentThread`。



### JNI 引用类型有哪些？如何避免内存泄漏？

**考察点**：对局部引用、全局引用、弱全局引用的区别与生命周期的掌握，以及防止引用表溢出和泄漏的实践。

**答案**：

- **局部引用**：在 Native 方法执行期间有效，方法返回后自动释放。若在循环或长时间逻辑中创建大量局部引用，应使用 `DeleteLocalRef` 及时释放，避免局部引用表溢出。
- **全局引用**：跨方法、跨线程有效，必须显式调用 `DeleteGlobalRef` 释放，否则会造成泄漏。
- **弱全局引用**：不阻止 GC 回收对象，使用前需用 `IsSameObject` 与 `NULL` 比较，或配合 `NewLocalRef` 检查对象是否已被回收。

避免泄漏：及时 `DeleteLocalRef`/`DeleteGlobalRef`，避免在 Native 层长期持有不必要的全局引用；注意 Direct Buffer、全局引用与循环引用的关系。



## 二、内存管理与性能优化

### Java 与 Native 层数据传递时，如何避免不必要的内存拷贝？

**考察点**：是否了解 JNI 数据传递的拷贝成本，以及 Direct Buffer、`GetDirectBufferAddress` 等零拷贝手段。

**答案**：使用**直接缓冲区（Direct Buffer）**。通过 `ByteBuffer.allocateDirect()` 分配的内存位于堆外，Native 可通过 `GetDirectBufferAddress` 拿到指针直接读写，无需经 JNI 再做一次拷贝。适合大块、高频传递的数据（如音视频帧、图像缓冲区）。需注意 Direct Buffer 的回收和生命周期，避免 Native 持有过久导致 GC 压力或误用已回收内存。



### 如何排查 JNI 层的内存泄漏？

**考察点**：是否掌握 Native 内存的观测手段和常见泄漏来源（全局引用、Direct Buffer、第三方 so 等）。

**答案**：

- **工具**：Android Studio 的 Memory Profiler 查看 Native Heap；`adb shell dumpsys meminfo <包名>` 看进程 Native 内存；LeakSanitizer（ASan）可检测 Native 泄漏。
- **代码层面**：检查全局引用是否在合适时机 `DeleteGlobalRef`；Direct Buffer 是否在不用时释放或不再被 Native 长期引用；是否存在循环引用或 so 内部静态/全局变量持有 Java 或 Native 资源不释放。



### 什么是 `RegisterNatives`？它比静态注册（按名称查找）有什么优势？

**考察点**：是否区分“按 JNI 命名规则自动查找”与“手动注册”，以及注册方式对性能、混淆、安全的影响。

**答案**：`RegisterNatives` 是**动态注册**：在运行时主动把 Native 函数与 Java 方法做映射。对比的是**静态注册**：按 `Java_包名_类名_方法名` 命名，由 JVM 在首次调用时按名称在 so 中查找符号。优势包括：**性能**：避免运行时按符号名查找；**安全/混淆**：Native 函数名不必暴露为固定格式，便于混淆；**灵活**：函数命名不受 JNI 命名规则限制，便于维护和封装。



## 三、多线程与并发

### 如何在 JNI 中创建和同步线程？

**考察点**：是否会在 Native 层使用 pthread 或等价机制，以及是否具备基本的线程同步意识。

**答案**：

- **创建**：使用 `pthread_create` 创建 POSIX 线程（或 C++ 的 `std::thread` 等）。
- **同步**：使用 `pthread_mutex`（互斥锁）、`pthread_cond`（条件变量）等保证对共享资源的访问是线程安全的。若与 Java 交互，需在该线程通过 `JavaVM::AttachCurrentThread` 获取 `JNIEnv` 后再调用 JNI。



### JNI 中如何正确处理异常？

**考察点**：是否知道在调用可能抛异常的 JNI 接口后检查并处理异常，以及异常未清除时不能继续调用其他 JNI 函数。

**答案**：

- **检查异常**：在调用可能抛出异常的 JNI 函数（如 `CallVoidMethod`、`FindClass`）后，用 `ExceptionCheck()` 或 `ExceptionOccurred()` 检查是否有 Java 异常。
- **处理方式**：可用 `ExceptionClear()` 清除后自行处理；或直接返回，让 Java 层捕获。**注意**：在异常未清除前，不应再调用其他 JNI 函数，否则行为未定义。



## 四、CMake 与构建系统

### CMake 中 `target_link_libraries` 与 `find_library` 的区别？

**考察点**：是否理解“查找库”与“把库链接到目标”是两个步骤，以及常见 CMake 指令的用途。

**答案**：

- **`find_library`**：在系统/NDK 中查找预编译库（如 `log`、`android`），将结果存到变量中，供后续使用。
- **`target_link_libraries`**：把已找到的库或本项目编译出的库**链接到**指定 target（可执行文件或动态库）。通常先 `find_library` 再在 `target_link_libraries` 中引用该变量。



### ABI（Application Binary Interface）是什么？如何适配不同 CPU 架构？

**考察点**：是否理解 ABI 含义，以及如何在构建和打包时选择/过滤架构以控制 APK 体积与兼容性。

**答案**：ABI 定义了二进制（如 so）与系统之间的调用约定、数据类型、寄存器用法等。Android 常见 ABI 有 `armeabi-v7a`、`arm64-v8a`、`x86`、`x86_64`。适配方式：在 CMake 中可通过 `ANDROID_ABI` 等变量区分架构；在 Gradle 中通过 `ndk.abiFilters` 或 `splits.abi` 指定要编译和打包的 ABI，未选中的架构不会打入 APK，从而减小体积。



## 五、实战与调试

面试官常会结合业务场景（如图像处理、音视频编解码、加密、跨平台 C++ 库集成）考察 NDK 的实际应用，建议准备 1～2 个有 Native 参与的实战项目，能说清选型原因、性能收益和遇到的问题与解决方式。

### Native 代码崩溃（如 SIGSEGV）后如何定位问题？

**考察点**：是否会用 tombstone、addr2line、ndk-stack 等工具把崩溃地址还原到源码行号。

**答案**：查看崩溃日志（tombstone 或 logcat 中的 backtrace），得到崩溃地址和 so 路径。使用 **ndk-stack**（传入 tombstone 或 logcat）或 **addr2line**（需对应架构的 toolchain，用 `-e <so路径> -f` 和崩溃地址）将地址解析为文件名和行号，再结合调用栈和业务逻辑定位根因（空指针、越界、use-after-free 等）。



### 如何实现 Java 与 Native 的双向通信？

**考察点**：能否完整描述“Java → Native”和“Native → Java”的两种方向及实现方式。

**答案**：

- **Java → Native**：在 Java 中用 `native` 声明方法，在 Native 层实现对应 JNI 函数（静态注册或 `RegisterNatives`），Java 调用该方法即进入 Native。
- **Native → Java**：通过当前线程的 `JNIEnv*` 使用 `CallMethod` 等系列函数调用 Java 方法；若需在其它线程或异步回调中调用，可先用 `NewGlobalRef` 持有 jclass/jobject，在目标线程 `AttachCurrentThread` 拿到 `JNIEnv` 后再调用并注意及时释放全局引用。



## 六、进阶问题（高级岗位）

### ART 与 Dalvik 对 JNI 的影响有何区别？

**考察点**：是否了解 ART 的 AOT、内联等对“按名称查找”的 JNI 可能带来的影响，以及为何推荐 `RegisterNatives`。

**答案**：ART 采用 AOT 将 Dex 编译为机器码，并会做方法内联等优化；Dalvik 主要为 JIT 解释执行。这些优化可能改变方法在运行时呈现的符号/行为，导致依赖“按固定方法名自动查找”的 JNI 在 ART 上不稳定。因此更推荐使用 `RegisterNatives` 显式注册，不依赖自动符号解析，兼容性和可维护性更好。



### 如何利用 NEON 指令进行性能优化？

**考察点**：是否了解 ARM SIMD（NEON）的概念及在 Android NDK 中的使用方式。

**答案**：NEON 是 ARM 的 SIMD（单指令多数据）扩展，一条指令可处理多路数据，适合图像处理、音视频编解码、数学运算等计算密集型场景。在 NDK 中可通过**编译器 intrinsics**（如 `<arm_neon.h>`）或**内联汇编**编写 NEON 代码，由编译器生成对应指令。使用时需注意 ABI（如 arm64-v8a）和运行时检测（可选），以兼顾兼容性与性能。
