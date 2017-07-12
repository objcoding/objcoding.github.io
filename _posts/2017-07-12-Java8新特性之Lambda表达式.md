---
layout: post
title: "Java8新特性之Lambda表达式"
categories: Java
tags: Java8 Lambda
author: zch
---

* content
{:toc}
前面的文章很多都用到了Lambda表达式，Lambda表达式是一种函数式编程的风格，以前我们是这样实现多线程编程的：

```java
new Thread(new Runnable(){
             public void run(){
                System.out.println("帅比辉");
             }
        };
).start();
```

如上，用到了匿名内部类，但这样做会造成代码重复率多，且臃肿，不易阅读，我们通过Lambda表达式改进一下：

```java
new Thread(() -> {
		System.out.println("帅比辉");
}).start();
```

look，像上面写是不是很紧凑，是不是很喜欢，是不是看得有点懵逼？别急，接下来慢慢讲解：







## 函数式接口

想要用Lambda表达式需要满足方法参数需要是一个接口类型的参数，比如Thread的构造方法的参数就是一个Runnable接口，且接口只能有一个需要实现的方法，Runnable接口只有一个实现方法run()，Java8本身提供了很多这种函数式接口，如：

- Predicate 接口只有一个参数，返回boolean类型。该接口包含多种默认方法来将Predicate组合成其他复杂的逻辑（比如：与，或，非）：

```java
// 源码
@FunctionalInterface
public interface Predicate<T> {

    boolean test(T t);

    default Predicate<T> and(Predicate<? super T> other) {
        Objects.requireNonNull(other);
        return (t) -> test(t) && other.test(t);
    }

    default Predicate<T> negate() {
        return (t) -> !test(t);
    }

    default Predicate<T> or(Predicate<? super T> other) {
        Objects.requireNonNull(other);
        return (t) -> test(t) || other.test(t);
    }
  
    static <T> Predicate<T> isEqual(Object targetRef) {
        return (null == targetRef)
                ? Objects::isNull
                : object -> targetRef.equals(object);
    }
}
```

```java
Predicate<String> p = (s) -> s.length() > 0;
```

- Function接口有一个参数并且返回一个结果，并附带了一些可以和其他函数组合的默认方法（compose, andThen）：

```java
// 源码
@FunctionalInterface
public interface Function<T, R> {

    R apply(T t);
  
    default <V> Function<V, R> compose(Function<? super V, ? extends T> before) {
        Objects.requireNonNull(before);
        return (V v) -> apply(before.apply(v));
    }
  
    default <V> Function<T, V> andThen(Function<? super R, ? extends V> after) {
        Objects.requireNonNull(after);
        return (T t) -> after.apply(apply(t));
    }
  
    static <T> Function<T, T> identity() {
        return t -> t;
    }
}
```

```java
Function<String, Integer> toInteger = (s) -> Integer.valueOf(s);
Function<String, String> backToString = toInteger.andThen((i) -> String.valueOf(i));
backToString.apply("123");
```

- Supplier接口返回一个任意范型的值，和Function接口不同的是该接口没有任何参数：

```java
// 源码
@FunctionalInterface
public interface Supplier<T> {
    T get();
}
```

```java
Supplier<String> supplier = () -> "帅比辉";
```

- Consumer接口表示执行在单个参数上的操作：

```java
@FunctionalInterface
public interface Consumer<T> {

    void accept(T t);

    default Consumer<T> andThen(Consumer<? super T> after) {
        Objects.requireNonNull(after);
        return (T t) -> { accept(t); after.accept(t); };
    }
}
```

```java
Consumer<Person> greeter = (p) -> System.out.println("Hello, " + p.firstName);
greeter.accept(new Person("Luke", "Skywalker"));
```





Java8自带还有很多这样的函数式接口，这里就不一一介绍了。

我们也可以自定以一个函数式接口：

```
@FunctionalInterface
public interface MyFunction<T> {
    T get(T t);
}
```

```java
MyFunction<Integer> function = (t) -> 100 * t;
```



## 语法







## 方法引用





其实把Function那个实例改成方法引用更为恰当：

```java
Function<String, Integer> toInteger = Integer::valueOf;
Function<String, String> backToString = toInteger.andThen(String::valueOf);
backToString.apply("123");
```

