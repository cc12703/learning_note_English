

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