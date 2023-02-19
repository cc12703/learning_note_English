

# Spring框架

* [原始文档](https://docs.spring.io/spring-framework/docs/current/reference/html/)



## 核心技术

### IoC容器

#### 简介
* IoC也被称为DI（依赖注入）
* 容器再创建bean时会注入其对应的依赖
* Spring框架中的IoC容器在包`org.springframework.beans`和`org.springframework.context`中
* `BeanFactory`提供了配置框架和基础功能
* `ApplicationContext`添加了更多业务相关的功能
* 在Spring中，Bean是那些构成应用核心的、由IoC容器进行实例化、装配、管理的对象


#### 容器综述
