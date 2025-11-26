---
layout: post
categories: 技术文章
author: chen
title: CompletableFuture如何使用
---
## 一、知识扫盲

是什么？

为多线程使用提供了更强大的异步任务编排能力，在大型应用rpc调用众多地场景下更简易地实现并发调用

## 二、使用举例

### 写在前面

CompletableFuture提供了众多api如下，可以先用一个tip过个眼熟

1.以Async结尾的方法，都是异步方法，对应的没有Async则是同步方法，一般都是一个异步方法对应一个同步方法。

2.以run开头的方法，其入口参数一定是无参的，并且没有返回值

3.以supply开头的方法，入口也是没有参数的，但是有返回值

4.以Accept开头或者结尾的方法，入口参数是有参数，但是没有返回值

5.以Apply开头或者结尾的方法，入口有参数，有返回值

![img](https://raw.githubusercontent.com/chen-1110/image/main/image-20210902093026059.png)

### 代码举例

#### 零依赖

![图6 零依赖](https://raw.githubusercontent.com/chen-1110/image/main/ff663f95c86e22928c0bb94fc6bd8b8715722.png)

```
ExecutorService executor = Executors.newFixedThreadPool(5);
//1、使用runAsync或supplyAsync发起异步调用
CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> {
  return "result1";
}, executor);
```

#### 一元依赖

![图7 一元依赖](https://raw.githubusercontent.com/chen-1110/image/main/373a334e7e7e7d359e8f042c7c9075e215479.png)

```
CompletableFuture<String> cf3 = cf1.thenApply(result1 -> {
  //result1为CF1的结果
  //......
  return "result3";
});
CompletableFuture<String> cf5 = cf2.thenApply(result2 -> {
  //result2为CF2的结果
  //......
  return "result5";
});
```

#### 二元依赖

![图8 二元依赖](https://raw.githubusercontent.com/chen-1110/image/main/fa4c8669b4cf63b7a89cfab0bcb693b216006.png)

```
CompletableFuture<String> cf4 = cf1.thenCombine(cf2, (result1, result2) -> {
  //result1和result2分别为cf1和cf2的结果
  return "result4";
});
```

#### 多元依赖

![图9 多元依赖](https://raw.githubusercontent.com/chen-1110/image/main/92248abd0a5b11dd36f9ccb1f1233d4e16045.png)

```
CompletableFuture<Void> cf6 = CompletableFuture.allOf(cf3, cf4, cf5);
CompletableFuture<String> result = cf6.thenApply(v -> {
  //这里的join并不会阻塞，因为传给thenApply的函数是在CF3、CF4、CF5全部完成时，才会执行 。
  result3 = cf3.join();
  result4 = cf4.join();
  result5 = cf5.join();
  //根据result3、result4、result5组装最终result;
  return "result";
});
```

#### 异常处理

1.task执行过程中出现异常（如NPE），不会抛出到主线程，只有调用get(), join()才会抛出异常

2.task执行过程中的异常，可以用whenComplete，exceptionally等方法处理该异常

```
CompletableFuture<String> future
        = CompletableFuture.supplyAsync(() -> {
    if (true) {
        throw new RuntimeException("Computation error!");
    }
    return "hello!";
}).exceptionally(ex -> {
    System.out.println(ex.toString());// CompletionException
    return "world!";
});
assertEquals("world!", future.get());
```

## 三、简单的源码分析

这里笔者不做太深入分析，简单说说，参考https://tech.meituan.com/2022/05/12/principles-and-practices-of-completablefuture.html

completableFuture主要的成员变量有两个，result和stack，result是当前cf的计算结果，stack是一个栈将依赖该cf的后续操作压入栈中

比如当我们执行cf2 = cf1.thenApplyAsync(fn2)时，结构如下图

![图12 thenApply简图](https://raw.githubusercontent.com/chen-1110/image/main/f45b271b656f3ae243875fcb2af36a1141224.png)

看下源码，当我们调用该方法时，会调用uniApplyStage函数，该函数中调用uniApply判断前置cf1任务是否执行完，如果不需要用线程池且前置任务执行完了就直接在当前线程执行fn2返回，若需要用线程池或者前置cf1任务未执行完，就加入观察者节点压入stack中，待cf1执行完便会判断stack是否空，若非空则执行fn2

```
    public <U> CompletableFuture<U> thenApplyAsync(
        Function<? super T,? extends U> fn) {
        return uniApplyStage(asyncPool, fn);
    }
    
    private <V> CompletableFuture<V> uniApplyStage(
        Executor e, Function<? super T,? extends V> f) {
        if (f == null) throw new NullPointerException();
        CompletableFuture<V> d =  new CompletableFuture<V>();
        if (e != null || !d.uniApply(this, f, null)) {
            UniApply<T,V> c = new UniApply<T,V>(e, d, this, f);
            push(c);
            c.tryFire(SYNC);
        }
        return d;
    }
    
```

