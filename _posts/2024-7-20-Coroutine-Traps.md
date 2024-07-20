---
layout: post
title: Coroutine 陷阱
---

Coroutine 并不是函数，但它的行为看起来很像同步调用的函数。
它存在的意义也是为用户提供一个类同步调用的异步编程环境。
对 Coroutine 不是很了解的程序员就会对其作出错误的假设，
从而导致一些所谓的灵异事件。
本文主要介绍一些和 Coroutine 有关的对象生命期问题。

## 调用协程

```c++
int func()
{
    // ...
    return 1;
}

task<int> coro()
{
    // ...
    co_return 1;
}

int iresult = func();
task<int> ifut = coro();
```

其中 `func()` 调用 `func` 函数，执行其函数体，并在最后返回一个`int`类型的值，存储在 `iresult` 中。
但 `coro()` 调用，并不直接执行其*协程体*（即`//...; co_return 1; `），
而首先隐式调用 `operator new` 为该协程分配其frame内存，**按用户配置**构造一个类 *future/promise* 对象对。
在用户编写的*协程体*代码之前，编译器会插一个 Initial Suspend 点，其行为由用户配置为挂起或不挂起。
若用户在此处配置挂起，那么`coro()`调用返回一个类*future*的协程句柄；若用户配置不挂起，则执行协程体，
在遇到下一个挂起点时，`coro()`调用才会返回。之后用户可以凭返回的类*future*对象操作该协程，如销毁(destroy)，重入(resume)等。

> 按习惯称在 Initial suspend point 挂起的协程为 lazy task; 
> 在 Initial suspend point 不挂起的协程为 eager task.
> eager 和 lazy 是针对何时开始执行协程体而言的。

挂起点为协程提供了异步运行的机会，不像函数，协程无需在其主调方返回前完成执行。它的控制流可以跨越多个函数调用
(某函数可以构造一个协程，并将其句柄返回到其主调函数，如下文示例代码函数`foo`)
而基于scope的对象生命期并不能跟踪这一点。因此，在协程形参中使用引用要格外主要，防止垂悬引用。

## 形参垂悬引用陷阱

```
// The real coroutine
task<int> coro(const int& iref)
{
    co_return 1 + iref;
}

// A function just return a coroutine handler.
task<int> foo()
{
    int i{};
    auto coro_handle = coro(i);
    return coro_handle;
}

// The function actually execute the coroutine.
void bar()
{
    auto coro_handle = foo();
    coro_handle.resume();
    int result = coro_handle.get_result();
}
```

上示例代码中函数 `foo` 调用 `coro(i)` 时，将本地（栈上）变量的引用传递给 `coro`， 
我们在此处假设 `coro` 在其 Initial Suspend 点挂起，即并不执行其*协程体* `co_return 1 + i;`，直接返回一个句柄给 `foo`。
而 `foo` 拿到句柄后什么也不做，将其返回给其主调函数 `bar`，由 `bar` 来 `resume` `coro` 协程。
这样 coro 的执行实际上跨越了两个函数，也就是说其部分执行流离开了变量 `i` 所在的 scope。
那么，在 `bar` 中重入(`resume`) `coro` 后，`iref` 为垂悬引用，其原先引用的对象`i`生命已经结束。

## 协程 Lambda 捕获陷阱

由于 lambda 本身在首个实际的挂起点返回，并destruct(其捕获的对象也一同destruct)。
所以，此后访问该lambda捕获的对象就会导致 use-after-free 或者 stack-use-after-return bug.

```c++
task<> coro()
{
    auto a = func1();
    auto b = func2();
    auto c = func3();

    auto callback = [a, b, c] -> task<> {
        co_await something_suspend();
        do_something(a, b, c); // use-after-free
    }
    
    return callback();
}
```

不难想到可以通过按值传递的方式避开这个问题（`auto callback = [] (auto a, auto b, auto c) -> task<> { ... }`），
但实际调用lambda的场景不被程序员所控制（如将其作为回调函数传递到其他地方）。
除此之外还可以嵌套lambda，通过外层 lambda 捕获对象:

```c++
auto outter_lambda = [a, b, c] { 
    return [](auto a, auto b, auto c) -> lazy_task<> { 
        co_await something_suspend();
        co_await do_somethng(a, b, c);
    }(a, b, c);
};
```

不过我觉得这样有点麻烦。

在 eager 的 协程 lambda 中可以在首个挂起点之前备份捕获的变量。
不过在代码的后续维护过程中程序员仍可能以外地访问到旧的对象，导致 stack-use-after-return.
