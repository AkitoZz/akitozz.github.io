---
layout: post
title: aysncio事件循环冲突解决
categories: [Python, asyncio, event_loop]
description: Python, asyncio, event_loop
keywords: Python, asyncio, 异常处理, event_loop
---

## 问题现象

最近在用asyncio调用motor实现mongodb异步读写的时候，遇到了事件循环冲突的问题，代码和报错如下

代码

```python

import asyncio
import motor.motor_asyncio

classMongoUtil:
    def__init__(self):
        conn = motor.motor_asyncio.AsyncIOMotorClient()
        db = conn.exercise
        self.collection = db.person_info

    asyncdefread_people(self):
        asyncfordocinself.collection.find({}, {‘_id’: 0}):
        print(doc)

util = MongoUtil()
asyncio.run(util.read_people())


```

报错

```python

RuntimeError: Task <Task pending name=’Task-1′ coro=<MongoUtil.read_people() running at /Users/mika/py_workspace/test/main.py:12> cb=[_run_until_complete_cb() at /Users/mika/opt/anaconda3/envs/py38/lib/python3.8/asyncio/base_events.py:184]> got Future <Future pending cb=[_chain_future.<locals>._call_check_cancel() at /Users/mika/opt/anaconda3/envs/py38/lib/python3.8/asyncio/futures.py:360]> attached to a different loop


```

其中最关键的一句话是：attached to a different loop

这段报错说明当前的这个线程里面，在运行asyncio.run之前，就已经存在一个事件循环了。而根据 asyncio 的规定，一个线程里面只能有一个事件循环正在运行，所以就导致报错。


## 关于asyncio

众所周知，Python的并发编程长期以来一直受到批评。例如，Python 多线程实际上并不是真正的多线程，它使用 GIL 在线程之间切换。只有多进程才能真正发挥多个 CPU 核心的优势。然而，从 2014年的Python3.4开始，随着 asyncio 模块的引入，线程已成为Python中异步编程的核心。

在早期版本（Python3.7之前）中，调用协程需要明确控制事件管理器。代码大致如下：

```python
import asyncio
import aiohttp

async def main():
    async with aiohttp.ClientSession() as client:
        resp = awaitclient.get(‘http://httpbin.org/ip’)
        ip = awaitresp.json()
        print(ip)

loop = asyncio.get_event_loop()
loop.run_until_complete(main())

```

get_event_loop 函数明确创建一个事件循环，然后运行它使用 run_until_complete 注册的协程。然而从 Python 3.7 开始，引入了 run 方法，它会自动创建并管理事件循环，因此用法比以前简单多了。

```python
import asyncio
import aiohttp
asyncdefmain():
    async with aiohttp.ClientSession() as client:
        resp = awaitclient.get('http://httpbin.org/ip')
        ip = awaitresp.json()
        print(ip)
asyncio.run(main())
```


## 错误原因

1. 首先是[官方文档](https://docs.python.org/3.10/library/asyncio-task.html#asyncio.run)中有如下说明

![offical doc](/images/blog/2025-01-26_1.png)

图中明确说明当另一个 asyncio 事件循环正在当前线程运行的时候，不能调用这个函数。
那么，除了由 asyncio.run() 自动创建的事件循环之外，还在哪里创建事件循环呢？如果我们查看 motor.motor_asyncio.AsyncIOMotorClient 的源代码，我们可以看到我们可以传递一个事件循环参数，如果我们不传递事件循环参数，则会创建一个新的事件循环。这将在 __init__ 函数中创建一个新的事件循环，导致后续的 asyncio.run() 失败。

2. [程序中所使用的motor版本源码](https://github.com/mongodb/motor/blob/4c7534c6200e4f160268ea6c0e8a9038dcc69e0f/motor/core.py#L154)


![motor code](/images/blog/2025-01-26_2.png)

也就是在初始化函数中执行了get_event_loop，由于当前没有正在运行的事件循环，所以asyncio.get_event_loop就会创建一个，并让它运行起来。
所以当我们使用 Motor 初始化 MongoDB 的连接时，就已经创建了一个事件循环了。但当代码运行到asyncio.run的时候，又准备创建一个新的事件循环，自然而然程序就运行错了。



## 解决方法

有两种方法可以解决此问题：
- 首先，可以通过外部使用asyncio.get_event_loop()来获取当前线程的事件循环（即motor创建的事件循环）。这使得你可以避免有多个事件循环。


```python

import asyncio
import motor.motor_asyncio

class MongoUtil:
    def __init__(self):
        conn = motor.motor_asyncio.AsyncIOMotorClient()
        db = conn.exercise
        self.collection = db.person_info

    async def read_people(self):
        async for doc in self.collection.find({}, {‘_id’: 0}):
            print(doc)

util = MongoUtil()
loop = asyncio.get_event_loop()
loop.run_until_complete(util.read_people())
```

- 另一个选择是升级组件版本。从3.0版本开始，AsyncIOMotorClient在初始化时不再自动创建事件循环，因此不再发生冲突。

## 后记

深入研究 asyncio 如何解决此错误，我们发现事件循环就像协程的调度程序一样。协程具有可等待属性（awaitable），这意味着它们可以暂停自身并使用 await 关键字将控制权返回给事件循环。

这使得事件循环可以调度和运行其他协程，实现并发。这一切都发生在单个线程中，线程之间没有上下文切换，这使得协程成为一种轻量级且高效的并发方式。

而遇到报错的时候，阅读源码以及文档通常能帮助你准确定位到问题所在。
