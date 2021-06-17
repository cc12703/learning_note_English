


# JPA参考文档

* [原始文档](https://docs.spring.io/spring-data/jpa/docs/2.5.1/reference/html/#reference)

[TOC]

## 4 数据仓库

## 4.1 核心概念
* Repository是SpringData仓库抽象层中的核心接口类
    * 该类接收一个领域类用于管理，领域类的ID类型作为类型参数
    * 该类的主要作用是作为一个标记接口来捕获要操作的类型、辅助发现扩展该类的接口


### CrudRepository   

#### 功能
* 该接口类为管理的实体类提供了复杂的CRUD功能

#### 示例 
```java
public interface CrudRepository<T, ID> extends Repository<T, ID> {

  <S extends T> S save(S entity);  // 保存传入的实体对象

  Optional<T> findById(ID primaryKey); // 返回指定ID的实体对象

  Iterable<T> findAll();          // 返回所有的实体对象           

  long count();                   // 返回实体数量      

  void delete(T entity);          // 删除传入的实体 

  boolean existsById(ID primaryKey);   // 检查指定ID的实体是否存在

  // … more functionality omitted.
}
```

#### 注意
* 我们也提供了持久化技术相关的抽象，像JpaRepository或MongoRepository
* 这些类扩展于CrudRepository，导出了基于相关持久化技术的能力


### PagingAndSortingRepository

#### 功能
* 该接口类基于CrudRepository增加了分页访问功能

#### 示例 
```java
public interface PagingAndSortingRepository<T, ID> extends CrudRepository<T, ID> {

  Iterable<T> findAll(Sort sort);

  Page<T> findAll(Pageable pageable);
}
```

#### 访问示例
* 访问User实体的第二页，每页20个对象
```java
PagingAndSortingRepository<User, Long> repository = // … get access to a bean
Page<User> users = repository.findAll(PageRequest.of(1, 20));
```


### 4.2 查询示例

#### 数量查询
```java
interface UserRepository extends CrudRepository<User, Long> {

  long countByLastname(String lastname);
}
```

#### 删除查询
```java
interface UserRepository extends CrudRepository<User, Long> {

  long deleteByLastname(String lastname);

  List<User> removeByLastname(String lastname);
}
```


### 查询方法
* 标准CRUD方法的仓库类经常会有一些基于底层的数据源查询功能
* Spring Data中使用4个步骤来定义这些查询

#### 步骤1
* 基于Repository类或其子类定义一个接口类，传入要处理的领域类和ID类型
* 示例
    ```java
    interface PersonRepository extends Repository<Person, Long> { … }
    ```

#### 步骤2
* 在接口类中定义查询方法
* 示例
    ```java
    interface PersonRepository extends Repository<Person, Long> {
        List<Person> findByLastname(String lastname);
    }
    ```

#### 步骤3
* 使用JavaConfig或XML让Spring为这些接口类创建代理实例
* Java配置示例
    ```java
    import org.springframework.data.jpa.repository.config.EnableJpaRepositories;

    @EnableJpaRepositories
    class Config { … }
    ```
* XML配置示例
    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:jpa="http://www.springframework.org/schema/data/jpa"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/data/jpa
        https://www.springframework.org/schema/data/jpa/spring-jpa.xsd">

    <jpa:repositories base-package="com.acme.repositories"/>

    </beans>
    ```


#### 步骤4
* 注入仓库实体并使用它
* 示例
    ```java
    class SomeClient {

    private final PersonRepository repository;

    SomeClient(PersonRepository repository) {
        this.repository = repository;
    }

    void doSomething() {
        List<Person> persons = repository.findByLastname("Matthews");
    }
    }
    ```


### 4.3 定义仓库接口类
* 要定义仓库接口类，首先需要定义一个领域类相关的仓库接口类
* 该类必须要扩展于Repository类并输入领域类和ID类型
* 如果要导出CRUD方法，就需要扩展于CrudRepository

#### 微调仓库定义
* 一般来说，你可以通过扩展`Repository`,`CrudRepository`,`PagingAndSortingRepository`来定义仓库接口
* 但是如果你不想扩展Spring Data接口，你也可以使用`@RepositoryDefinition`来标注仓库接口
* 扩展`CrudRepository`会导出一个完整的用于操作实体的方法集
    * 如果更喜欢选择一些方法导出，可以将要导出的方法从`CrudRepository`中拷贝到你的领域仓库接口中

##### 示例：选择导出CRUD方法
```java
@NoRepositoryBean
interface MyBaseRepository<T, ID> extends Repository<T, ID> {

  Optional<T> findById(ID id);

  <S extends T> S save(S entity);
}

interface UserRepository extends MyBaseRepository<User, Long> {
  User findByEmailAddress(EmailAddress emailAddress);
}
```
* 可以给全部的领域仓库类定义一个基础的公共接口类，来导出`findById(...)`和`save(...)`这类方法
* 这些方法会被路由到一个你从SpringData中选择的存储器的基本仓库实现类中（如果你使用JPA，实现类就是SimpleJpaRepository）
    * 因为这些方法会匹配到`CrudRepository`中的方法


#### 在多个SpringData模块下定义仓库类
* 在应用中使用一个SpringData模块会比较简单，因为所有定义的仓库接口类都会被绑定到SpringData模块上
* 有时候，应用会使用多个SpringData模块，这时候一个仓库类的定义必须可以区分不同的存储技术
* 当侦测到类路径中存储多个仓库工厂时，SpringData会进入严格配置模式
    * 严格配置模式会使用**仓库类的细节**或**领域类**来决定需要绑定到的SpringData模块

##### 算法
1. 如果定义的仓库类扩展于具体模块的仓库类，该特定的SpringData模块就是一个有效的候选对象
1. 如果领域类使用了具体模块的标注类进行了标注，该特定的SpringData模块就是一个有效的候选对象
    * SpringData接受第三方标注（像JPA的`@Entity`）或内建的标注（像SpringData MongoDB的`@Document`）

##### 示例：使用具体模块接口
```java
interface MyRepository extends JpaRepository<User, Long> { }

@NoRepositoryBean
interface MyBaseRepository<T, ID> extends JpaRepository<T, ID> { … }

interface UserRepository extends MyBaseRepository<User, Long> { … }
```
* MyRepository 和 UserRepository 都扩展于JpaRepository，所以SpringData JPA模块是有效的候选

##### 示例：使用通用接口
```java
interface AmbiguousRepository extends Repository<User, Long> { … }

@NoRepositoryBean
interface MyBaseRepository<T, ID> extends CrudRepository<T, ID> { … }

interface AmbiguousUserRepository extends MyBaseRepository<User, Long> { … }
```

##### 示例：使用带标注的领域类
```java
interface PersonRepository extends Repository<Person, Long> { … }

@Entity
class Person { … }

interface UserRepository extends Repository<User, Long> { … }

@Document
class User { … }
```
* PersonRepository引用了Persion类，该类标注了JPA的@Entity，所以该仓库属于SpringData JPA
* UserRepository引用了User类，该类标注了SpringData MongoDB的@Document

##### 示例：使用带混合标注的领域类
```java
interface JpaPersonRepository extends Repository<Person, Long> { … }

interface MongoDBPersonRepository extends Repository<Person, Long> { … }

@Entity
@Document
class Person { … }
```
* 由于Person类使用了JPA和MongoDB的标注，所以SpringData无法区分它们，会导致无法定义的行为

TODO


##### 区分方法：基于包名
* 最后一个区分仓库类的方法就是基于包名来界定仓库
* 基于包名定义扫描仓库类的起点，意味着仓库类的定义位于合适的包下
* 默认情况下，标注驱动配置会使用配置类的包

##### 示例：基于包的标注驱动配置
```java
@EnableJpaRepositories(basePackages = "com.acme.repositories.jpa")
@EnableMongoRepositories(basePackages = "com.acme.repositories.mongo")
class Configuration { … }
```


### 4.4 定义查询方法
仓库类代理有两种方法来从方法名中导出一个存储特定的查询语句
1. 从方法名字直接导出查询语句
1. 使用手动定义的查询语句


#### 查询搜索策略
* 下面的策略对于仓库解析查询语句是有效的
* 使用XML配置时，你可以通过`query-lookup-strategy`属性来配置策略
* 使用Java配置时，你可以使用`Enable${store}Repositories`标注的`queryLookupStrategy`属性

##### CREATE
* 尝试从查询方法名字中构造出存储特定的查询语句
* 大致方法：从名字中移除通用的前缀名，解析剩余的名字

##### USE_DECLARED_QUERY
* 尝试查找一个已定义的查询语句，如果找不到则抛出异常
* 该查询语句可以由一个标注来定义
* 如果仓库基础设施在启动时没有找到已定义的查询语句，启动就失败了

##### CREATE_IF_NOT_FOUND
* 该策略融合的前面两个策略
* 首先查询已定义的查询语句，如果煤矿有找到就从方法名字中构建一个出来
* 如果不配置策略的话，该策略是默认使用的策略


#### 创建查询
* 查询构建器算法内建于SpringData的仓库基础设施中，用于在仓库实体类上构建约束查询

##### 示例：从方法名中创建查询
```java
interface PersonRepository extends Repository<Person, Long> {

  List<Person> findByEmailAddressAndLastname(EmailAddress emailAddress, String lastname);

  // 为查询使能唯一标识功能
  List<Person> findDistinctPeopleByLastnameOrFirstname(String lastname, String firstname);
  List<Person> findPeopleDistinctByLastnameOrFirstname(String lastname, String firstname);

  // 为一个属性使能忽略大小功能
  List<Person> findByLastnameIgnoreCase(String lastname);
  // 为所有属性使能忽略大小写功能
  List<Person> findByLastnameAndFirstnameAllIgnoreCase(String lastname, String firstname);

  // 为查询使能静态的 ORDER BY
  List<Person> findByLastnameOrderByFirstnameAsc(String lastname);
  List<Person> findByLastnameOrderByFirstnameDesc(String lastname);
}
```

##### 解析方法名
* 方法名会被解析成为**对象**和**谓词**
* 第一个部分(`find..by`,`exists..by`)定义了查询的对象，第二部分构造了谓词
* 引入的子句（对象部分）可以包含表达式。在关键字`find`和`by`之间的文本都被认为是描述性的