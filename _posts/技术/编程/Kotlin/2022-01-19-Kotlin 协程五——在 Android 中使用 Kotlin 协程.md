---
layout: post
title: "Kotlin 协程五 —— 在Android 中使用 Kotlin 协程"
date:  2022-01-19 17:58:12 +0800
categories: ["技术", "编程", "Kotlin"]
tag: ["Kotlin", "协程"]
---

[toc]

## 一、Android MVVM 结构
Android 官方提供的架构图

![](/assets/images/技术/编程/kotlin/协程五/pic1.png)

## 二、添加依赖
如需在 Android 项目中使用协程，请将以下依赖项添加到应用的 build.gradle 文件中：

```
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.3.9")
}
```

## 三、在后台线程中执行
### 3.1 协程解决了什么问题
在安卓中，协程很好的解决了两个问题：

1. 耗时任务，运行时间过长阻塞主线程
2. 主线程安全，允许你在主线程中调用任意 suspend(挂起) 函数

获取网页，和 API 进行交互，都涉及到了网络请求。同样的，从数据库读取数据，从硬盘中加载图片，都涉及到了文件读取。这些就是我们所说的耗时任务。
为了避免在主线程中进行网络请求，一种通用的模式是使用 CallBack(回调)，它可以在将来的某一时间段回调进入你的代码。

```
class ViewModel: ViewModel() {
   fun getDataAndShow() {
       getApi() { result ->
           show(result)
       }
    }
}
```

尽管 getApi() 方法是在主线程调用的，但它会在另一个线程中进行网络请求。一旦网络请求的结果可用了，回调就会在主线程中被调用。这是处理耗时任务的一种好方式。
用协程来处理耗时任务可以简化代码。以上面的 fetchDocs() 方法为例，我们使用协程来重写之前的回调逻辑。

```
// Dispatchers.Main
suspend fun getDataAndShow() {
    // Dispatchers.IO
    val result = getApi()
    // Dispatchers.Main
    show(result)
}
// look at this in the next section
suspend fun getApi() = withContext(Dispatchers.IO){/*...*/}
```

协程中讲到，执行到 `withContext` 会挂起，执行完后会恢复。

- suspend —— 挂起当前协程的执行，保存所有局部变量
- resume —— 从被挂起协程挂起的地方继续执行

>协程的挂起和恢复共同工作来替代回调。使得我们用同步的方式来写异步的代码。

### 3.2 保证主线程安全
使用 suspend 并不意味着告诉 Kotlin 一定要在后台线程运行函数。为了让一个函数不会使主线程变慢，我们可以告诉 Kotlin 协程使用 Default 或者 IO 调度器。

- Room 在你使用 挂起函数 、RxJava 、LiveData 时自动提供主线程安全。
- Retrofit 和 Volley 等网络框架一般自己管理线程调度，当你使用 Kotlin 协程的时候不需要再显式保证主线程安全。

通过协程，可以细粒度的控制线程调度，因为 withContext 让你可以控制任意一行代码运行在什么线程上，而不用引入回调来获取结果。可将其应用在很小的函数中，例如数据库操作和网络请求。所以，比较好的做法是，使用 withContext 确保每个函数在任意调度器上执行都是安全的，包括 Main，这样调用者在调用函数时就不需要考虑应该运行在什么线程上。

>编写良好的挂起函数被任意线程调用都应该是安全的。

### 3.3 withContext 的性能
如果一个函数将对数据库进行10次调用，那么您可以告诉 Kotlin 在外部的 withContext 中调用一次切换。尽管数据库会重复调用 withContext ，但是他它将在同一个调度器下，寻找最快路径。此外，Dispatchers.Default 和 Dispatchers.IO 之间的协程切换已经过优化，以尽可能避免线程切换。

## 四、结构化并发
### 4.1 追踪协程
使用代码手动追踪一千个协程的确是很困难的。你可以尝试去追踪它们，并且手动保证它们最后会完成或者取消，但是这样的代码冗余，而且容易出错。如果你的代码不够完美，你将失去对一个协程的追踪，我把它称之为任务泄露。

>泄露的协程会浪费内存，CPU，磁盘，甚至发送一个不需要的网络请求。

为了避免泄露协程，Kotlin 引入了 structured concurrency(结构化并发)。结构化并集合了语言特性和最佳实践，遵循这个原则将帮助你追踪协程中的所有任务。

在 Android 中，我们使用结构化并发可以做三件事：

1. 取消不再需要的任务
2. 追踪所有正在进行的任务
3. 协程失败时的错误信号

### 4.2 通过作用域取消任务
在 Kotlin 中，协程必须运行在 CoroutineScope 中。CoroutineScope 会追踪你的协程，即使协程已经被挂起。为了保证所有的协程都被追踪到，Kotlin 不允许你在没有 CoroutineScope 的情况下开启新的协程。你可以把 CoroutineScope 想象成具有特殊能力的轻量级的 ExecutorServicce。它赋予你创建新协程的能力，这些协程都具备我们在上篇文章中讨论过的挂起和恢复的能力。CoroutineScope 会追踪所有的协程，并且它也可以取消所有由他开启的协程。这很适合 Android 开发者，当用户离开当前页面后，可以保证清理掉所有已经开启的东西。

>CoroutineScope 会追踪所有的协程，并且它也可以取消所有由他开启的协程。

>值得注意的是，已经取消的作用域，不能再启动新协程，如果只是想取消作用域内的某个协程，需要使用协程的 Job 对其进行取消。

#### 4.2.1 启动新协程
启动协程有两种方法，且有不同的用法：

1. 使用 launch 协程构建器启动一个新的协程，这个协程是没返回值的
2. 使用 async 协程构建器启动一个新的协程，它允许你返回一个结果，通过挂起函数 await 来获取。

在大多数情况下，如何从一个普通函数启动协程的答案都是使用 launch，因为普通函数是不能调用 await 的。准确的说，是普通函数不能调用挂起函数。

```
fun foo(scope: CoroutineScope) {
    scope.launch {
        test()  // 允许
    }

    val task = scope.async { }
    task.await() // 不允许
    
    test() // 不允许
}
```

launch 连接了普通函数中的代码和协程的世界。在 launch 内部，你可以调用挂起函数。因为 launch 启动了一个协程。

这很好理解，挂起函数只能直接或者间接地在协程中被调用。

>Launch 是把普通函数带进协程世界的桥梁。

>提示：launch 和 async 很大的一个区别是异常处理。async 期望你通过调用 await 来获取结果（或异常），所以它默认不会抛出异常。这就意味着使用 async 启动新的协程，它会悄悄的把异常丢弃。

假设我们编写了一个 suspend 方法，出现了异常

```
val unrelatedScope = MainScope()

// example of a lost error
suspend fun lostError() {
    // async without structured concurrency
    unrelatedScope.async {
        throw InAsyncNoOneCanHearYou("except")
    }
}
```

注意，上面的代码中声明了一个未经关联的协程作用域，并且未通过结构化并发启动新协程。
上面代码中的错误会丢失，因为 async 认为你会调用 await，这时候会重新抛出异常。但是如果你没有调用 await，这个错误将永远被保存，静静的等待被发现。

如果我们使用结构化并发写上面的代码，异常将会正确的抛给调用者。

```
suspend fun foundError() {
    coroutineScope {
        async {
            throw StructuredConcurrencyWill("throw")
        }
    }
}
```

由于 coroutineScope 会等待所有子协程执行完成，所以当子协程失败时它也会知道。当 coroutineScope 启动的协程抛出了异常，coroutineScope 会将异常扔给调用者。如果使用 coroutineScope 代替 supervisorScope，当异常抛出时，会立刻停止所有的子协程。

>结构化并发保证当一个协程发生错误，它的调用者或者作用域可以发现。

#### 4.2.2 在 ViewModel 中启动
如果一个 CoroutineScope 追踪在其中启动的所有协程，launch 会新建一个协程，那么你应该在何处调用 launch 并将其置于协程作用域中呢？还有，你应该在什么时候取消在作用域中启动的所有协程呢？
在 Android 中，通常将 CoroutineScope 和用户界面相关联起来。这将帮助你避免协程泄露，并且使得用户不再需要的 Activity 或者 Fragment 不再做额外的工作。当用户离开当前页面，与页面相关联的 CoroutineScope 将取消所有工作。

>结构化并发保证当协程作用域取消，其中的所有协程都会取消。

当通过 Android Architecture Components 集成协程时，一般都是在 ViewModel 中启动协程。这里是许多重要任务开始工作的地方，并且你不必担心旋转屏幕会杀死协程。
通过 viewModelScope 启动协程，当 viewModelScope 被清除（即 onCleared() 被调用）时，它会自动取消由它启动的所有协程。

当你需要协程和 ViewModel 的生命周期保持一致时，使用 viewModelScope 来从普通函数切换到协程。那么，由于 viewModelScope 会自动取消协程，可以确保任何工作，即使是死循环，都能在不再需要执行的时候将其取消。

### 4.3 使用结构化并发
使用未关联的 CoroutineScope（注意是大写字母 C），或者使用全局作用域 GlobalScope ，会导致非结构化并发。只有在少数情况下，你需要协程的生命周期长于调用者的作用域时，才考虑使用非结构化并发。通常情况下，你都应该使用结构化并发来追踪协程，处理异常，拥有良好的取消机制。

结构化并发帮助我们解决的三个问题：

1. 取消不再需要的任务
2. 追踪所有正在进行的任务
3. 协程失败时的错误信号

结构化并发给予我们如下保证：

1. 当作用域取消，其中的协程也会取消
2. 当挂起函数返回，其中的所有任务都已完成
3. 当协程发生错误，其调用者会得到通知

这些加在一起，使得我们的代码更加安全，简洁，并且帮助我们避免任务泄露。

## 五、 Android中使用协程的一些最佳做法
### 5.1 注入调度器
在创建新协程或调用 `withContext` 时，请勿对 `Dispatchers` 进行硬编码。

```
// DO inject Dispatchers
class NewsRepository(
    private val defaultDispatcher: CoroutineDispatcher = Dispatchers.Default
) {
    suspend fun loadNews() = withContext(defaultDispatcher) { /* ... */ }
}

// DO NOT hardcode Dispatchers
class NewsRepository {
    // DO NOT use Dispatchers.Default directly, inject it instead
    suspend fun loadNews() = withContext(Dispatchers.Default) { /* ... */ }
}
```

这种依赖项注入模式可以降低测试难度，因为您可以使用 TestCoroutineDispatcher 替换单元测试和插桩测试中的这些调度程序，以提高测试的确定性。

### 5.2 挂起函数应该保证线程安全
即 3.2 所说的内容，挂起函数应该保证对任意线程安全，不应该由挂起函数的调用方来切换线程。

```
class NewsRepository(private val ioDispatcher: CoroutineDispatcher) {

    // As this operation is manually retrieving the news from the server
    // using a blocking HttpURLConnection, it needs to move the execution
    // to an IO dispatcher to make it main-safe
    suspend fun fetchLatestNews(): List<Article> {
        withContext(ioDispatcher) { /* ... implementation ... */ }
    }
}

// This use case fetches the latest news and the associated author.
class GetLatestNewsWithAuthorsUseCase(
    private val newsRepository: NewsRepository,
    private val authorsRepository: AuthorsRepository
) {
    // This method doesn't need to worry about moving the execution of the
    // coroutine to a different thread as newsRepository is main-safe.
    // The work done in the coroutine is lightweight as it only creates
    // a list and add elements to it
    suspend operator fun invoke(): List<ArticleWithAuthor> {
        val news = newsRepository.fetchLatestNews()

        val response: List<ArticleWithAuthor> = mutableEmptyList()
        for (article in news) {
            val author = authorsRepository.getAuthor(article.author)
            response.add(ArticleWithAuthor(article, author))
        }
        return Result.Success(response)
    }
}
```

此模式可以提高应用的可伸缩性，因为调用挂起函数的类无需担心使用哪个 Dispatcher 来处理哪种类型的工作。该责任将由执行相关工作的类承担。

### 5.3 ViewModel 应创建协程
ViewModel 类应首选创建协程，而不是公开挂起函数来执行业务逻辑。如果只需要发出一个值，而不是使用数据流公开状态，ViewModel 中的挂起函数就会非常有用。

```
// DO create coroutines in the ViewModel
class LatestNewsViewModel(
    private val getLatestNewsWithAuthors: GetLatestNewsWithAuthorsUseCase
) : ViewModel() {

    private val _uiState = MutableStateFlow<LatestNewsUiState>(LatestNewsUiState.Loading)
    val uiState: StateFlow<LatestNewsUiState> = _uiState

    fun loadNews() {
        viewModelScope.launch {
            val latestNewsWithAuthors = getLatestNewsWithAuthors()
            _uiState.value = LatestNewsUiState.Success(latestNewsWithAuthors)
        }
    }
}

// Prefer observable state rather than suspend functions from the ViewModel
class LatestNewsViewModel(
    private val getLatestNewsWithAuthors: GetLatestNewsWithAuthorsUseCase
) : ViewModel() {
    // DO NOT do this. News would probably need to be refreshed as well.
    // Instead of exposing a single value with a suspend function, news should
    // be exposed using a stream of data as in the code snippet above.
    suspend fun loadNews() = getLatestNewsWithAuthors()
}
```

视图不应直接触发任何协程来执行业务逻辑，而应将这项工作委托给 ViewModel。这样一来，业务逻辑就会变得更易于测试，因为可以对 ViewModel 对象进行单元测试，而不必使用测试视图所必需的插桩测试。

此外，如果工作是在 viewModelScope 中启动，您的协程将在配置更改后自动保留。如果您改用 lifecycleScope 创建协程，则必须手动进行处理该操作。如果协程的存在时间需要比 ViewModel 的作用域更长，请查看“在业务和数据层中创建协程”部分。

直白点理解就是业务逻辑，应该在 ViewModel 中启动协程处理，而不是在 View 中。

>注意：视图应对与界面相关的逻辑启动协程。例如，从互联网提取映像或设置字符串格式。

### 5.4 不要公开可变类型
最好向其他类公开不可变类型。这样一来，对可变类型的所有更改都会集中在一个类中，便于在出现问题时进行调试。

```
// DO expose immutable types
class LatestNewsViewModel : ViewModel() {

    private val _uiState = MutableStateFlow(LatestNewsUiState.Loading)
    val uiState: StateFlow<LatestNewsUiState> = _uiState

    /* ... */
}

class LatestNewsViewModel : ViewModel() {

    // DO NOT expose mutable types
    val uiState = MutableStateFlow(LatestNewsUiState.Loading)

    /* ... */
}
```

### 5.5 数据层和业务层应公开挂起函数和数据流
数据层和业务层中的类通常会公开函数以执行一次性调用，或接收数据随时间变化的通知。这些层中的类应该针对一次性调用公开挂起函数，并公开数据流以接收关于数据更改的通知。

```
// Classes in the data and business layer expose
// either suspend functions or Flows
class ExampleRepository {
    suspend fun makeNetworkRequest() { /* ... */ }

    fun getExamples(): Flow<Example> { /* ... */ }
}
```

采用该最佳做法后，调用方（通常是演示层）能够控制这些层中发生的工作的执行和生命周期，并在需要时取消相应工作。

在业务层和数据层中创建协程
对于数据层或业务层中因不同原因而需要创建协程的类，它们可以选择不同的选项。

如果仅当用户查看当前屏幕时，要在这些协程中完成的工作才具有相关性，则应遵循调用方的生命周期。在大多数情况下，调用方将是 ViewModel。在这种情况下，应使用 coroutineScope 或 supervisorScope。

```
class GetAllBooksAndAuthorsUseCase(
    private val booksRepository: BooksRepository,
    private val authorsRepository: AuthorsRepository,
    private val defaultDispatcher: CoroutineDispatcher = Dispatchers.Default
) {
    suspend fun getBookAndAuthors(): BookAndAuthors {
        // In parallel, fetch books and authors and return when both requests
        // complete and the data is ready
        return coroutineScope {
            val books = async(defaultDispatcher) {
                booksRepository.getAllBooks()
            }
            val authors = async(defaultDispatcher) {
                authorsRepository.getAllAuthors()
            }
            BookAndAuthors(books.await(), authors.await())
        }
    }
}
```

如果只要应用处于打开状态，要完成的工作就具有相关性，并且此工作不限于特定屏幕，那么此工作的存在时间应该比调用方的生命周期更长。对于这种情况，您应使用外部 CoroutineScope。

```
class ArticlesRepository(
    private val articlesDataSource: ArticlesDataSource,
    private val externalScope: CoroutineScope,
    private val defaultDispatcher: CoroutineDispatcher = Dispatchers.Default
) {
    // As we want to complete bookmarking the article even if the user moves
    // away from the screen, the work is done creating a new coroutine
    // from an external scope
    suspend fun bookmarkArticle(article: Article) {
        externalScope.launch(defaultDispatcher) {
            articlesDataSource.bookmarkArticle(article)
        }
            .join() // Wait for the coroutine to complete
    }
}
```

externalScope 应由存在时间比当前屏幕更长的类进行创建和管理，并且可由 Application 类或作用域限定为导航图的 ViewModel 进行管理。

### 5.6 在测试中注入 TestCoroutineDispatcher
应在测试内的类中注入 TestCoroutineDispatcher 的实例。TestCoroutineDispatcher 会立即执行任务，并让您能够控制协程在测试中执行的时机。

在测试主体中使用 TestCoroutineDispatcher 的 runBlockingTest，以等待所有使用相应调度程序的协程完成。

```
class ArticlesRepositoryTest {

    private val testDispatcher = TestCoroutineDispatcher()

    @Test
    fun testBookmarkArticle() {
        // Execute all coroutines that use this Dispatcher immediately
        testDispatcher.runBlockingTest {
            val articlesDataSource = FakeArticlesDataSource()
            val repository = ArticlesRepository(
                articlesDataSource,
                // Make the CoroutineScope use the same dispatcher
                // that we use for runBlockingTest
                CoroutineScope(testDispatcher),
                testDispatcher
            )
            val article = Article()
            repository.bookmarkArticle(article)
            assertThat(articlesDataSource.isBookmarked(article)).isTrue()
        }
        // make sure nothing else is scheduled to be executed
        testDispatcher.cleanupTestCoroutines()
    }
}
```

由于被测类创建的所有协程都使用同一 TestCoroutineDispatcher，并且测试主体会使用 runBlockingTest 等待协程执行，因此您的测试将变得具有确定性，并且不会受到竞态条件的影响。

>注意：如果被测代码中未使用其他 Dispatchers，则会出现以上情况。因此，我们不建议在类中对 Dispatchers 进行硬编码。如果需要注入多个 Dispatchers，您可以传递 TestCoroutineDispatcher 的同一实例。

### 5.7 避免使用 GlobalScope
这类似于“注入调度程序”最佳做法。通过使用 GlobalScope，您将对类使用的 CoroutineScope 进行硬编码，而这会带来一些问题：

1. 提高硬编码值。如果您对 GlobalScope 进行硬编码，则可能同时对 Dispatchers 进行硬编码。
2. 这会让测试变得非常困难，因为您的代码是在非受控的作用域内执行的，您将无法控制其执行。
3. 您无法设置一个通用的 CoroutineContext 来对内置于作用域本身的所有协程执行。

而您可以考虑针对存在时间需要比当前作用域更长的工作注入一个 CoroutineScope。如需详细了解此主题，请参阅“在业务层和数据层中创建协程”部分。

### 5.8 将协程设为可取消
协程取消属于协作操作，也就是说，在协程的 Job 被取消后，相应协程在挂起或检查是否存在取消操作之前不会被取消。如果您在协程中执行阻塞操作，请确保相应协程是可取消的。
例如，如果您要从磁盘读取多个文件，请先检查协程是否已取消，然后再开始读取每个文件。若要检查是否存在取消操作，有一种方法是调用 ensureActive 函数。

```
someScope.launch {
    for(file in files) {
        ensureActive() // Check for cancellation
        readFile(file)
    }
}
```

kotlinx.coroutines 中的所有挂起函数（例如 withContext 和 delay）都是可取消的。如果您的协程调用这些函数，您无需执行任何其他操作。

### 5.9 留意异常
不当处理协程中抛出的异常可能导致您的应用崩溃。如果可能会发生异常，请在使用 viewModelScope 或 lifecycleScope 创建的任何协程内容中捕获相应异常。

```
class LoginViewModel(
    private val loginRepository: LoginRepository
) : ViewModel() {

    fun login(username: String, token: String) {
        viewModelScope.launch {
            try {
                loginRepository.login(username, token)
                // Notify view user logged in successfully
            } catch (error: Throwable) {
                // Notify view login attempt failed
            }
        }
    }
}
```