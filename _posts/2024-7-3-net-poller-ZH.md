---
layout: post
title: Async-IO on `io_uring(7)`
---

**本人项目目前使用 io_uring 作为异步IO的核心**，
但 *io_uring* 并不仅是一种通知机制。

## 相关技术简介

高性能并行系统上使用阻塞IO调用是不可接受的。
POSIX IO 函数如 `read`, `write`, `send` etc. 虽可通过 `fnctl` (`O_NONBLOCK`) 或调用时指定 `flags` 参数来实现非阻塞调用，
但不提供相关IO事件回调函数，因此需要一种通知机制来告知应用程序异步操作的状态。

## `select`, `poll`, `epoll` etc.

为了避免直接阻塞操作导致的阻塞期间计算资源浪费。
要在 socket 上执行IO操作，可以在多路复用设施上注册该 socket fd 相关IO事件。
当该 socket fd 可IO时，***先*由多路复用设备通知用户程序。*然后*，用户再在相关fd上执行非阻塞IO调用**。

为了实现这一需求，允许线程处理多路IO，POSIX 提供 `select(2)`, `poll(2)` 等设施实现多路复用。
在此基础上 Linux 还提供了 `epoll(2)`， 上述都是IO事件通知机制——注册的IO事件发生时，内核通知应用程序的一种手段，
但这些设施不能替用户程序执行IO操作。**具体的IO操作在用户收到该通知后，由用户处理。**

## `io_uring`

用法上和 epoll 相似，`io_uring(7)` 也需要用户将要执行的IO操作注册进设施。
不同的是，实际的IO操作是在一内核线程中，由 Linux io_uring 子系统(subsystem)执行。
并**在IO完成后通知用户**（epoll 是在IO操作可行时通知用户，由用户执行IO操作）。

实现上，在内核与用户空间共享的内存上构造两个无锁循环队列:
- Submission Queue (SQ)
- Completion Queue (CQ)

具体来说，用户将要执行的IO操作，描述成 SQE (Submission Queue Entry) 放入 SQ 中，
并通知内核（或由内核线程polling该SQE自动获悉变动）。
内核每完成一个SQE表示的IO操作，就在CQ中放一个CQE (Completion Qeueu Entry) 以通知用户。

## 事件驱动的应用

基于上述设施，用户可以实现出事件循环，辅以线程池实现事件驱动的异步IO运行时。
其中，事件循环的每次迭代分为两部分:

- 从多路复用设施获得已触发的事件。
    - 若没有触发的事件要处理，则可执行线程池任务队列中剩余的任务。
- 处理已触发的事件
    - 处理已触发的事件往往是调用用户程序设定的回调函数。
    - 或唤醒相关挂起的协程(如 C++20 Coroutine, goroutine etc.)

第二步中，用户回调函数往往会请求其他IO操作，进而在多路复用设施中注册其他事件。
如此驱动系统运行，得名事件驱动型应用程序(Event-Driven Application).

## Demo

处理触发的事件：
```c++
void iouring_event_loop_perthr::
do_occured_nonblk() noexcept
{
    auto lk = get_lk();
                                                                                
    const size_t left = ::io_uring_cq_ready(&m_ring);
    if (left == 0) 
    {
        mis_shot_this_time();
        return;
    }
    shot_this_time();
                                                                                
    ::io_uring_cqe* cqep{};
    for (size_t i{}; i < left; ++i)
    {
        int e = ::io_uring_peek_cqe(&m_ring, &cqep);
        // no any CQE available
        if (e == - EAGAIN) [[unlikely]] 
        {
            break;
        }
                                                                                
        dealwith_cqe(cqep);
        // mark this cqe has been processed
        ::io_uring_cqe_seen(&m_ring, cqep); 
    }
}

```

*此段代码出自本人异步协程项目: [Koios](https://github.com/JPewterschmidt/koios).*

## 技术特点

上文提到非功能方面 io_uring 通过两个在内核和用户态之间共享的循环无锁队列实现用户内核间IO任务的双向通知。
具体IO操作由io_uring子系统在内核空间完成，事后通知用户IO任务完成情况。
相比用户使用传统 POSIX IO，io_uring 免去了几乎所有的IO相关系统调用，减少大量执行上下文切换。
使用固定大小的循环无锁队列表示IO任务，实现用户和内核间异步的任务处理。
固定大小的队列一方面免去了用户态内核态之间复杂的动态内存管理；一方面减少缓存失配减少系统性能损失。

除此之外，功能方面 io_uring 还支持将多个IO操作进行链接，可使一组相互依赖的IO操作链成为一**原子**IO操作，
并可对该原子IO操作设置超时。链上任一IO操作失败或超时，会导致当前IO链剩余部分不再执行。

## 相关项目

- [liburing](https://github.com/axboe/liburing): liburing provides helpers to setup and teardown io_uring instances, and also a simplified interface for applications that don't need (or want) to deal with the full kernel side implementation.
- [libuv](https://github.com/libuv/libuv):  a multi-platform support library with a focus on asynchronous I/O.
- [QEMU](https://github.com/qemu/qemu): A generic and open source machine & userspace emulator and virtualizer.
- [Koios](https://github.com/JPewterschmidt/koios): A C++ async runtime library. (本人项目)

*未完待续*
