---
layout: post
title:  "测试"
parent: "Coroutines"
nav_order: 3
---

# 协程相关测试

先加入依赖

```kotlin
dependencies {
    testImplementation "org.jetbrains.kotlinx:kotlinx-coroutines-test:$coroutines_version"
}

```
对于使用协程的测试，请使用 ```runTest```(当然```runBlocking```也是可以的，但没有```runTest```那么推荐)

由于 JUnit 测试函数本身并不是挂起函数，因此您需要在测试中调用协程构建器以启动新的协程。
runTest 是用于测试的协程构建器。使用此构建器可封装包含协程的任何测试。请注意，协程不仅可以直接在测试主体中启动，还可以通过测试中使用的对象启动。
```kotlin
suspend fun fetchData(): String {
    delay(1000L)
    return "Hello world"
}

@Test
fun dataShouldBeHelloWorld() = runTest {
    val data = fetchData()
    assertEquals("Hello world", data)
}
```

## 在测试中的调度器
这里使用的是在测试代码中注入主调度器，可以在测试中使需要mock的协程部分的代码正常运行。

```kotlin
class HomeViewModel : ViewModel() {
    private val _message = MutableStateFlow("")
    val message: StateFlow<String> get() = _message

    fun loadMessage() {
        viewModelScope.launch {
            _message.value = "Greetings!"
        }
    }
}
```

```kotlin
class HomeViewModelTest {
    @Test
    fun settingMainDispatcher() = runTest {
        val testDispatcher = UnconfinedTestDispatcher()
        Dispatchers.setMain(testDispatcher)

        try {
            val viewModel = HomeViewModel()
            viewModel.loadMessage() // Uses testDispatcher, runs its coroutine eagerly
            assertEquals("Greetings!", viewModel.message.value)
        } finally {
            Dispatchers.resetMain()
        }
    }
}
```

当然一个测试文件中有多个协程测试方法时候，使用以下写法设置测试协程的调度器，会更加简单。

```kotlin
@ExperimentalCoroutinesApi
private val testDispatcher = TestCoroutineDispatcher()

@Before
fun before() {
    Dispatchers.setMain(testDispatcher)
}

@After
fun tearDown() {
    Dispatchers.resetMain()
}
```