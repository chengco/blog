---
title: Spring 从一个实例看依赖注入的方式
tags:
  - Java
  - Spring
categories: Java
abbrlink: 49430
date: 2022-03-09 22:24:28
---
Spring可通过三种方式实现依赖注入，基于构造函数的依赖注入，基于Setter方法的依赖注入，以及基于Field的依赖注入。

<!-- more -->

## 实例：抽象类的依赖注入

现有一个`Component` A， 通常采用构造函数的方式注入依赖 `dependencyBean`

```java
@Component
public class A {
    private DependencyBean dependencyBean;

    public A(DependencyBean dependencyBean) {
        this.dependencyBean = dependencyBean;
    }

    //some logic using dependencyBean
}
```

而当有另一个`Component`B：

```java
@Component
public class B {
    private final DependencyBean dependencyBean;

    public B(DependencyBean dependencyBean) {
        this.dependencyBean = dependencyBean;
    }

    //some logic using dependencyBean
}
```

A和B有着相同的依赖和相似的逻辑，这时通常会使用模版方法，将相同的逻辑抽象到共同的父类中，父类是不需要增加 `@Component` 注解声明。

```java
public abstract class AbstractBean {
    private DependencyBean dependencyBean;

    public AbstractBean(DependencyBean dependencyBean) {
        this.dependencyBean = dependencyBean;
    }

    //some logic using dependencyBean
}

@Component
public class A extends AbstractBean{

    public A(DependencyBean dependencyBean) {
        super(dependencyBean);
    }

}

@Component
public class B extends AbstractBean{
 
    public B(DependencyBean dependencyBean) {
        super(dependencyBean);
    }
}
```

上面的做法注入dependencyBean的逻辑还是维护在子类中，会造成代码冗余。使用Setter方法的依赖注入可减少代码冗余：

```java
public abstract class AbstractBean {
    private DependencyBean dependencyBean;

    @Autowired
    public void setDependencyBean(DependencyBean dependencyBean) {
        this.dependencyBean = dependencyBean;
    }

    //some logic using dependencyBean
}

@Component
public class A extends AbstractBean{
    //other logic
}

@Component
public class B extends AbstractBean{
    //other logic
}
```

使用构造函数的依赖注入和Setter方法的依赖注入有啥区别，下面看一下Spring Bean的依赖注入实现方式。

## Spring Bean的三种依赖注入实现方式

Spring可通过三种方式实现依赖注入，基于构造函数的依赖注入，基于Setter方法的依赖注入，以及基于Field的依赖注入。

### 基于构造函数的依赖注入

```java
@Component
public class DemoBean{
    public final DependencyBean dependencyBean;

    public DemoBean(DependencyBean dependencyBean) {
        this.dependencyBean = dependencyBean;
    }
}
```

基于构造函数注入的优点是，可以很方便地构造immutable的对象，将注入的字段声明为 final，在类实例化期间注入依赖。

### 基于Setter方法的依赖注入

```java
@Component
public class DemoBean{
    public DependencyBean dependencyBean;

    @Autowired
    public void setDependencyBean(DependencyBean dependencyBean) {
        this.dependencyBean = dependencyBean;
    }
}
```

基于Setter方法注入的优点是，可以在实例Bean构造完成后注入，或者重新注入Bean，其中一个应用场景是通过[JMX MBean](https://docs.spring.io/spring-framework/docs/current/reference/html/integration.html#jmx)进行管理。

### 基于Field的依赖注入

```
@Component
public class DemoBean{
    @Autowired
    private DependencyBean dependencyBean;
}
```

Spring Framework官方已经不推荐这种依赖注入方法，文档中也去掉了相关的使用说明，编译器也会给出警告 `Field injection is not recommended`，究其原因，使用Field依赖注入时，有一些弊端：

- 相比基于构造函数的依赖注入方式，基于Field的依赖注入无法构造immutable对象，因为如果将dependencyBean声明为final，则必须在构造函数中初始化。
- 对单元测试不友好，如果没有对外的set接口，则很难替换mock bean

### 如何选择？

Spring官方推荐使用基于构造函数的依赖注入方式，结合final，可以构建immutable的对象，能够保证所有的依赖都不会为null。而当需要在实例Bean构造完成后注入，或者重新注入Bean时，选择使用Setter方法的注入方式。

除此之外，当在Bean中使用模版方法，在Abstract类中注入依赖时，需要使用Setter方法注入。

### 参考

[https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-dependencies](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-dependencies)

[https://blog.marcnuri.com/field-injection-is-not-recommended](https://blog.marcnuri.com/field-injection-is-not-recommended)
