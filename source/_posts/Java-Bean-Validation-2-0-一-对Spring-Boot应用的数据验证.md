---
title: 'Java Bean Validation 2.0 (一): 对Spring Boot应用的数据验证'
tags: Java Validation
categories: Java
abbrlink: 31379
date: 2020-11-26 22:58:16
---
## 写在前面

数据验证是一个应用开发中非常常见的需求，不管是表现层、模型层还是持久层都有类似的需求来保证数据的有效性。Eclipse基金会在2019年8月份发布了[Jakarta Bean Validation 2.0规范](https://beanvalidation.org/2.0)。这个规范定义了用于Java Bean验证的元数据模型和API，可以使用注解或XML做数据验证，也可以扩展元数据，定义自己的验证逻辑。仅支持Java 8以上的版本。
<!-- more -->

## 包依赖

对于Spring Boot应用来说，不需要特别配置，以gradle配置为例，当有以下配置时会自动支持验证功能。

>    compile 'org.springframework.boot:spring-boot-starter-web'

Java Bean验证是一个相对通用的规范，并不依赖某个框架，对任意Java应用，均可通过依赖`validation-api`和`hibernate-validator`引入此功能，`validation-api`是Bean Validation规范定义的API，`hibernate-validator`是对应的实现，也是事实标准。注意`hibernate-validator`与Hibernate持久化框架并无依赖关系，可单独依赖使用。

## 一个例子

以下代码为Validation Controller, 当Spring Boot发现参数注解为`@Valid`时, 会自动启动默认的Vilidation实现`Hibernate Validator`进行验证。如果验证失败则抛出`MethodArgumentNotValidException`异常。

```java
@RestController
public class TestController {

    @PostMapping(value = "/testValidation", consumes = MediaType.APPLICATION_JSON_VALUE)
    public void testValidation(@Valid @RequestBody TestResource resource){
        //noop
    }
}
```

Resoure实现，对name字段进行NotNull验证。

```java
@Data
public class TestResource {
    @NotNull(message = "name must not be null")
    private String name;
    private int age;
}

```

执行结果展示默认的验证结果：

```json
{
    "timestamp": "2020-09-28T05:41:23.558+0000",
    "status": 400,
    "error": "Bad Request",
    "errors": [
        {
            "codes": [
                "NotNull.testResource.name",
                "NotNull.name",
                "NotNull.java.lang.String",
                "NotNull"
            ],
            "arguments": [
                {
                    "codes": [
                        "testResource.name",
                        "name"
                    ],
                    "arguments": null,
                    "defaultMessage": "name",
                    "code": "name"
                }
            ],
            "defaultMessage": "name must not be null",
            "objectName": "testResource",
            "field": "name",
            "rejectedValue": null,
            "bindingFailure": false,
            "code": "NotNull"
        }
    ],
    "message": "Validation failed for object='testResource'. Error count: 1",
    "path": "/testValidation"
}
```


## ExceptionHandler：格式化验证结果输出

在本文开始的例子中，展示了验证结果的输出格式，对于仅有一个字段的验证，输出了大量的信息，但真正关注的可能仅是其中一小部分。对于Spring Boot应用来说，可以通过ExceptionHandler拦截`MethodArgumentNotValidException`异常，来实现验证结果的格式化输出。除错误信息外，错误码也可通过ExceptionHandler定制。

```java
@ControllerAdvice
public class ExceptionController {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<String> handleValidationException(MethodArgumentNotValidException e, WebRequest request) {
        BindingResult bindingResult = e.getBindingResult();
        String errorMessage = bindingResult.getFieldErrors()
                .stream()
                .map(error -> String.format("%s - %s", error.getField(), error.getDefaultMessage()))
                .collect(Collectors.joining(" "));
        return new ResponseEntity<>(errorMessage, HttpStatus.INTERNAL_SERVER_ERROR);
    }
}
```

输出结果为如下，简洁了不少。

> user.name - name must not be null

## Bean Validation 2.0规范内建注解

对于常见的约束校验，`Jakarta Bean Validation 2.0规范`提供了一些内建的注解来支持：

- @Null 值必须为null
- @NotNull 值不能为null
- @AssertTrue 值必须为true
- @AssertFalse 值必须为false
- @Min 值必须大于等于给定的数字，支持BigDecimal/BigInteger/byte/short/int/long类型
- @Max 值必须小于等于给定的数字，支持BigDecimal/BigInteger/byte/short/int/long类型
- @DecimalMin 值必须大于等于给定的数字（字符串类型），支持BigDecimal/BigInteger/CharSequence/byte/short/int/long类型
- @DecimalMax 值必须小于等于给定的数字（字符串类型），支持BigDecimal/BigInteger/CharSequence/byte/short/int/long类型
- @Negative 值必须为负数
- @NegativeOrZero 值必须为零或负数
- @Positive 值必须为正数
- @PositiveOrZero 值必须为零或正数
- @Size 指定元素个数范围，支持CharSequence/Collection/Map/Array
- @Digits 指定数字整数部分和小数部分最大长度
- @Past 值必须为过去的时间
- @PastOrPresent 值必须为现在或过去的时间
- @Future 值必须为将来的时间
- @FutureOrPresent 值必须为现在或将来的时间
- @Pattern 匹配正则表达式
- @NotEmpty 非空，支持CharSequence/Collection/Map/Array
- @NotBlank 必须不能为null且至少包含一个非空白字符
- @Email 必须为合法的email地址

## 使用@Valid做嵌套结构验证

对于嵌套结构的结构，需要使用`@Valid`注解传递验证声明，如下例，期望验证`UserResource`的name字段，因为存在嵌套结构，所以需要中user字段上添加`@Valid`注解声明。

```java
@Data
public class TestResource {
    @Valid
    private UserResource user; 
}
```

```java
@Data
public class UserResource {
    @NotNull(message = "name must not be null")
    private String name;
    private int age;
}
```

## 总结

- `validation-api`是Bean Validation规范定义的API，`hibernate-validator`是对应的默认实现和事实标准。
- ExceptionHandler：格式化验证结果输出
- Jakarta Bean Validation 2.0规范，提供了更丰富的内建功能
- 对嵌套结构验证时，使用@Valid