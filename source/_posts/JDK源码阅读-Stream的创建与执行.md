---
title: JDK源码阅读-Stream的创建与执行
abbrlink: 52053
date: 2020-11-26 22:30:49
tags: [Java, JDK, Stream]
categories: 源码阅读
---
本文主要涉及java.util.stream包的代码, Stream的创建与执行，所使用JDK源码版本为jdk-11.0.3.

<!-- more -->

# JDK源码阅读-Stream的创建与执行

## 写在前面

本文主要涉及java.util.stream包的代码，所使用JDK源码版本为jdk-11.0.3.

一个简单的Stream例子：

```java
import java.util.stream.Stream;

public class Test {

    public static void main(String[] args) {
        Stream.of(1, 2, 3, 4, 5, 6, 7, 8, 9)
                .map(item -> item * 2)
                .forEach(item -> System.out.print(item + " "));

        System.out.println();

        Stream.of(1, 2, 5, 3, 4, 5, 6, 7, 8, 9)
                .distinct()
                .parallel()
                .map(item -> item * 2)
                .forEach(item -> System.out.print(item + " "));

        System.out.println();

        Stream.of(1, 2, 3, 4, 5, 6, 7, 8, 9)
                .parallel()
                .map(item -> item * 2)
                .forEachOrdered(item -> System.out.print(item + " "));
    }
}
```

输出：

>2 4 6 8 10 12 14 16 18 
> 12 10 14 6 18 8 4 2 16 
> 2 4 6 8 10 12 14 16 18 

## 总览

### Stream相关的类与接口

<img src="/images/stream-class.png" width = "600" height = "200" />
Stream相关的接口与类均在java.util.stream包中。BaseStream是所有Stream的公共接口，提供类基本的iterator、sequential、parallel、onClose接口。Stream（可以理解为引用类型）、IntStream、LongStream、DoubleStream扩展了BaseStream接口并提供了对应类型的接口，比如：相比Stream接口，IntStream、LongStream、DoubleStream拥有特有的average、sum等接口。

每种Stream又有对应pipeline的实现类：ReferencePipeline、IntPipeline、LongPipeline、DoublePipeline，pipeline的概念下一节具体介绍。

### Stream pipeline

Stream的执行过程被抽象出一个pipeline的概念，每个pipeline会包含不同的阶段(stage):

- 起始阶段(source stage)，有且仅有一个，Stream创建的时候即被创建，比如：通过Stream.of接口创建时，会实例化ReferencePipeline.Head作为Stream的起始阶段；
- 过程阶段(intermediate stage)，0个或多个，如下例中包含两个过程阶段：distinct、map，parallel是一个特殊的存在，它只是设置一个标志位， 并不是一个过程阶段；对于过程阶段的各种操作，又有无状态操作(StatelessOp)和有状态操作(StatefulOp)之分, 比如：对于distinct、dropWhile、sorted需要在执行过程种记录执行状态，即有状态操作，而map、filter则属于无状态操作;
- 终端阶段(terminal stage)，有且仅有一个，用于结果计算或者执行一些副作用，如下例中的forEach。

```java
Stream.of(1, 2, 5, 3, 4, 5, 6, 7, 8, 9)
                .distinct()
                .parallel()
                .map(item -> item * 2)
                .forEach(item -> System.out.print(item + " "));
```

上例中，最终构造的pipeline如图所示，pipeline数据结构是一个双向链表，每个节点分别存储上一阶段，下一阶段，及起始阶段。终端操作前均为lazy操作，所有操作并未真正执行。而终端操作会综合所有阶段执行计算。
![Alt text here](/images/stream.dio.png)

## Stream源码分析

### Stream的创建

Stream、IntStream、LongStream、DoubleStream接口都提供了静态方法`of`，用于便捷地创建Stream，分别用于创建引用类型、int、long、double的Stream。以Stream接口为例，传入一系列有序元素，比如`Stream.of(1, 2, 3, 4, 5, 6, 7, 8, 9)`。如下所示，`Stream.of`方法通过调用`Arrays.stream`实现。所以`Stream.of(1, 2, 3, 4, 5, 6, 7, 8, 9)`等同于`Arrays.stream(new int[]{1, 2, 3, 4, 5, 6, 7, 8, 9})`

```java_holder_method_tree
//Stream.java
public static<T> Stream<T> of(T... values) {
        return Arrays.stream(values);
    }
```

```java_holder_method_tree
//Arrays.java
public static <T> Stream<T> stream(T[] array) {
        return stream(array, 0, array.length);
    }
    
public static <T> Stream<T> stream(T[] array, int startInclusive, int endExclusive) {
        return StreamSupport.stream(spliterator(array, startInclusive, endExclusive), false);
    }
```

StreamSupport是一个工具类用于创建顺序或并行Stream。StreamSupport.stream需要两个参数：

- spliterator，Spliterator类型，可通过`Spliterators.spliterator`创建，用于遍历/拆分数组，
- parallel，boolean类型，标示是否并行，默认为false。

ReferencePipeline实现了stream接口，是实现Stream过程阶段或起始阶段的抽象基类。ReferencePipeline.Head是原始阶段的实现。

```
//StreamSupport.java
public static <T> Stream<T> stream(Spliterator<T> spliterator, boolean parallel) {
        Objects.requireNonNull(spliterator);
        return new ReferencePipeline.Head<>(spliterator,
                                            StreamOpFlag.fromCharacteristics(spliterator),
                                            parallel);
    }
```

至此，Stream创建的简单流程就完成了，IntStream、LongStream、DoubleStream的创建也是类似的。

### Stream执行过程

以下面的测试代码为例：

```java
Stream.of(1, 2, 5, 3, 4, 5, 6, 7, 8, 9)
                .distinct()
                .parallel()
                .map(item -> item * 2)
                .forEach(item -> System.out.print(item + " "));
```

#### distinct

当我们调用`distinct`方法期望去除重复元素时，会执行如下代码，通过`DistinctOps.makeRef`创建并append一个disctinct的操作到当前Stream。具体代码如下：

```java
//ReferencePipeline.java
// Stateful intermediate operations from Stream
@Override
    public final Stream<P_OUT> distinct() {
        return DistinctOps.makeRef(this);
    }
```

```java
/**
     * Appends a "distinct" operation to the provided stream, and returns the
     * new stream.
     *
     * @param <T> the type of both input and output elements
     * @param upstream a reference stream with element type T
     * @return the new stream
     */
    static <T> ReferencePipeline<T, T> makeRef(AbstractPipeline<?, T, ?> upstream) {
        //实例化一个有状态操作。
        return new ReferencePipeline.StatefulOp<T, T>(upstream, StreamShape.REFERENCE,
                                                      StreamOpFlag.IS_DISTINCT | StreamOpFlag.NOT_SIZED) {
            //并行计算去除重复元素
            @Override
            <P_IN> Node<T> opEvaluateParallel(PipelineHelper<T> helper,
                                              Spliterator<P_IN> spliterator,
                                              IntFunction<T[]> generator) {
                ...
            }

            //Lazy执行，包装DistinctSpliterator对象，用于后续执行去除重复元素
            @Override
            <P_IN> Spliterator<T> opEvaluateParallelLazy(PipelineHelper<T> helper, Spliterator<P_IN> spliterator) {
                ...
            }

            @Override
            Sink<T> opWrapSink(int flags, Sink<T> sink) {
                ...
        };
    }
```

#### map

`map`是无状态操作，所以只需实例化一个`StatelessOp`，而`opWrapSink`则是包装一层mapper的执行操作。

```java
@Override
    @SuppressWarnings("unchecked")
    public final <R> Stream<R> map(Function<? super P_OUT, ? extends R> mapper) {
        Objects.requireNonNull(mapper);
        return new StatelessOp<P_OUT, R>(this, StreamShape.REFERENCE,
                                     StreamOpFlag.NOT_SORTED | StreamOpFlag.NOT_DISTINCT) {
            @Override
            Sink<P_OUT> opWrapSink(int flags, Sink<R> sink) {
                return new Sink.ChainedReference<P_OUT, R>(sink) {
                    @Override
                    public void accept(P_OUT u) {
                        downstream.accept(mapper.apply(u));
                    }
                };
            }
        };
    }
```

#### parallel

parallel并不是pipeline种的一个阶段，pipeline相关的实现类(如：ReferencePipeline、Intpipeline)有一个共同的基类AbstractPipeline，其上维护了一个变量parallel来标示pipeline是否并行, 仅维护在起始阶段的结构上。如下代码所示：

```java
//AbstractPipeline.java
/**
     * True if pipeline is parallel, otherwise the pipeline is sequential; only
     * valid for the source stage.
     */
    private boolean parallel;
```

设置Stream为并行模式

```java
//AbstractPipeline.java
    @Override
    @SuppressWarnings("unchecked")
    public final S parallel() {
        sourceStage.parallel = true;
        return (S) this;
    }
```

设置Stream为串行模式

```java
//AbstractPipeline.java
    @Override
    @SuppressWarnings("unchecked")
    public final S sequential() {
        sourceStage.parallel = false;
        return (S) this;
    }
```


#### forEach

forEach是一个终端操作，同样，通过工厂方法`ForEachOps.makeRef`创建一个forEach的操作，用于执行Stream的最终计算。

```java
// Terminal operations from Stream

    @Override
    public void forEach(Consumer<? super P_OUT> action) {
        evaluate(ForEachOps.makeRef(action, false));
    }
```

evaluate是一个通用的终端操作求值方法，对于forEach、reduce等终端操作均执行该方法，接收`TerminalOp`对象，首先判断`isParallel()`,如需并行执行`terminalOp.evaluateParallel`, 否则执行`terminalOp.evaluateSequential`。不管是执行那个方法都需要两个参数：

- this, 是起始操作或最后一个过程操作（本例中是`map`操作），此参数为`PipelineHelper`类型，从类图结构看，`PipelineHelper`是所有pipeline类型的顶级父类，用于Stream pipeline的执行，描述了Stream pipeline的所有信息。
- `sourceSpliterator(terminalOp.getOpFlags())`，Spliterator类型，用于遍历/拆分数组，对于串行执行或者无状态并行执行的Stream，其为起始阶段默认创建的Spliterator对象，否则需要对所有有状态操作进行计算。

```java
// Terminal evaluation methods

    /**
     * Evaluate the pipeline with a terminal operation to produce a result.
     *
     * @param <R> the type of result
     * @param terminalOp the terminal operation to be applied to the pipeline.
     * @return the result
     */
    final <R> R evaluate(TerminalOp<E_OUT, R> terminalOp) {
        assert getOutputShape() == terminalOp.inputShape();
        if (linkedOrConsumed)
            throw new IllegalStateException(MSG_STREAM_LINKED);
        linkedOrConsumed = true;

        return isParallel()
               ? terminalOp.evaluateParallel(this, sourceSpliterator(terminalOp.getOpFlags()))
               : terminalOp.evaluateSequential(this, sourceSpliterator(terminalOp.getOpFlags()));
    }
```

对于`evaluateSequential`,分两部分执行：

- `wrapSink`,将所有过程操作比如map、filter等包装成一个Sink；
- `copyInto`,执行计算


```java
        @Override
        public <S> Void evaluateSequential(PipelineHelper<T> helper,
                                           Spliterator<S> spliterator) {
            return helper.wrapAndCopyInto(this, spliterator).get();
        }
```

```java
    @Override
    final <P_IN, S extends Sink<E_OUT>> S wrapAndCopyInto(S sink, Spliterator<P_IN> spliterator) {
        copyInto(wrapSink(Objects.requireNonNull(sink)), spliterator);
        return sink;
    }
```

```java
    @Override
    final <P_IN> void copyInto(Sink<P_IN> wrappedSink, Spliterator<P_IN> spliterator) {
        Objects.requireNonNull(wrappedSink);

        if (!StreamOpFlag.SHORT_CIRCUIT.isKnown(getStreamAndOpFlags())) {
            wrappedSink.begin(spliterator.getExactSizeIfKnown());
            spliterator.forEachRemaining(wrappedSink);
            wrappedSink.end();
        }
        else {
            copyIntoWithCancel(wrappedSink, spliterator);
        }
    }
```

对于`evaluateParallel`, 则拆分成多个task，并行执行，基于Fork/Join框架实现。

```java
        @Override
        public <S> Void evaluateParallel(PipelineHelper<T> helper,
                                         Spliterator<S> spliterator) {
            if (ordered)
                new ForEachOrderedTask<>(helper, spliterator, this).invoke();
            else
                new ForEachTask<>(helper, spliterator, helper.wrapSink(this)).invoke();
            return null;
        }
```

## 总结

- 可使用`Stream`、`IntStream`、`LongStream`、`DoubleStream`接口或`Array.stream`来创建一个`Stream`，不同类型的`Stream`间可通过`mapToObj`、`mapToInt`互相转换
- Stream的执行过程被抽象出一个pipeline的概念，每个pipeline会包含不同的阶段(stage)：起始阶段、过程阶段、终端阶段，用于结果计算的数据和策略的准备。
- 对于过程阶段的操作，又有无状态操作(StatelessOp)和有状态操作(StatefulOp)之分, 比如：对于distinct、dropWhile、sorted需要在执行过程种记录执行状态，即有状态操作