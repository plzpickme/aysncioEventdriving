# chapter1 异步事件驱动框架

## preface

异步事件驱动框架集中应该解决两个问题什么异步和什么是事件驱动。

其中异步可以理解等价为并发，即程序并不是单一的线性执行，而是从程序启动或者在执行到某个节点的时候，同时并发地执行的两个或者多个子程序，多个子程序之间互不影响，多个子程序之间如需访问请求同一资源，需使用锁机制，python多线程锁机制如下图，后文再做详细讨论。

![Screen Shot 2021-06-17 at 2.55.39 PM](/Users/zhengkaiyang/Desktop/chunlin_intern/high-frequency_trading_system/aioquant-master/docs_from_yang/image/python_MultiThread_lock.png)

同样我们可以通过下面这张图来讨论异步是如何工作的

![asyncio](/Users/zhengkaiyang/Desktop/chunlin_intern/high-frequency_trading_system/aioquant-master/docs_from_yang/image/asyncio.jpg)

MainThread主线程在执行完前面所有同步程序后到达异步程序起点即图中 submit async task 此时程序同时提交多个异步任务，这些异步任务互不干扰，并发执行，若主线程需要等待多个异步任务返回结果此时需要阻塞主线程，若不需要等待返回结果主线程可继续执行当前任务无需阻塞。python中实现异步的方式主要有多进程，多线程，协程的方式，其中任务切换开销从大到小为多进程，多线程，协程，而值得考虑的是python中的GIL（全局解释器锁 global interperter lock）的存在，无论是多进程，多线程，协程都无法实现并行即一个程序同时在多个cpu上运行实现无论在逻辑还是在物理层面都实现同时运行，而并发实则使用一个cpu只实现了逻辑上的并发即将一个cpu使用时间分成若干片段提供给不同的任务使用，只有在程序属于IO密集型时，并发机制能更好的利用CPU资源。异步事件驱动数字货币量化交易框架aioquant使用的是python3以后提供协程支持的原生库asyncio，协程相比较多进程多线程具有更加轻便切换消耗资源更少的优点，后文会详细介绍分析asyncio的作用和使用。

那么什么又是事件驱动，谈论事件驱动的同时我们需要了解量化交易中的两种主流回测框架：向量方式和事件驱动方式。在对时序因子进行回测的时，我们会拿到一种标的的时序数据，根据数据传入回测系统的方式进行分类，把时序数据以向量的方式整体传入回测系统的方式，称为向量方式，这种方式的特点是通过向量和矩阵运算就能对策略的表现作出评估，代码冗余少，结构简单，计算速度快是目前很主流的回测方式，比较适用于中低频策略，同时向量方式也有其缺点，因为将整体数据全部一次传入回测系统，对数据的处理方式是通过对向量矩阵进行操作，极有可能引入未来函数，同时向量方式回测框架并不能接入实盘进行交易，当我们进行交易时需要开发另一套交易系统。当然，事件驱动的交易回测系统就能完美解决上诉向量回测框架的问题，事件驱动是指通过将数据以事件的方式传入，即将数据以其最小单位（orderbook数据以一个tick，min_kline数据以一分钟）封装成一个事件传入回测或者交易系统，策略在拿到事件后根据自身的决策逻辑作出响应。事件驱动的整个过程完全真实的模拟了交易中的实际发生行为，故事件驱动的代码在回测和交易系统上的代码可重用性非常出色，仅仅只需要改动少许行情代码和撮合系统代码即可实现复用，同样由于回测系统获取数据的方式采用drip in的方式，每次回测系统只能拿到当前一条数据，所以从源头上禁止了未来函数使用的可能性。但是，无法避免地，事件驱动方式也有其明显的弊端，相比较向量方式，事件驱动方式的计算速度要慢很多，同样它的结构更为复杂，所以事件驱动方式更适合中高频策略。

谈论完什么是事件驱动过后，我们继续讨论下异步事件驱动数字货币交易框架aioquant如何实现事件驱动。

![rabbitmq](/Users/zhengkaiyang/Desktop/chunlin_intern/high-frequency_trading_system/aioquant-master/docs_from_yang/image/rabbitmq.jpg)

为了实现事件驱动，aioquant使用消息中间件rabbitmq，rabbitmq是基于和实现高级消息队列（AMQP）的开源消息代理软件，通过此消息中间件就可以实现各个模块之间的解耦并且实现在各个模块之间通信，事件的传递也通过消息中间件rabbitmq，当系统从历史数据或者交易所api中获得一个最小粒度的数据后，会对数据进行抽象加工成数据事件（orderbook事件 或者kline事件)这个事件将被行情服务模块投递到rabbitmq消息中间件中，通过这个消息中间件策略模块会事先订阅数据事件，若有任何数据事件传入rabbitmq，rabbitmq会立刻将事件推送给策略模块，策略模块可以获取事件中的数据，并对数据作出反应，同理其余模块也能通过rabbitmq实现完全解耦。

## asyncio库

[asyncio](https://docs.python.org/zh-cn/3/library/asyncio.html)里面主要有4个需要关注的基本概念

### Eventloop

Eventloop可以说是asyncio应用的核心是中央总控。Eventloop实例提供了注册、取消和执行任务和回调的方法。

把一些异步函数(就是任务，Task，一会就会说到)注册到这个事件循环上，事件循环会循环执行这些函数(但同时只能执行一个)，当执行到某个函数时，如果它正在等待I/O返回，事件循环会暂停它的执行去执行其他的函数；当某个函数完成I/O后会恢复，下次循环到它的时候继续执行。因此，这些异步函数可以协同(Cooperative)运行，这就是事件循环的目标。

### Coroutine

协程(Coroutine)本质上是一个函数，特点是在代码块中可以将执行权交给其他协程：

```python
import asyncio

async def a():
    print('Suspending a')
    await asyncio.sleep(0)
    print('Resuming a')

async def b():
    print('In b')

async def main():
    await asyncio.gather(a(), b())

if __name__ == '__main__':
    asyncio.run(main())
```

这里面有4个重要关键点：

1. 协程要用`async def`声明，Python 3.5时的装饰器写法已经过时，就不列出来了。
2. asyncio.gather用来并发运行任务，在这里表示协同的执行a和b2个协程
3. 在协程a中，有一句`await asyncio.sleep(0)`，await表示调用协程，sleep 0并不会真的sleep（因为时间为0），但是却可以把控制权交出去了。
4. asyncio.run是Python 3.7新加的接口，要不然你得这么写:

```python
loop = asyncio.get_event_loop()
loop.run_until_complete(main())
loop.close()
```

好了，我们先运行一下看看:

```bash
❯ python coro1.py
Suspending a
In b
Resuming a
```

在并发执行中，协程a被挂起又恢复过。

### Task

Eventloop除了支持协程，还支持注册Future和Task，2种类型的对象，那为什么要存在Future和Task这2种类型呢？

先回忆前面的例子，Future是协程的封装，Future对象提供了很多任务方法(如完成后的回调、取消、设置任务结果等等)，但是开发者并不需要直接操作Future这种底层对象，而是用Future的子类Task协同的调度协程以实现并发，简单来说，future是task的基类，通常情况下我们在开发过程中只需要用到task类，故不对future类做任何介绍。

task类是对future类的继承，也是对协程对象的封装，协程完成后的回调，协程任务当前的状态，协程任务的取消，协程任务的结果都被封装在task类中，使用async def 定义的函数在调用时都不会立即执行，而是会返回一个协程对象供使用。我们可以使用这个协程对象创建一个task的实例。

Task非常容易创建和使用:

```python
# 或者用task = asyncio.ensure_future(a())
In : task = loop.create_task(a())

In : task
Out: <Task pending coro=<a() running at /Users/dongwm/mp/2019-05-22/coro1.py:4>>

In : task.done()
Out: False

In : await task
Suspending a
Resuming a

In : task
Out: <Task finished coro=<a() done, defined at /Users/dongwm/mp/2019-05-22/coro1.py:4> result=None>

In : task.done()
Out: True
```

在创建了一个task 实例后，并不会立刻执行，我们需要将task放入eventloop事件循环中，task才会被eventloop根据情况分配cpu资源执行，我们接下来看看task是如何被执行的。

asyncio.create_task（）

```python
def create_task(coro):#coro 协程对象
    loop = events.get_running_loop()
    return loop.create_task(coro)
```

这里看到当我们调用asyncio.create_task（coro）时，已经初始化了一个eventloop对象，并且在loop对象中创建了一个task 任务。当然也可直接使用

```python
loop = asyncio.get_event_loop()
# 执行coroutine
loop.loop.create_task(coro）
loop.close()
```

来运行task任务。同时asyncio.ensure_future也可以实现task的注册运行，除了接受协程对象，还可以是Future对象或者awaitable对象:

1. 如果参数是协程，其实底层还是用的loop.create_task，返回Task对象
2. 如果是Future对象会直接返回
3. 如果是一个awaitable对象会await这个对象的__await__方法，再执行一次ensure_future，最后返回Task或者Future

所以就像ensure_future名字说的，确保这个是一个Future对象：Task是Future 子类，前面说过一般情况下开发者不需要自己创建Future。对于绝大多数场景要并发执行的是协程，所以直接用asyncio.create_task就足够了。

## docker

Docker 是一个开源的应用容器引擎，基于 Go 语言并遵从 Apache2.0 协议开源。Docker 可以让开发者打包他们的应用以及依赖包到一个轻量级、可移植的容器中，然后发布到任何流行的 Linux 机器上，也可以实现虚拟化。容器是完全使用沙箱机制，相互之间不会有任何接口（类似 iPhone 的 app）,更重要的是容器性能开销极低。简单来说，docker将你的应用程序和和各种环境打包放入一个容器（镜像）打包发布下载运行，在aioquant的框架中，消息中间件rabbitmq就是被放在docker容器中，如有兴趣可以查看[docker教程](https://www.bilibili.com/video/BV1og4y1q7M4?from=search&seid=10170020119404038765)

## [Rabbitmq](https://www.rabbitmq.com/documentation.html)

更推荐大家使用[rabbitmq中文文档](http://rabbitmq.mr-ping.com)

### RabbitMQ的特点

RabbitMQ是一款使用Erlang语言开发的，实现AMQP(高级消息队列协议)的开源消息中间件。首先要知道一些RabbitMQ的特点，**[官网](https://link.zhihu.com/?target=https%3A//www.rabbitmq.com/)**可查：

- 可靠性。支持持久化，传输确认，发布确认等保证了MQ的可靠性。
- 灵活的分发消息策略。这应该是RabbitMQ的一大特点。在消息进入MQ前由Exchange(交换机)进行路由消息。分发消息策略有：简单模式、工作队列模式、发布订阅模式、路由模式、通配符模式。
- 支持集群。多台RabbitMQ服务器可以组成一个集群，形成一个逻辑Broker。
- 多种协议。RabbitMQ支持多种消息队列协议，比如 STOMP、MQTT 等等。
- 支持多种语言客户端。RabbitMQ几乎支持所有常用编程语言，包括 Java、.NET、Ruby 等等。
- 可视化管理界面。RabbitMQ提供了一个易用的用户界面，使得用户可以监控和管理消息 Broker。
- 插件机制。RabbitMQ提供了许多插件，可以通过插件进行扩展，也可以编写自己的插件。

### RabbitMQ中的组成部分

从上面的HelloWord例子中，我们大概也能体验到一些，就是RabbitMQ的组成，它是有这几部分：

- Broker：消息队列服务进程。此进程包括两个部分：Exchange和Queue。
- Exchange：消息队列交换机。**按一定的规则将消息路由转发到某个队列。**
- Queue：消息队列，存储消息的队列。
- Producer：消息生产者。生产方客户端将消息同交换机路由发送到队列中。
- Consumer：消息消费者。消费队列中存储的消息。

这些组成部分是如何协同工作的呢，大概的流程如下，请看下图：

![rabbimq2](/Users/zhengkaiyang/Desktop/chunlin_intern/high-frequency_trading_system/aioquant-master/docs_from_yang/image/rabbimq2.jpg)

- 消息生产者连接到RabbitMQ Broker，创建connection，开启channel。
- 生产者声明交换机类型、名称、是否持久化等。
- 生产者发送消息，并指定消息是否持久化等属性和routing key。
- exchange收到消息之后，根据routing key路由到跟当前交换机绑定的相匹配的队列里面。
- 消费者监听接收到消息之后开始业务处理

### [aioamqp](https://aioamqp.readthedocs.io/en/latest/)

Aioamqp is a library to connect to an amqp broker. It uses asyncio under the hood.aioamqp

aioamqp是用来连接一个基于amqp协议服务进程的客户端库，在aioamqp底层使用asyncio进行异步并发。在aioquant异步事件驱动框架中，即使用aioamqp与rabbitmq服务器进行通信，接下来详细介绍aioamqp的基本使用。

There are two principal objects when using aioamqp:

- The protocol object, used to begin a connection to aioamqp,
- The channel object, used when creating a new channel to effectively use an AMQP channel

我们在使用aioamqp时，主要会涉及两个对象protocol对象是在我们使用aioamqp.connect()方法后拿到的对象，在拿到protocol对象后可以使用protocol.channel()方法申请一个channel对象，在拿到channel对象后就能实现基本的创建交换机，创建队列，交换机绑定队列等操作

```python
import asyncio
import aioamqp

async def connect():
    try:
        transport, protocol = await aioamqp.connect()  # use default parameters
    except aioamqp.AmqpClosedConnection:
        print("closed connections")
        return

    print("connected !")
    await asyncio.sleep(1)

    print("close connection")
    await protocol.close()
    transport.close()

asyncio.get_event_loop().run_until_complete(connect())
```

在拿到protocol对象后，使用protocol.channel方法获得channel对象

```python
channel = await protocol.channel()
```

当你想produce一些消息，你可以申明一个队列，然后发布你的消息进入这个队列（这里使用的是默认交换机）

```python
await channel.queue_declare("my_queue")
await channel.publish("aioamqp hello", '', "my_queue")
```

当你消费一条消息时，你首先要定义消费者，即callback函数，在aioamqp中定义callback函数是这样定义的。

```python
import asyncio
import aioamqp

async def callback(channel, body, envelope, properties):
    print(body)

channel = await protocol.channel()
await channel.basic_consume(callback, queue_name="my_queue")
```

其中channel是channel对象，body是消息的主体，envelope对象封装了消息队列的一组amqp参数

```python
consumer_tag    #你的consumer的id
delivery_tag    #你确认消息时需要用到的标识
exchange_name   #交换机
routing_key     #路由key
is_redeliver    #是否重新投递
```

properties是属性对象，主要包含了下面成员：

```python
content_type
content_encoding
headers
delivery_mode
priority
correlation_id
reply_to
expiration
message_id
timestamp
message_type
user_id
app_id
cluster_id
```

其中使用较多的函数为队列的声明，交换机的声明，绑定队列和交换机

Channel.queue_declare(queue_name, passive, durable, exclusive, auto_delete, no_wait, arguments, timeout) → dict

Channel.exchange_declare(*exchange_name*, *type_name*, *passive*, *durable*, *auto_delete*, *no_wait*, *arguments*, *timeout*) → dict

Channel.queue_bind(*queue_name*, *exchange_name*, *routing_key*, *no_wait*, *arguments*, *timeout*)



