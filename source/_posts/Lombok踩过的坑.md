---
title: Lombok踩过的坑
tags: [Java, Lombok]
categories: 日常踩坑
abbrlink: 23605
date: 2020-11-26 22:29:04
---
Lombok是一个比较流行的Java类库，可以减少大量的重复代码或者模版代码，提高生产力，也使代码看起来更简洁。但如果不清楚实现机制，可能会出现意想不到的bug，最近在项目上遇到几个这样的坑。
<!-- more -->

## @Builder

@Builder注解提供了很方便的通过Builder模式构建一个对象的功能. 比如下面的例子:

```java
@Builder
public class User {
    private UUID id;
    private String name;
}
```

当我们构造一个User对象时，我们可以这样写:

```java
User build = User.builder()
                .id("12")
                .name("testName")
                .build();
```

### 变量初始化问题

对于某些字段会有初始值是一个固定值或可以自动生成的场景，比如我们希望每次build一个新的User对象时给id生成一个初始值，直观上说我们希望通过下面的代码来实现：

```java
@Builder
public class User {
    private UUID id = UUID.randomUUID();
    private String name;
}
```

理想的情况上面代码中的`private UUID id = UUID.randomUUID();`会起到生成初始值的效果，但其实并没有, 如下是编译后生成的代码, 可以看到生成的UserBuilder类的id变量并没有初始值，而在执行build方法时会new一个User对象，这是如果没有显式的指定id，则build出来的User对象的id就会是null。

```java 
public class User {
    private UUID id = UUID.randomUUID();
    private String name;

    User(final UUID id, final String name) {
        this.id = id;
        this.name = name;
    }

    public static User.UserBuilder builder() {
        return new User.UserBuilder();
    }

    public static class UserBuilder {
        private UUID id;
        private String name;

        UserBuilder() {
        }

        public User.UserBuilder id(final UUID id) {
            this.id = id;
            return this;
        }

        public User.UserBuilder name(final String name) {
            this.name = name;
            return this;
        }

        public User build() {
            return new User(this.id, this.name);
        }
    }
```

好在如果按上面的代码来写初始值的赋值，在编译时Lombok会产生一条警告，提示@Builder会忽略此初始值，建议使用@Builder.Default

> @Builder will ignore the initializing expression entirely. If you want the initializing expression to serve as default, add @Builder.Default

```java
@Builder
public class User {
    @Builder.Default
    private UUID id = UUID.randomUUID();
    private String name;
}
```

编译后生成的代码：

```java
public class User {
    private UUID id;
    private String name;

    private static UUID $default$id() {
        return UUID.randomUUID();
    }

    User(final UUID id, final String name) {
        this.id = id;
        this.name = name;
    }

    public static User.UserBuilder builder() {
        return new User.UserBuilder();
    }

    public String toString() {
        return "User(id=" + this.id + ", name=" + this.name + ")";
    }

    public static class UserBuilder {
        private boolean id$set;
        private UUID id;
        private String name;

        UserBuilder() {
        }

        public User.UserBuilder id(final UUID id) {
            this.id = id;
            this.id$set = true;
            return this;
        }

        public User.UserBuilder name(final String name) {
            this.name = name;
            return this;
        }

        public User build() {
            UUID id = this.id;
            if (!this.id$set) {
                id = User.$default$id();
            }

            return new User(id, this.name);
        }
    }
```

为id增加了`@Builder.Default`注解后自然初始值是可以被赋值的，可以看到Lombok为User类生成了一个`$default$id`方法，将初始值通过这个方法返回，而build User对象时，会检查id是否被赋值，如果未被赋值则会通过`$default$id`取id的初始值。

### @Builder与其他注解的使用

构建者模式通常用于构造一个复杂对象，不暴露内部状态，而如果自己来实现的话，正常来讲不会提供构造函数，但是当使用Lombok时，添加构造函数或Get/Set方法变得并不是那么明显，需要的时候加一个注解就搞定来。这样可能会导致注解的泛滥，如下面的例子。Lombok官方也不推荐这样的用法，对@Builder注解的说明是不建议与构造函数注解和`@EqualsAndHashCode`一起使用的。

```java
@ToString
@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
@EqualsAndHashCode
public class User implements Serializable {
    @Builder.Default
    private UUID id = UUID.randomUUID();
    private String name;
}
```

### toBuilder浅拷贝

使用`@Builder(toBuilder = true)`可以获取Builder，并多次调用build来构造对象，但需要注意的是这种方式构造的对象只能保证对象的浅拷贝，深层的对象还是同样的引用。

## @NonNull

@NonNull可以使用在实例变量/参数/方法上以做非null检查，这个注解并不是一个注释，而是会在所有使用的地方切实的注入null检查，如果检测结果为null则抛出NullPointerException. 反而显式的null-check是更好的选择。

```java
@Data
public class User implements Serializable {
    private UUID id = UUID.randomUUID();
    @NonNull
    private String name;
}
```

编译后生成的代码：

```java
public class User implements Serializable {
    private UUID id = UUID.randomUUID();
    @NonNull
    private String name;

    public User(@NonNull final String name) {
        if (name == null) {
            throw new NullPointerException("name is marked non-null but is null");
        } else {
            this.name = name;
        }
    }

    public UUID getId() {
        return this.id;
    }

    @NonNull
    public String getName() {
        return this.name;
    }

    public void setName(@NonNull final String name) {
        if (name == null) {
            throw new NullPointerException("name is marked non-null but is null");
        } else {
            this.name = name;
        }
    }
}
```

## 总结

- 使用`@Builder.Default`初始化变量
- @Builder要注意与其他注解的使用，特别是构造函数和`@EqualsAndHashCode`
- 使用`@Builder(toBuilder = true)` 只能实现浅拷贝
- @NonNull，如果检测结果为null则抛出NullPointerException. 反而显式的null-check是更好的选择。
