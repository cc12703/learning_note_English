

# Flows

* https://developer.android.com/kotlin/flow


[TOC]



## 综述

### 概述
* 在协程中，flow是一种用于序列化发送至的数据类型
    * suspend函数刚好相反，只会返回一个单一值
    * 例子：你可以使用flow来接收数据库的实时更新
* flow构建在协程的基础上，可以提供多个值
* 一个flow概念上来说是一个可以进行异步计算的数据流
    * 发送的数据必须是相同的类型
    * 例子：Flow<Int> 是一个发送整型值的flow
* 一个flow和一个迭代器类似，用于产生一个序列化的值
    * 但是flow使用suspend函数来异步地生产和消费值
    * 例子：flow可以安全的发起网络请求来生产值，而不阻塞主线程


### 涉及数据流的实体
* 生产者：生产数据，并加入到流中
    * 得益于协程，flow可以异步的生产数据
* 中间者(可选)：可以修改被发送进流中的每一个值
* 消费者：从流中消费值

#### 图示
![](https://gitee.com/cc12703/figurebed/raw/master/img/20210514143756.png)


### 举例
* 在android中，一个数据源或仓库是一个典型的界面数据的生产者
    * 视图作为消费者最终显示这些数据
* 另一方面，视图可以作为用户输入事件的生产者
    * 体系层次中的其他层会消费这些事件
* 在生产者和消费者之间的层经常会扮演中间者的角色，修改流中的数据来达到后续层的要求



### 创建flow
* 要创建flow，可以使用flow构建器接口
    * flow构建器会创建一个新的flow
    * 使用emit函数来手动向数据流中发送新生成的值

#### 示例
```kotlin
class NewsRemoteDataSource(
    private val newsApi: NewsApi,
    private val refreshIntervalMs: Long = 5000
) {
    val latestNews: Flow<List<ArticleHeadline>> = flow {
        while(true) {
            val latestNews = newsApi.fetchLatestNews()
            emit(latestNews) // 向flow中发送请求的结果  
            delay(refreshIntervalMs)  //挂起协程一段时间 
        }
    }
}

// 一个接口，使用suspend函数来生成网络请求
interface NewsApi {
    suspend fun fetchLatestNews(): List<ArticleHeadline>
}
```

#### 限制
flow构建器运行在一个协程中，可以得益于相同的异步API，但是会有一些限制
* flow是顺序式的：因为生产者运行在一个协程中，当其调用一个suspend函数时，生产者会被挂起直到suspend函数返回
    * 上面的例子中，生产者会被挂起直到fetchLastNews的网络请求完成，只有这个时候只才会被发送到流中
* 使用flow构造器，生产者无法从不同的CoroutineContext中发送数据
    * 所以在一个由创建新协程或使用withContext而产生的不同的CoroutineContext中不要调用emit函数
    * 这种情况下，你可以使用其他的构建器，像callbackFlow


### 修改流
* 中间者可以使用中间者操作来在不消费数据的情况下修改数据流
* 这些操作符就是一些可以应用在数据流上的函数
* 操作符可以创建出一个操作链，只有数据流中的数据被消费的时候这些操作才会被执行
* [操作符文档](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-flow/)


#### 示例
```kotlin
class NewsRepository(
    private val newsRemoteDataSource: NewsRemoteDataSource,
    private val userData: UserData
) {
    /**
     * 通过在flow上使用变换器来获取热门的最新新闻
     * 这些操作符都是懒启动的，不会触发flow。它们只会变换当前发送的值
     */
    val favoriteLatestNews: Flow<List<ArticleHeadline>> =
        newsRemoteDataSource.latestNews
            // 使用中间者操作符来过滤热门的主题
            .map { news -> news.filter { userData.isFavoriteTopic(it) } }
            // 使用中间者操作符来保存新闻到缓存中
            .onEach { news -> saveInCache(news) }
}
```

#### 操作符链
* 中间者操作可以一个接着一个的被应用
* 组成的操作符链是懒执行的，只有当数据被送入flow中时才会执行
* 注意：只是简单地将一个操作符应用给一个流是不会启动flow的


### 从flow中搜集数据
#### 概述
* 使用终端操作符可以触发flow开始监听值
* [操作符文档](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-flow/)


#### collect操作符
* 使用collect可以获取到发送给流的所有数据
* collect是一个suspend函数，所以需要在协程中执行
* 接收一个lambda作为参数，该参数会对每个值都调用一次
* 因为是suspend函数，调用该函数的协程会被挂起直到flow被关闭

#### 示例
```kotlin
class LatestNewsViewModel(
    private val newsRepository: NewsRepository
) : ViewModel() {

    init {
        viewModelScope.launch {
            // 触发flow, 使用collect消费数据
            newsRepository.favoriteLatestNews.collect { favoriteNews ->
                // 使用最新的热门新闻更新视图
            }
        }
    }
}
```
* 收集flow会触发生产者刷新最新的新闻，并按指定间隔发送网络请求的结果
* 因为生产者使用while(true)循环，只有当ViewModel被清除或者viewModelScope被取消时数据流才会被关闭


#### flow收集停止的原因
* 运行收集的协程被取消，会同时停止下面的生产者
* 生产者结束发送数据，数据流会被关闭，调用收集的协程会恢复运行


#### 其他
* flow是冷的和懒的，除非指定了其他中间操作符
    * 意味着flow上每次终止操作符被调用，生产者代码就会被执行一次




### 捕获未知异常
* 生产者会使用第三方库来实现，意味着会抛出未知的异常
    * 可以使用catch中间操作符来处理这些异常

#### 示例1
```kotlin
class LatestNewsViewModel(
    private val newsRepository: NewsRepository
) : ViewModel() {

    init {
        viewModelScope.launch {
            newsRepository.favoriteLatestNews
                // catch中间操作符，如果有异常抛出，会捕获住并更新界面
                .catch { exception -> notifyError(exception) }
                .collect { favoriteNews ->
                    // 使用最新热门新闻更新界面
                }
        }
    }
}
```
* 当发生异常时，collect的闭包不会被调用



#### 示例2
```kotlin
class NewsRepository(...) {
    val favoriteLatestNews: Flow<List<ArticleHeadline>> =
        newsRemoteDataSource.latestNews
            .map { news -> news.filter { userData.isFavoriteTopic(it) } }
            .onEach { news -> saveInCache(news) }
            // If an error happens, emit the last cached values
            // 如果一个错误发生，就发送最后一个缓冲的值
            .catch { exception -> emit(lastCachedNews()) }
}
```
* catch也可以发送数据到flow中
* 当发生异常时，catch的闭包会被调用，一个新的数据会被发送到流中



### 在不同CoroutineContext中执行
* 默认情况下，flow构建的生产者会运行在收集该flow的协程的CoroutineContext中
* 前面提到过，从一个不同的CoroutineContext中是无法发送值的
* 有些情况下，这个特征是不受欢迎的
    * 例如在贯穿本主题的例子中，repository层无法在viewModelScope使用的Dispatchers.Main上运行操作

#### flowOn操作符
* 使用flowOn中间操作符可以改变flow的CoroutineContext
* flowOn会改变flow上游的CoroutineContext，但不会影响下游的CoroutineContext
    * 上游：生产者和flowOn之前的所有中间操作符
    * 下游：flowOn之后的中间操作符连同消费者
* 如果使用了多个flowOn操作符，每一个都会改变当前位置的上游

#### 示例1
```kotlin
class NewsRepository(
    private val newsRemoteDataSource: NewsRemoteDataSource,
    private val userData: UserData,
    private val defaultDispatcher: CoroutineDispatcher
) {
    val favoriteLatestNews: Flow<List<ArticleHeadline>> =
        newsRemoteDataSource.latestNews
            .map { news -> // 在默认的分发器中执行
                news.filter { userData.isFavoriteTopic(it) }
            }
            .onEach { news -> // 在默认的分发器中执行 
                saveInCache(news)
            }
            // flowOn影响flow的上游
            .flowOn(defaultDispatcher)
            // 不会影响flow的下游
            .catch { exception -> // 在消费者上下文中执行 
                emit(lastCachedNews())
            }
}
```
* onEach, map操作符使用defaultDispatcher
* catch操作符、消费者执行在viewModelScope使用的Dispatchers.Main中


#### 示例2
```kotlin
class NewsRemoteDataSource(
    ...,
    private val ioDispatcher: CoroutineDispatcher
) {
    val latestNews: Flow<List<ArticleHeadline>> = flow {
        // 在IO分发器中执行
        ...
    }
        .flowOn(ioDispatcher)
}
```
* 因为数据源需要执行IO操作，所以需要选择一个为IO操作优化过的分发器



### Jetpack库
* flow已经被整合进了许多jetpack库中
* flow非常适合于LiveData的更新和无尽的数据流


#### Room
* 你可以在Room中使用flow来接收数据库的变更通知
* 但你使用DAO时，会返回一个flow用于获取实时更新

##### 示例
```kotlin
@Dao
abstract class ExampleDao {
    @Query("SELECT * FROM Example")
    abstract fun getExamples(): Flow<List<Example>>
}
```
* 每次Example表有变动，一个新的带有新数据项的列表会被发送给流



### 基于回调的API转换成flow
* callbackFlow是一个用于将基于回调的API转换成flow的构建器

#### 示例
```kotlin
class FirestoreUserEventsDataSource(
    private val firestore: FirebaseFirestore
) {
    // 该方法从Firestore数据库中获取用户事件
    fun getUserEvents(): Flow<UserEvents> = callbackFlow {

        // 用于引用事件
        var eventsCollection: CollectionReference? = null
        try {
            eventsCollection = FirebaseFirestore.getInstance()
                .collection("collection")
                .document("app")
        } catch (e: Throwable) {
            // 如果firebase没有初始化，关闭数据流
            // flow的消费者就会停止收集数据，协程就会恢复运行
            close(e)
        }

        // 向firestore注册回调函数，在有新事件的时候会被调用
        val subscription = eventsCollection?.addSnapshotListener { snapshot, _ ->
            if (snapshot == null) { return@addSnapshotListener }
            // 向flow中发送事件，消费者就会拿到新的事件
            try {
                offer(snapshot.getEvents())
            } catch (e: Throwable) {
                // 事件不会发送给flow
            }
        }

        // 当flow被关闭或取消时，awaitClose中的回调就会执行
        // 这个例子中，就是从Firestore中移除回调
        awaitClose { subscription?.remove() }
    }
}
```

#### 特点
* callbackFlow允许从一个不同的CoroutineContext中发送数据
    * 使用send函数或在协程外部使用offer函数

#### channel
* 内部callbackFlow使用一个channel，从概念上说channel非常像一个阻塞的队列
* channel可以配置一个容量，即可以缓存的元素的最大个数
    * 默认容量为64个
* 当向满的channel中加入新元素时
    * send函数会挂起生产者，直到有空闲空间出现
    * offet函数会直接返回false，而不会加入元素





## 测试flow

### 概述
* 对于使用flow的测试使用哪种测试方法，取决于测试的对象是输入还是输出
* 如果测试对象是监听flow的，你可以使用fake依赖来生成flow
* 如果测试对象是导出一个flow的，你可以读一个值、校验一个值


### 创建一个fake生产者
* 当测试对象是一个flow的消费者时，一个通用的测试方法就是使用一个fake实现来替换生产者
* 例子：一个类监听一个从两个数据源获取数据的repository
    * 图示![](https://gitee.com/cc12703/figurebed/raw/master/img/20210518001342.png)
* 为了使测试具有确定性，可以替换repository使其依赖于一个fake repository，可以发送相同的fake数据
    * 图示 ![](https://gitee.com/cc12703/figurebed/raw/master/img/20210518001652.png)


#### 步骤
1. 使用flow构建器，在flow中发送预定义的值
    ```kotlin
    class MyFakeRepository : MyRepository {
        fun observeCount() = flow {
            emit(ITEM_1)
        }
    }
    ```
1. 测试时，将fake repository注入替换真实的实现
    ```kotlin
    @Test
    fun myTest() {
        // 使用fake依赖创建一个类型
        val sut = MyUnitUnderTest(MyFakeRepository())
        // 触发并校验
        ...
    }
    ```


### 断言flow的发送
* 当测试对象导出一个flow时，测试需要断言数据流中的元素是否正确
* 取决于测试的需求，你可以检查首个发射值或者最后几个来自flow的值

#### 示例1
```kotlin
@Test
fun myRepositoryTest() = runBlocking {
    // 创建一个从两个数据源合并值的repository
    val repository = MyRepository(fakeSource1, fakeSource2)

    // 当repository发送一个值时
    val firstItem = repository.counter.first() // 返回flow中的首个值 

    // 使用AssertJ来检查该值是否是期待的值
    assertThat(firstItem, isEqualTo(ITEM_1) 
}
```
* 通过调用first()可以获取flow中的首个值

#### 示例2
```kotlin
@Test
fun myRepositoryTest() = runBlocking {
    // 创建一个使用fake数据源的repository，会发送ALL_MESSAGES
    val messages = repository.observeChatMessages().toList()

    assertThat(messages, isEqualTo(ALL_MESSAGES))
}
```
* 如果需要检查多个值，可以使用toList()
* 该函数会使flow等待源发送完全部的值，返回一个列表
    * 该函数只能用于有效的数据流


#### 示例3
```kotlin
// 获取第二个数据项
outputFlow.drop(1).first()

// 获取开始的5个数据项
outputFlow.take(5).toList()

// 获取开始的5个不重复的数据项
outputFlow.take(5).toSet()

// 获取满足条件最先的2个的数据项
outputFlow.takeWhile(predicate).take(2).toList()

// 获取满足条件的首个数据项
outputFlow.firstWhile(predicate)

// 对每个值应用一个变化，并获取5个数据项
outputFlow.map(transformation).take(5)

// Takes the first item verifying that the flow is closed after that
// 获取首个值，来验证该flow是否已经关闭
outputFlow.single()

// 有限的数据流
// 校验该flow是否发送了特定的N个元素
outputFlow.count()
outputFlow.count(predicate)
```


### 将CoroutineDispatcher作为依赖
* 如果测试对象需要将CoroutineDispatcher作为一个依赖
    * 可以创建一个TestCoroutineDispatcher实例
    * 并使用runBlockingTeset来执行测试代码
* 示例
    ```kotlin
    private val coroutineDispatcher = TestCoroutineDispatcher()
    private val uut = MyUnitUnderTest(coroutineDispatcher)

    @Test
    fun myTest() = coroutineDispatcher.runBlockingTest {
        // 测试代码
    }
    ```






## StateFlow和ShareFlow

### 概述
* StateFlow和SharedFlow是一些flow API，对于发送状态变更和多个消费者进行了优化


### StateFlow
* StateFlow是一个状态保持者的可观察flow
    * 发送当前状态和新的状态给收集者
    * 当前的状态值可以通过value属性读取
* 为了更新状态和将其发送到flow中，需要给MutableStateFlow中的value属性赋值一个新值
* android中，StateFlow最适合用于管理一个可观测的可变状态


#### 示例-生产者
```kotlin
class LatestNewsViewModel(
    private val newsRepository: NewsRepository
) : ViewModel() {

    // 将属性隐藏起来，防止其他类型更新状态
    private val _uiState = MutableStateFlow(LatestNewsUiState.Success(emptyList()))
    // 界面从该StateFlow中进行收集，来获取到状态更新
    val uiState: StateFlow<LatestNewsUiState> = _uiState

    init {
        viewModelScope.launch {
            newsRepository.favoriteLatestNews
                // 使用最新的收藏新闻更新界面
                // 将该值写入到MutableStateFlow的value属性上
                // 将新元素加入到flow中，更新相关的收集者
                .collect { favoriteNews ->
                    _uiState.value = LatestNewsUiState.Success(favoriteNews)
                }
        }
    }
}

// 展现最新新闻界面的不同状态
sealed class LatestNewsUiState {
    data class Success(news: List<ArticleHeadline>): LatestNewsUiState()
    data class Error(exception: Throwable): LatestNewsUiState()
}
```
* StateFlow从LatestNewsViewModel中导出，视图可以监听界面状态的更新
* 用于更新MutableStateFlow的类是生产者
* 从StateFlow搜集数据的是消费者
* 不同于使用flow构造器生成的冷flow，StateFlow是热的
    * 从flow中进行收集不会触发任何生产者代码的运行
* 一个StateFlow一直都是在激活状态的，存在在内存中，只有没有GC根引用的时候才会被GC掉


#### 示例-消费者
```kotlin
class LatestNewsActivity : AppCompatActivity() {
    private val latestNewsViewModel = // getViewModel()

    override fun onCreate(savedInstanceState: Bundle?) {
        ...
        // 该协程只会在生命周期至少是Started状态时才会执行给定的代码块
        //当视图进入Stopped状态时就会被挂起
        lifecycleScope.launchWhenStarted {
            // 触发flow, 开始监听值
            latestNewsViewModel.uiState.collect { uiState ->
                // 接收到新值
                when (uiState) {
                    is LatestNewsUiState.Success -> showFavoriteNews(uiState.news)
                    is LatestNewsUiState.Error -> showError(uiState.exception)
                }
            }
        }
    }
}
```
* 当新的消费者开始从flow收集数据时，它会接收到流中的最新状态和其他的子状态
* 这个行为和LiveData类比较像


##### 注意
* 在界面层使用launchWhen()去收集一个flow也不是一直都安全的
    * 当界面进入后台，协程会被挂起，会使生产者保持在激活状态，潜在的发送一些视图无法消费的值
    * 这种行为会浪费CPU和内存资源
* 在ViewModel作用域中，使用launchWhen()去收集StateFlow是安全的
    当视图进入后台时，StateFlow会将值都保存在内存中，只会通知视图关于视图的状态

##### 警告
* 如果界面需要被更新，决不要使用launch()或launchIn()从界面来来收集一个flow
    * 当视图不可见时，这些函数也会处理事件
    * 这些行为会导致应用崩溃


#### 总结
* 使用stateIn中间操作符，可以将任何flow转换为StateFlow




### StateFlow、Flow和LiveData
* StateFlow和LiveData很相似
    * 都是可观测数据的保存类
    * 在使用时都遵循相同的模式

#### StateFlow和LiveData的不同点
* StateFlow生成时需要传入一个初始状态，LiveData不需要
* LiveData.observe()在视图进入STOPPED状态时，会自动注销消费者。但是StateFlow不会


#### 示例1
手动停止flow的收集
```kotlin
class LatestNewsActivity : AppCompatActivity() {
    ...
    // 协程监听界面状态
    private var uiStateJob: Job? = null

    override fun onStart() {
        super.onStart()
        // 当界面可见时，启动收集
        uiStateJob = lifecycleScope.launch {
            latestNewsViewModel.uiState.collect { uiState -> ... }
        }
    }

    override fun onStop() {
        // 当视图进入后台时，停止收集
        uiStateJob?.cancel()
        super.onStop()
    }
}
```

#### 示例2
将flow转化成LiveData
```kotlin
class LatestNewsActivity : AppCompatActivity() {
    ...
    override fun onCreate(savedInstanceState: Bundle?) {
        ...
        latestNewsViewModel.uiState.asLiveData().observe(owner = this) { state ->
            // 处理界面状态
        }
    }
}
```



### 使用shareIn使flow变热
* StateFlows是一种热flow
    * flow被收集后就一直存在在内存中
    * 任何到flow的引用都来自GC根
* 可以使用shareIn操作来将冷flow转换成热的


#### 用途
* 以Kotlin flow中的callbackFlow为例，通过使用shareIn在Firestore和每个收集者之间共享检索到的数据，这样不用给每个收集者创建一个flow

#### shareIn参数
* 一个用于共享flow的CoroutineScope，该scope的存活期需要比任何共享该flow的消费者更长
* 给每个收集者重发送数据项的个数
* 启动策略


#### 示例
```kotlin
class NewsRemoteDataSource(...,
    private val externalScope: CoroutineScope,
) {
    val latestNews: Flow<List<ArticleHeadline>> = flow {
        ...
    }.shareIn(
        externalScope,
        replay = 1,
        started = SharingStarted.WhileSubscribed()
    )
}
```
* lastestNews流会重发送最后一个数据项到新的收集者
* 保持激活状态会和externalScope的存活期一样久

#### 启动策略
* SharingStarted.WhileSubscribed：当有激活的订阅者时，保持上游生产者在激活状态
* SharingStarted.Eagerly：当有订阅者时，立刻启动生产者
* SharingStarted.Lazily：



### SharedFlow
* shareIn函数返回一个SharedFlow
    * 一个热flow，可以发送给所有收集该flow的消费者
    * 一个SharedFlow就是一个高可配置的，泛化的StateFlow
* 可以不使用shareIn()来创建一个SharedFlow


#### 示例
```kotlin
// 该类用于刷新应用的内容
class TickHandler(
    private val externalScope: CoroutineScope,
    private val tickIntervalMs: Long = 5000
) {
    private val _tickFlow = MutableSharedFlow<Unit>(replay = 0)
    val tickFlow: SharedFlow<Event<String>> = _tickFlow

    init {
        externalScope.launch {
            while(true) {
                _tickFlow.emit(Unit)
                delay(tickIntervalMs)
            }
        }
    }
}

class NewsRepository(
    ...,
    private val tickHandler: TickHandler,
    private val externalScope: CoroutineScope
) {
    init {
        externalScope.launch {
            // 监听tick的更新
            tickHandler.tickFlow.collect {
                refreshLatestNews()
            }
        }
    }

    suspend fun refreshLatestNews() { ... }
    ...
}
```
* 使用SharedFlow给应用的其他部分发送ticks，保证所有的内容都在同时进行周期刷新
* 除了获取最新新闻，还可以使用收藏的主题来刷新用户信息


#### 自定义行为
* replay：设置给新订阅者发送先前发送值的重发送个数
* onBufferOverflow：用于设置缓冲区满后的策略
    * BufferOverflow.SUSPEND：会挂起调用者
    * BufferOverflow.LATEST：保留最新的数据
    * BufferOverflow.DROP_OLDEST: 丢弃最老的数据