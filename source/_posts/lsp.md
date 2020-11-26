---
title: 细说 LSP（里氏替换原则）
tags: 设计原则
categories: 设计原则
abbrlink: 37579
date: 2020-11-21 08:58:21
---

曾经在一次给新入职员工做Java开发的课程培训，讲到面向对象的SOLID设计原则时，大家普遍认为"里氏替换原则"是其中最难理解的一个。也许直接上UML并不是一个好的解释方法，从头说起才能弄清来龙去脉。

<!-- more -->

## 为什么叫里氏替换原则？
里氏替换原则在SOLID这五个设计原则中是比较特殊的存在：

- 如果违反了里氏替换原则，不只是降低软件设计的优雅性，很可能会导致Bug
- 只有里氏替换原则是以人名命令的
  
里氏替换原则译自Liskov substitution principle。Liskov是一位计算机科学家，也就是Barbara Liskov，麻省理工学院教授，也是美国第一个计算机科学女博士，师从图灵奖得主John McCarthy教授，人工智能概念的提出者。

里氏替换原则最初由Barbara Liskov在1987年的一次学术会议中提出，而真正正式发表是在1994年，Barbara Liskov 和 Jeannette Wing发表的一篇学术论文《A behavioral notion of subtyping》.

## 什么是里氏替换原则？
里氏替换原则在1994年Barbara Liskov 和 Jeannette Wing发表论文中的描述是：
>If S is a declared subtype of T, objects of type S should behave as objects of type T are expected to behave, if they are treated as objects of type T

从字面上翻译：如果S是T的子类型，对于S类型的任意对象，如果将他们看作是T类型的对象，则对象的行为也理应与期望的行为一致。

而另一种关于里氏替换原则的描述为Robert Martin在《敏捷软件开发：原则、模式与实践》一书中对原论文的解读：子类型（subtype）必须能够替换掉他们的基类型（base type）。这个是更简明的一种表述。

## 如何理解里氏替换原则？
不管是Barbara Liskov论文中的表述，还是Robert Martin的解读，都是比较抽象的表达。要理解里氏替换原则，其实就是要理解两个问题：

- 什么是替换？
- 什么是与期望行为一致的替换（Robert Martin所说的“必须能够替换”）？

### 什么是替换

替换的前提是面向对象语言所支持的多态特性，同一个行为具有多个不同表现形式或形态的能力。以JDK的集合框架为例，`List`接口的定义为有序集合，`List`接口有多个派生类，比如大家耳熟能详的`ArrayList`, `LinkedList`。那当某个方法参数或变量是`List`接口类型时，既可以是`ArrayList`的实现, 也可以是`LinkedList`的实现，这就是替换。

举个简单的例子：

```java
public String getFirst(List<String> values) {
        return values.get(0);
}
```

对于`getFirst`方法，接受一个`List`接口类型的参数，那既可以传递一个`ArrayList`类型的参数：

```java
List<String> values = new ArrayList<>();
values.add("a");
values.add("b");
String firstValue = getFirst(values);
```

又可以接收一个`LinkedList`参数：

```java
List<String> values = new LinkedList<>();
values.add("a");
values.add("b");
String firstValue = getFirst(values);
```

### 什么是与期望行为一致的替换？
在不了解派生类的情况下，仅通过接口或基类的方法，即可清楚的知道方法的行为，而不管哪种派生类的实现，都与接口或基类方法的期望行为一致。或者说接口或基类的方法是一种契约，使用方按照这个契约来使用，派生类也按照这个契约来实现。这就是与期望行为一致的替换。继续以上节中的例子说明：

```java
public String getFirst(List<String> values) {
        return values.get(0);
}
```

对于`getFirst`方法，接收`List`类型的参数，而`List`类型的`get`方法返回特定位置的元素，对于本例即为第一个元素。这些是不依赖派生类的知识的。所以对于上节中的示例，不管是`ArrayList`类型的实现，还是`LinkedList`的实现，`getFirst`方法最终的返回值是一样的。这就是与期望行为一致的替换。

## 违反里氏替换原则的场景
从直观上可能觉得派生类对象可以在替换其基类对象是理所当然的，但会有出现一些场景有意无意地违反了里氏替换原则。

### 子类中抛出了基类未定义的异常
还以JDK的集合框架为例，如果自定义一个List的派生类，如下：

```java
class CustomList<T> extends ArrayList<T> {
    @Override
    public T get(int index) {
        throw new UnsupportedOperationException();
    }
}
```

仅重写get方法，throw一个`UnsupportedOperationException`，因为`List`接口关于get方法的描述，仅会抛出`IndexOutOfBoundsException`, throw `UnsupportedOperationException`的行为并不是基类所期望的，即违反了里氏替换原则，

```java
/**
     * Returns the element at the specified position in this list.
     *
     * @param index index of the element to return
     * @return the element at the specified position in this list
     * @throws IndexOutOfBoundsException if the index is out of range
     *         ({@code index < 0 || index >= size()})
     */
    E get(int index);
```

### 子类改变了基类方法的语义或引入了副作用
同样，如果自定义另一个List的派生类，如下：

```java
class CustomList<T> extends ArrayList<T> {
    @Override
    public T get(int index) {
        if (index >= size()){
            return null;
        }
        return get(index);
    }
}
```

仅重写get方法，当输入index大于当前list的size时，返回null，而不抛出`IndexOutOfBoundsException`, 因为`List`接口关于get方法的描述，当index超出范围时抛出`IndexOutOfBoundsException`，所以改变了基类方法的语义，即违反了里氏替换原则。

## 违反里氏替换原则的危害
当我们违反了这一原则会带来有一些危害：

- 反直觉。期望所有子类行为是一致的，但如果不一致可能需要文档记录，或者在代码跑失败后涨此知识；
- 不可读。如果子类行为不一致，可能需要不同的逻辑分支来适配不同的行为，徒增代码复杂度；
- 不可用。可能出错的地方终将会出错。

## 如何避免违反里氏替换原则
谈到如何避免，当然要基于里氏替换原则的定义，`与期望行为一致的替换`。

- 从行为出发来设计。在做抽象或设计时，不只是要从模型概念出发，还要从行为出发，比如一个经典的例子，正方形和长方形，从现实的概念中正方形是一个长方形，但是在计算其面积的行为上是不一致的。
- 基于契约设计。这个契约即是基类方法签名、功能描述、参数类型、返回值等。在派生类的实现时，时刻保持派生类与基类的契约不被破坏。

## 参考

1. https://en.wikipedia.org/wiki/Liskov_substitution_principle
2. https://reflectoring.io/lsp-explained/
3. https://medium.com/hackernoon/liskov-substitution-principle-a982551d584a
4. 《敏捷软件开发：原则、模式与实践》
