---
title: 什么是POJO？
date: 2022-03-09 23:02:49
tags: [Java]
categories: Java
---
Plain Old Java Object， 普通的Java对象，封装业务逻辑，除Java语言规范外，没有任何其他的限制，不与任何框架绑定。
<!-- more -->

反例：

- 扩展特定的类，public class GFG extends javax.servlet.http.HttpServlet { … }
- 实现特定的接口，public class Bar implements javax.ejb.EntityBean { … }
- 包含特定的注解，@javax.persistence.Entity public class Baz { … }

正例：

```java
public class Employee {
    // default field
    String name;
    // public field
    public String id;
    // private salary
    private double salary;
  
    //arg-constructor to initialize fields
    public Employee(String name, String id, double salary) {
        this.name = name;
        this.id = id;
        this.salary = salary;
    }
  
    // getter method for name
    public String getName() {
        return name;
    }
  
    // getter method for id
    public String getId() {
        return id;
    }
  
    // getter method for salary
    public Double getSalary() {
        return salary;
    }
}
```

[https://martinfowler.com/bliki/POJO.html](https://martinfowler.com/bliki/POJO.html)

[https://www.geeksforgeeks.org/pojo-vs-java-beans/#:~:text=POJO stands for Plain Old,re-usability of a program](https://www.geeksforgeeks.org/pojo-vs-java-beans/#:~:text=POJO%20stands%20for%20Plain%20Old,re%2Dusability%20of%20a%20program).