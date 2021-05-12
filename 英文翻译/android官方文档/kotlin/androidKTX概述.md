

# androidKTX概述

* https://developer.android.com/kotlin/ktx

[TOC]



## 概述
* android KTX是一个Kotlin扩展函数集，被用于Jetpack和其他android库中
* KTX扩展提供了简洁的、符合Kotlin使用习惯的Jetpack、android平台、其他库的API
* 使用了以下的Kotlin特性
    * 扩展函数
    * 扩展属性
    * Lambda
    * 命名参数
    * 参数默认值
    * 协程


### 举例
* 当使用SharePreferences时，在修改数据前需要创建一个编辑器对象
    * 当完成编辑时，需要apply或者commit这些变动
* 示例
    ```kotlin
    sharedPreferences
            .edit()  // 创建一个编辑器
            .putBoolean("key", value)
            .apply() // 异步将数据写入磁盘
    ```
* Kotlin的lambda非常适合这种场景
    * 在编辑器被创建后，它允许你使用更简洁的方法将代码块传入执行
    * 它让代码执行，并让SharePreferences原子的应用这些变动


#### SharePreferences.edit
* SharePreferences.edit() 是KTX的核心函数之一
    * 该函数接收一个可选的布尔类型作为首个参数，来表明是commit还是apply变动
    * 该函数以lambda的形式接收一个需要执行的动作
* 示例代码
    ```kotlin
    // SharedPreferences.edit 一个扩展函数
    // inline fun SharedPreferences.edit(
    //         commit: Boolean = false,
    //         action: SharedPreferences.Editor.() -> Unit)

    // 异步提交一个新值
    sharedPreferences.edit { putBoolean("key", value) }

    // 同步提交一个新值
    sharedPreferences.edit(commit = true) { putBoolean("key", value) }
    ```
* 调用者可以选择是commit还是apply变动
* action lambda作为SharedPreferences.Editor的一个匿名扩展函数，就像函数签名中的那样返回值为Unit
    * 这就是为什么在代码块中，你可以直接调用SharedPreferences.Editor的功能
* SharedPreferences.edit()签名中包含一个inline关键字
    * 该关键字用于告诉Kotlin编译器，每次函数被使用时，需要拷贝粘贴该函数编译后的字节码
    * 这避免了每次函数被调用时，都要实例化一个action类的开销


### 总结
* KTX库提供的用于增强的手段包括
    * 使用lambda传递代码
    * 使用可以被覆盖的默认值
    * 使用inline扩展函数给已存在API增加功能




## 在工程中使用KTX

### 加入依赖
build.gradle文件
```groovy
repositories {
    google()
}
```


## AndroidX模块

### 概述
* KTX被组织成多个模块，每个模块包含一个、多个包
* 你必须在build.gradle中给每个模块制品加入对应的依赖信息
    * 记住要给每个制品追加版本号
* KTX包含一个单独的核心模块，用于给框架API提供扩展、许多领域相关的扩展
* 除了核心模块，所有的KTX模块制品都对应一个Java的依赖库
    * 替换规则：将androidx.fragment:fragment替换成androidx.fragment:fragment-ktx


### 核心KTX
* 核心KTX模块提供了anroid框架中公共库的扩展
* 这些库没有对应的Java依赖
* build.gradle示例
    ```groovy
    dependencies {
        implementation "androidx.core:core-ktx:1.3.2"
    }
    ```
* 模块中包含的包：androidx.core.xxxx


### 集合KTX
* 集合扩展包含了一些用于android内存优化集合库的辅助函数
    * 集合包括：ArrayMap, LongSpareArray, LruCache
* build.gradle示例
    ```groovy
    dependencies {
        implementation "androidx.collection:collection-ktx:1.1.0"
    }
    ```
* 集合扩展使用了Kotlin重载操作的优势，简化了像集合连接之类的操作
    ```kotlin
    // Combine 2 ArraySets into 1.
    val combinedArraySet = arraySetOf(1, 2, 3) + arraySetOf(4, 5, 6)

    // Combine with numbers to create a new sets.
    val newArraySet = combinedArraySet + 7 + 8
    ```


### Fragment KTX
* 该模块提供了一些简化fragment接口的扩展
* build.gradle示例
    ```groovy
    dependencies {
        implementation "androidx.fragment:fragment-ktx:1.3.3"
    }
    ``` 

#### 示例
* 使用lambda简化fragment的事务处理
    ```kotlin
    fragmentManager().commit {
        addToBackStack("...")
        setCustomAnimations(
                R.anim.enter_anim,
                R.anim.exit_anim)
        add(fragment, "...")
    }
    ```
* 使用一行来绑定ViewModel
    ```kotlin
    // 获取一个作用范围在该Fragment中的ViewModel
    val viewModel by viewModels<MyViewModel>()

    // 获取一个作用范围在Activity的ViewModel
    val viewModel by activityViewModels<MyViewModel>()
    ```


### Lifecycle KTX
* 该模块给每个Lifecycle对象定义了一个LifecycleScope作用域
    * 在该作用域中启动的任何协程，当对应的Lifecycle被销毁后会被自动取消运行
* 通过lifecycle.coroutineScope或者lifecycleOwner.lifecycleScope属性可以访问Lifecycle的CoroutineScope
* build.gradle示例
    ```groovy
    dependencies {
        implementation "androidx.lifecycle:lifecycle-runtime-ktx:2.3.1"
    }
    ``` 
* 示例：异步创建一个预计算的文本
    ```kotlin
    class MyFragment: Fragment() {
        override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
            super.onViewCreated(view, savedInstanceState)
            viewLifecycleOwner.lifecycleScope.launch {
                val params = TextViewCompat.getTextMetricsParams(textView)
                val precomputedText = withContext(Dispatchers.Default) {
                    PrecomputedTextCompat.create(longTextContent, params)
                }
                TextViewCompat.setPrecomputedText(textView, precomputedText)
            }
        }
    }
    ```



### LiveData KTX
* 当使用LiveData时，我们会需要异步的计算一些值
    * 例子：获取一个用户的偏好，显示在界面上
* 该模块提供了一个liveData构建器，内部可以调用一个suspend函数，将结果作为一个LiveData对象
* build.gradle示例
    ```groovy
    dependencies {
        implementation "androidx.lifecycle:lifecycle-livedata-ktx:2.3.1"
    } 
    ```
* 示例：在liveData中异步调用loadUser()函数，并使用emit()来发送数据
    ```kotlin
    val user: LiveData<User> = liveData {
        val data = database.loadUser() // loadUser 是一个 suspend 函数.
        emit(data)
    }
    ```


### Navigation KTX
* 导航库的每个组件都有自己的KTX版本，可以使API更简洁和Kotlin惯用化
* build.gradle示例
    ```groovy
    dependencies {
        implementation "androidx.navigation:navigation-runtime-ktx:2.3.5"
        implementation "androidx.navigation:navigation-fragment-ktx:2.3.5"
        implementation "androidx.navigation:navigation-ui-ktx:2.3.5"
    } 
    ``` 
* 示例：
    ```kotlin
    class MyDestination : Fragment() {
        // 从bundle中获取类型安全的参数
        val args by navArgs<MyDestinationArgs>()

        ...
        override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
            view.findViewById<Button>(R.id.next)
                .setOnClickListener {
                    // 使用Fragment扩展来从目的地获取一个NavController对象
                    findNavController().navigate(R.id.action_to_next_destination)
                }
        }
        ...
    }
    ```


### Palette KTX
* 该模块给颜色调色板提供了Kotlin化的接口
* build.gradle示例
    ```groovy
    dependencies {
        implementation "androidx.palette:palette-ktx:1.0.0"
    } 
    ```
* 示例：
    ```kotlin
    val palette = Palette.from(bitmap).generate()
    val swatch = palette[target]
    ```


### 反应式流 KTX
* 该模块允许你从ReactiveStreams发布器中创建一个可观察的LiveData流
* build.gradle示例
    ```groovy
    dependencies {
        implementation "androidx.lifecycle:lifecycle-reactivestreams-ktx:2.3.1"
    }
    ```


#### 举例
* 设想一个应用，数据库中保存着少量的用户信息
* 你需要将其加载到内存中显示显示到界面上
* 你使用了RxJava和Room库来实现上述功能
* 这时候，你必须要跨fragment或activity的生命周期来管理Rx发布者的订阅
* 使用LiveDataReactiveStream，你可以获取到RxJava的优势（丰富的操作符和工作调用能力）并使用LiveData的简洁性
    ```kotlin
    val fun getUsersLiveData() : LiveData<List<User>> {
        val users: Flowable<List<User>> = dao.findUsers()
        return LiveDataReactiveStreams.fromPublisher(users)
    }
    ```

### Room KTX
* 该模块给数据库事务增加了协程支持
* build.gradle示例
    ```groovy
    dependencies {
        implementation "androidx.room:room-ktx:2.3.0"
    }
    ```
* 示例：
    ```kotlin
    @Query("SELECT * FROM Users")
    suspend fun getUsers(): List<User>

    @Query("SELECT * FROM Users")
    fun getUsers(): Flow<List<User>>
    ```


### SQLite KTX
* 该模块支持在事务中包含SQL相关代码，排除大量歧义代码
* build.gradle示例
    ```groovy
    dependencies {
        implementation "androidx.sqlite:sqlite-ktx:2.1.0"
    }
    ```
* 示例：使用transaction扩展来执行数据库事务
    ```kotlin
    db.transaction {
        // 插入数据
    }
    ```


### ViewModel KTX
* 该模块提供了viewModelScope()函数，来简化在ViewModel中启动协程
* 该协程作用域绑定到了Dispatchers.Main中，当ViewModel清除时协程会自动取消运行
* 你可以使用viewModelScope()来给每个ViewModel创建一个新的作用域
* build.gradle示例
    ```groovy
    dependencies {
        implementation "androidx.lifecycle:lifecycle-viewmodel-ktx:2.3.1"
    }
    ```
* 示例：使用协程在后台线程中进行一次网络请求，该模块会自动处理作用域的创建和清理
    ```kotlin
    class MainViewModel : ViewModel() {
        // 创建一个不会阻塞UI线程的网络请求
        private fun makeNetworkRequest() {
            // 在ViewModelScope中启动一个协程
            viewModelScope.launch  {
                remoteApi.slowFetch()
                ...
            }
        }

        // 不需要覆写onCleared()函数
    }
    ```


### WorkManager KTX
* 该模块提供了协程支持
* build.gradle示例
    ```groovy
    dependencies {
        implementation "androidx.work:work-runtime-ktx:2.5.0"
    }
    ```
* 使用该库后，你需要扩展CoroutineWorker，而不是Worker
* 示例：
    ```kotlin
    class CoroutineDownloadWorker(context: Context, params: WorkerParameters)
            : CoroutineWorker(context, params) {

        override suspend fun doWork(): Result = coroutineScope {
            val jobs = (0 until 100).map {
                async {
                    downloadSynchronously("https://www.google.com")
                }
            }

            // 如果有一个下载失败，awaitAll会抛出一个异常
            // CoroutineWorker会将其视为失败
            jobs.awaitAll()
            Result.success()
        }
    }
    ```
