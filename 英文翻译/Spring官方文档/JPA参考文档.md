


# JPA参考文档

* [原始文档](https://docs.spring.io/spring-data/jpa/docs/2.5.1/reference/html/#reference)



## 仓库

## 核心概念
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


### 查询示例

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


### 定义仓库接口类
* 要定义仓库接口类，首先需要定义一个领域类相关的仓库接口类
* 该类必须要扩展于Repository类并输入领域类和ID类型
* 如果要导出CRUD方法，就需要扩展于CrudRepository

#### 微调仓库定义
* 