---
layout: default
title: "使用"
parent: "Coroutines"
nav_order: 2
---

* TOC
{:toc}

## 启动一个协程
```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    CoroutineScope(Dispatchers.Main).launch {
        delay(SECOND_DURATION)
    }
}
```
以上代码就是在主线程之中开启了协程，完成了一个延时的简单功能。

这里说一下协程存在的四个基础概念：
- suspend function。即挂起函数，比如```delay()``` 就是协程库提供的一个用于实现非阻塞式延时的挂起函数。
- CoroutineScope。即协程作用域。比如GlobalScope 是 CoroutineScope 的一个实现类，用于指定协程的作用范围，可用于管理多个协程的生命周期，所有协程都需要通过 CoroutineScope协程作用域 来启动。这个作用域一般不建议使用全局的GlobalScope作用域。
- CoroutineContext。即协程上下文，包含多种类型的配置参数。Dispatchers.Main 就是 CoroutineContext 这个抽象概念的一种实现，用于指定协程的运行载体，用于指定协程要运行在哪类线程上。**即协程切走后，切回到哪类线程上**。Dispatchers线程调度器一般分为：
1. Dispatchers.Main： 使用此调度程序可在 Android 主线程上运行协程。此调度程序只能用于与界面交互和执行快速工作。示例包括调用 suspend 函数，运行 Android 界面框架操作，以及更新 LiveData 对象。
2. Dispatchers.IO： 此调度程序经过了专门优化，适合在主线程之外执行磁盘或网络 I/O。示例包括使用 Room 组件、从文件中读取数据或向文件中写入数据，以及运行任何网络操作
3. Dispatchers.Default： 此调度程序经过了专门优化，适合在主线程之外执行占用大量 CPU 资源的工作。用例示例包括对列表排序和解析 JSON

- CoroutineBuilder。即协程构建器，协程在 CoroutineScope 的上下文中通过 launch、async 等协程构建器来进行声明并启动。launch、async 均被声明为 CoroutineScope 的扩展方法。

# suspend 关键字

suspend关键字是kotlin协程之中修饰挂起函数的关键字，kotlin代码中，只有挂起函数和协程能调用另一个挂起函数。**挂起，就是一个稍后会被自动切回来的线程调度操作。**
这个挂起之后，切回来的动作，在 Kotlin 里就叫做 resume，恢复。

这里拿delay挂起函数举例。```delay()``` 函数类似于 Java 中的 ```Thread.sleep()```，而之所以说 ```delay()``` 函数是非阻塞的，是因为它和单纯的线程休眠有着本质的区别。例如，当在 ThreadA 上运行的 CoroutineA 调用了delay(1000L)函数指定延迟一秒后再运行，ThreadA 会转而去执行 CoroutineB，等到一秒后再来继续执行 CoroutineA。所以，ThreadA 并不会因为 CoroutineA 的延时而阻塞，而是能继续去执行其它任务，所以挂起函数并不会阻塞其所在线程，这样就极大地提高了线程的并发灵活度，最大化了线程的利用效率。而如果是使用```Thread.sleep()```的话，线程就只能干等着而不能去执行其它任务，降低了线程的利用效率。

协程是运行于线程上的，一个线程可以运行多个（几千上万个）协程。线程的调度行为是由操作系统来管理的，而协程的调度行为是可以由开发者来指定并由编译器来实现的，协程能够细粒度地控制多个任务的执行时机和执行线程，当线程所执行的当前协程被 suspend 后，该线程也可以腾出资源去处理其他任务。

## suspend挂起与恢复
以下示例展示了一项任务（假设 get 方法是一个网络请求任务）的简单协程实现：
```kotlin
suspend fun fetchDocs() {                             // Dispatchers.Main
    val result = get("https://developer.android.com") // Dispatchers.IO for `get`
    show(result)                                      // Dispatchers.Main
}

suspend fun get(url: String) = withContext(Dispatchers.IO) {
     /* ... */ 
}
```

在上面的示例中，```get()``` 仍在主线程上被调用，但它会在启动网络请求之前**暂停协程也就是挂起**。```get()``` 主体内通过调用 withContext(Dispatchers.IO) 创建了一个在 IO 线程池中运行的代码块，在该块内的任何代码都始终通过 IO 调度器执行。当网络请求完成后，```get()``` 会恢复已暂停的协程也**就是恢复（切回）**，使得主线程协程可以直接拿到网络请求结果而不用使用回调来通知主线程。
Kotlin 使用 堆栈帧 来管理要运行哪个函数以及所有局部变量。暂停协程时，系统会复制并保存当前的堆栈帧以供稍后使用。恢复时，会将堆栈帧从其保存的位置复制回来，然后函数再次开始运行。虽然代码可能看起来像普通的顺序阻塞请求，但由协程来确保不会阻塞线程。
在主线程进行的 暂停协程 和 恢复协程 的两个操作，既实现了将耗时任务交由后台线程完成，保障了主线程安全，又以同步代码的方式完成了实际上的多线程异步调用。可以说，在 Android 平台上协程主要就用来解决两个问题：

- 处理耗时任务 (Long running tasks)，这种任务常常会阻塞主线程
- 保证主线程安全 (Main-safety)，即确保安全地从主线程调用任何 suspend 函数

# CoroutineScope
CoroutineScope 即 协程作用域，用于对协程进行追踪。如果我们启动了多个协程但是没有一个可以对其进行统一管理的途径的话，就会导致我们的代码臃肿杂乱，甚至发生内存泄露或者任务泄露。为了确保所有的协程都会被追踪，Kotlin 不允许在没有 CoroutineScope 的情况下启动协程。CoroutineScope 可被看作是一个具有超能力的 ExecutorService（Java 线程池） 的轻量级版本。

所有的协程都需要通过 CoroutineScope 来启动，它会跟踪通过 launch 或 async 创建的所有协程，你可以随时调用 scope.cancel() 取消正在运行的协程。CoroutineScope 本身并不运行协程，它只是确保你不会失去对协程的追踪，即使协程被挂起也是如此。在 Android 中，某些 ktx 库为某些生命周期类提供了自己的 CoroutineScope，例如，ViewModel 有 viewModelScope，Lifecycle 有 lifecycleScope
CoroutineScope 大体上可以分为三种：
1. GlobalScope。即全局协程作用域，在这个范围内启动的协程可以一直运行直到应用停止运行。GlobalScope 本身不会阻塞当前线程，且启动的协程相当于守护线程，不会阻止 JVM 结束运行
2. runBlocking。一个顶层函数，和 GlobalScope 不一样，它会阻塞当前线程直到其内部所有相同作用域的协程执行结束
3. 自定义 CoroutineScope。可用于实现主动控制协程的生命周期范围，对于 Android 开发来说最大意义之一就是可以在 Activity、Fragment、ViewModel 等具有生命周期的对象中按需取消所有协程任务，从而确保生命周期安全，避免内存泄露

# 一些协程函数和api

## runBlocking函数
使用 runBlocking 这个顶层函数来启动协程，展示如下：
```kotlin
fun main() {
    log("start")
    runBlocking {
        launch {
            repeat(3) {
                delay(100)
                log("launchA - $it")
            }
        }
        launch {
            repeat(3) {
                delay(100)
                log("launchB - $it")
            }
        }
        GlobalScope.launch {
            repeat(3) {
                delay(120)
                log("GlobalScope - $it")
            }
        }
    }
    log("end")
}
```
打印结果是：
```kotlin
[main] start
[main] launchA - 0
[main] launchB - 0
[DefaultDispatcher-worker-1] GlobalScope - 0
[main] launchA - 1
[main] launchB - 1
[DefaultDispatcher-worker-1] GlobalScope - 1
[main] launchA - 2
[main] launchB - 2
[main] end
```
想必看出，```runBlocking``` 的一个特别之处就是：只有当内部相同作用域的所有协程都运行结束后，声明在 ```runBlocking``` 之后的代码才能执行，即 ```runBlocking``` 会阻塞其所在线程，但其内部运行的协程是非阻塞的。下面例子解释一下这个情况：
```kotlin
fun main() {
    GlobalScope.launch(Dispatchers.IO) {
        delay(600)
        log("GlobalScope")
    }
    runBlocking {
        delay(500)
        log("runBlocking")
    }
    //主动休眠两百毫秒，使得和 runBlocking 加起来的延迟时间多于六百毫秒
    Thread.sleep(200)
    log("after sleep")
}
```
打印结果是：
```kotlin
[main] runBlocking
[DefaultDispatcher-worker-1] GlobalScope
[main] after sleep
```

## coroutineScope函数
coroutineScope 函数用于创建一个独立的协程作用域，直到所有启动的协程都完成后才结束自身。runBlocking 和 coroutineScope 看起来很像，因为它们都需要等待其内部所有相同作用域的协程结束后才会结束自己。两者的主要区别在于 ```runBlocking``` 方法会阻塞当前线程，而 coroutineScope不会，而是会挂起并释放底层线程以供其它协程使用。基于这个差别，```runBlocking``` 是一个普通函数，而 coroutineScope 是一个挂起函数。
```kotlin
fun main() = runBlocking {
    launch {
        delay(100)
        log("Task from runBlocking")
    }
    coroutineScope {
        launch {
            delay(500)
            log("Task from nested launch")
        }
        delay(50)
        log("Task from coroutine scope")
    }
    log("Coroutine scope is over")
}
```
打印结果是：
```kotlin
[main] Task from coroutine scope
[main] Task from runBlocking
[main] Task from nested launch
[main] Coroutine scope is over
```

## supervisorScope函数
supervisorScope 函数用于创建一个使用了 SupervisorJob 的 coroutineScope，该作用域的特点就是抛出的异常不会连锁取消同级协程和父协程。
```kotlin
fun main() = runBlocking {
    launch {
        delay(100)
        log("Task from runBlocking")
    }
    supervisorScope {
        launch {
            delay(500)
            log("Task throw Exception")
            throw Exception("failed")
        }
        launch {
            delay(600)
            log("Task from nested launch")
        }
    }
    log("Coroutine scope is over")
}
```
打印结果是：
```kotlin
[main] Task from runBlocking
[main] Task throw Exception
[main] Task from nested launch
[main] Coroutine scope is over

```

## withContext
对于以下代码，get方法内使用withContext(Dispatchers.IO) 创建了一个指定在 IO 线程池中运行的代码块，该区间内的任何代码都始终通过 IO 线程来执行。由于 withContext 方法本身就是一个挂起函数，因此 get 方法也必须定义为挂起函数。

```kotlin
suspend fun fetchDocs() {                      // Dispatchers.Main
    val result = get("developer.android.com")  
    show(result)                               // Dispatchers.Main
}

suspend fun get(url: String) =                 // Dispatchers.Main
    withContext(Dispatchers.IO) {              // Dispatchers.IO (main-safety block)
        /* perform network IO here */          // Dispatchers.IO (main-safety block)
}                                              // Dispatchers.Main

```

## withTimeout
```withTimeout``` 函数用于指定协程的运行超时时间，如果超时则会抛出 TimeoutCancellationException，从而令协程结束运行
```kotlin
fun main() = runBlocking {
    log("start")
    val result = withTimeout(300) {
        repeat(5) {
            delay(100)
        }
        200
    }
    log(result)
    log("end")
}
```

## 启动一个协程成员变量并取消
```kotlin
private var myJob: Job? = null

// 给协程变量赋予内容
private fun startJob() {
    myJob.cancel() // 每次启动全局协程之时，可能会需要取消之前的那个协程。
    myJob = viewModelScope.launch {
        delay(PLAY_HANDLE_DISAPPEAR_TIME)
        _playState.update {
            it.copy(
                displayButton = false,
            )
        }
    }
}

fun main() {
    // xxxx
    startJob() // 此处若需要协程方法则开启。
    myJob?.cancel() // 此处若不需要协程，则可以在此处取消。
}
```

## 协程的异常处理
当一个协程由于异常而运行失败时，它会传播这个异常并传递给它的父协程。接下来，父协程会进行下面几步操作：
- 取消它自己的子级
- 取消它自己
- 将异常传播并传递给它的父级

## compose中启动一些协程
让Composable支持协程的重要意义是，可以让一些简单的业务逻辑直接Composable的形式封装并实现复用，而无需额外借助ViewModel。

一些api函数如下：

- LaunchedEffect
LaunchedEffect函数，可在compose的组合项内安全调用挂起函数。可以传入一个 key 值，当 key 改变时 LaunchedEffect 会重启。当compose组件重组时，对应的协程将被自动取消，并在新的协程中启动新的挂起函数。不需要像使用```DisposableEffect```调用onDispose结束生命周期和销毁操作。

```kotlin
var searchQuery by remember { mutableStateOf("") }
LaunchedEffect(searchQuery) {
    // execute search and receive result
}
```
如上，当检索词变化时，发起检索。

- rememberCoroutineScope
rememberCoroutineScope也是让我们在compose里可以使用协程的函数api。如下，是一个简单使用例子。
```kotlin
val scope = rememberCoroutineScope()
Column(modifier = Modifier.padding(16.dp)) {
    scope.launch(Dispatchers.Main) {
            delay(100)
        }
    }
```

## Android ktx的协程
- ViewModel ktx 库提供了一个 viewModelScope，用于在 ViewModel 中启动协程，该作用域的生命周期和 ViewModel 相等，当 ViewModel 回调了 onCleared()方法时会自动取消该协程作用域。
- Lifecycle ktx 为每个 Lifecycle 对象（Activity、Fragment、Process 等）定义了一个 LifecycleScope，该作用域具有生命周期安全的保障，在此范围内启动的协程会在 Lifecycle 被销毁时同时取消，可以使用 lifecycle.coroutineScope 或 lifecycleOwner.lifecycleScope 属性来拿到该 CoroutineScope。