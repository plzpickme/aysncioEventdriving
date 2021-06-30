# 第五节 event

event模块是理解事件驱动最核心的模块，其中event类和eventcenter类是最核心的两个类。

```python
class Event:
    """Event base.

    Attributes:
        name: Event name.
        exchange: Exchange name.
        queue: Queue name.
        routing_key: Routing key name.
        pre_fetch_count: How may message per fetched, default is `1`.
        data: Message content.
    """

    def __init__(self, name=None, exchange=None, queue=None, routing_key=None, pre_fetch_count=1, data=None):
        """Initialize."""
        self._name = name
        self._exchange = exchange
        self._queue = queue
        self._routing_key = routing_key
        self._pre_fetch_count = pre_fetch_count
        self._data = data
        self._callback = None  # Asynchronous callback function.

    @property
    def name(self):
        return self._name

    @property
    def exchange(self):
        return self._exchange

    @property
    def queue(self):
        return self._queue

    @property
    def routing_key(self):
        return self._routing_key

    @property
    def prefetch_count(self):
        return self._pre_fetch_count

    @property
    def data(self):
        return self._data

    def dumps(self):
        d = {
            "n": self.name,
            "d": self.data
        }
        s = json.dumps(d)
        b = zlib.compress(s.encode("utf8"))
        return b
    #压缩 name 和 data
    def loads(self, b):
        b = zlib.decompress(b)
        d = json.loads(b.decode("utf8"))
        self._name = d.get("n")
        self._data = d.get("d")
        return d
    #解压 name 和 data
    def parse(self):
        raise NotImplemented

    def subscribe(self, callback, multi=False):
        """Subscribe a event.

        Args:
            callback: Asynchronous callback function.
            multi: If subscribe multiple channels?
        """
        from aioquant import quant
        self._callback = callback
        SingleTask.run(quant.event_center.subscribe, self, self.callback, multi)

    def publish(self):
        """Publish a event."""
        from aioquant import quant
        SingleTask.run(quant.event_center.publish, self)

    async def callback(self, channel, body, envelope, properties):
        self._exchange = envelope.exchange_name
        self._routing_key = envelope.routing_key
        self.loads(body)
        o = self.parse()
        await self._callback(o)

    def __str__(self):
        info = "EVENT: name={n}, exchange={e}, queue={q}, routing_key={r}, data={d}".format(
            e=self.exchange, q=self.queue, r=self.routing_key, n=self.name, d=self.data)
        return info

    def __repr__(self):
        return str(self)
```
我们将event类中的维护字段一一解析
```python
				self._name = name
        self._exchange = exchange
        self._queue = queue
        self._routing_key = routing_key
        self._pre_fetch_count = pre_fetch_count
        self._data = data
        self._callback = None  # Asynchronous callback function.
```

name为事件的命名，exchange为该event应该分属的交换机，通常情况下根据数据类型确定，orderbookEvent分属orderbook交换机，queue通常情况下是用platform加上symbol进行命名，routing_key是用severid+platform+symbol进行命名（上面内容需要对amqp协议以及rabbitmq有了解，安装好rabbitmq之后在rabbimq的web管理工具中有docs入口，或者在第一章中的aioamqp介绍下有官方文档入口），pre_fetch_count是每次获取事件数量默认值为1，data存储数据，callback为该事件的回调函数，event类中维护的回调函数_callback是对传入的callback的根据amqp协议的标准化封装，即该事件的消费者。

```python
def dumps(self):
    d = {
        "n": self.name,
        "d": self.data
    }
    s = json.dumps(d)
    b = zlib.compress(s.encode("utf8"))
    return b
#压缩 name 和 data
def loads(self, b):
    b = zlib.decompress(b)
    d = json.loads(b.decode("utf8"))
    self._name = d.get("n")
    self._data = d.get("d")
    return d
```

dumps是对数据的压缩，loads提取压缩数据并且更新当前该event最新的数据
```python
def parse(self):
        raise NotImplemented
```

parse函数是需要子类实现的函数这个函数读取数据并返回相应的数据对象，比如orderbookEvent中这样实现

```python
def parse(self):
    orderbook = Orderbook().load_smart(self.data)
    return orderbook
```

在event中最重要的还是完成回调函数的订阅，以及事件的发布

```python
def subscribe(self, callback, multi=False):
    """Subscribe a event.

    Args:
        callback: Asynchronous callback function.
        multi: If subscribe multiple channels?
    """
    from aioquant import quant
    self._callback = callback
    SingleTask.run(quant.event_center.subscribe, self, self.callback, multi)

def publish(self):
    """Publish a event."""
    from aioquant import quant
    SingleTask.run(quant.event_center.publish, self)
```

在subscribe中我们要拿到我们在quant模块中初始化的event_center,并调用event_center中的subscribe方法，将callback订阅和消费自己的数据。而publish方法即是将自身数据通过event_center发布到消息队列中。

整个异步事件驱动交易回测框架最核心的部分便是EventCenter类，在这实现了如何和rabbitmq的交互，实现了对callback函数的绑定消费。

```python
class EventCenter:
    """Event center.
    """

    def __init__(self):
        self._host = config.rabbitmq.get("host", "localhost")
        self._port = config.rabbitmq.get("port", 5672)
        self._username = config.rabbitmq.get("username", "guest")
        self._password = config.rabbitmq.get("password", "guest")
        self._protocol = None
        self._channel = None  # Connection channel.
        self._connected = False  # If connect success.
        self._subscribers = []  # e.g. `[(event, callback, multi), ...]`
        self._event_handler = {}  # e.g. `{"exchange:routing_key": [callback_function, ...]}`

        # Register a loop run task to check TCP connection's healthy.
        LoopRunTask.register(self._check_connection, 10)

        # Create MQ connection.
        asyncio.get_event_loop().run_until_complete(self.connect())

    @async_method_locker("EventCenter.subscribe")
    async def subscribe(self, event: Event, callback=None, multi=False):
        """Subscribe a event.

        Args:
            event: Event type.
            callback: Asynchronous callback.
            multi: If subscribe multiple channel(routing_key) ?
        """
        logger.info("NAME:", event.name, "EXCHANGE:", event.exchange, "QUEUE:", event.queue, "ROUTING_KEY:",
                    event.routing_key, caller=self)
        self._subscribers.append((event, callback, multi))

    async def publish(self, event):
        """Publish a event.

        Args:
            event: A event to publish.
        """
        if not self._connected:
            logger.warn("RabbitMQ not ready right now!", caller=self)
            return
        data = event.dumps()
        await self._channel.basic_publish(payload=data, exchange_name=event.exchange, routing_key=event.routing_key)

    async def connect(self, reconnect=False):
        """Connect to RabbitMQ server and create default exchange.

        Args:
            reconnect: If this invoke is a re-connection ?
        """
        logger.info("host:", self._host, "port:", self._port, caller=self)
        if self._connected:
            return

        # Create a connection.
        try:
            transport, protocol = await aioamqp.connect(host=self._host, port=self._port, login=self._username,
                                                        password=self._password, login_method="PLAIN")
        except Exception as e:
            logger.error("connection error:", e, caller=self)
            return
        finally:
            if self._connected:
                return
        channel = await protocol.channel()
        self._protocol = protocol
        self._channel = channel
        self._connected = True
        logger.info("Rabbitmq initialize success!", caller=self)

        # Create default exchanges.
        exchanges = ["Orderbook", "Kline"]
        for name in exchanges:
            await self._channel.exchange_declare(exchange_name=name, type_name="fanout")
        logger.debug("create default exchanges success!", caller=self)

        if reconnect:
            self._bind_and_consume()
        else:
            # Maybe we should waiting for all modules to be initialized successfully.
            asyncio.get_event_loop().call_later(5, self._bind_and_consume)

    def _bind_and_consume(self):
        async def do_them():
            for event, callback, multi in self._subscribers:
                await self._initialize(event, callback, multi)
        SingleTask.run(do_them)

    async def _initialize(self, event: Event, callback=None, multi=False):
        if event.queue:
            await self._channel.queue_declare(queue_name=event.queue, auto_delete=True)
            queue_name = event.queue
        else:
            result = await self._channel.queue_declare(exclusive=True)
            queue_name = result["queue"]
        await self._channel.queue_bind(queue_name=queue_name, exchange_name=event.exchange,
                                       routing_key=event.routing_key)
        await self._channel.basic_qos(prefetch_count=event.prefetch_count)
        if callback:
            if multi:
                await self._channel.basic_consume(callback=callback, queue_name=queue_name, no_ack=True)
                logger.info("multi message queue:", queue_name, caller=self)
            else:
                self._add_event_handler(event, callback)
                await self._channel.basic_consume(self._on_consume_event_msg, queue_name=queue_name)
                logger.info("queue:", queue_name, caller=self)


    async def _on_consume_event_msg(self, channel, body, envelope, properties):
        try:
            key = "{exchange}:{routing_key}".format(exchange=envelope.exchange_name, routing_key=envelope.routing_key)
            funcs = self._event_handler[key]
            for func in funcs:
                SingleTask.run(func, channel, body, envelope, properties)
        except:
            logger.error("event handle error! body:", body, caller=self)
            return
        finally:
            await self._channel.basic_client_ack(delivery_tag=envelope.delivery_tag)  # response ack

    def _add_event_handler(self, event: Event, callback):
        key = "{exchange}:{routing_key}".format(exchange=event.exchange, routing_key=event.routing_key)
        if key in self._event_handler:
            self._event_handler[key].append(callback)
        else:
            self._event_handler[key] = [callback]
        logger.debug("event handlers:", self._event_handler.keys(), caller=self)

    async def _check_connection(self, *args, **kwargs):
        if self._connected and self._channel and self._channel.is_open:
            return
        logger.error("CONNECTION LOSE! START RECONNECT RIGHT NOW!", caller=self)
        self._connected = False
        self._protocol = None
        self._channel = None
        self._event_handler = {}
        SingleTask.run(self.connect, reconnect=True)
```

我们首先来看看Event_center类维护的字段以及其构造器所完成的工作
```python
				self._host = config.rabbitmq.get("host", "localhost")
        self._port = config.rabbitmq.get("port", 5672)
        self._username = config.rabbitmq.get("username", "guest")
        self._password = config.rabbitmq.get("password", "guest")
        self._protocol = None
        self._channel = None  # Connection channel.
        self._connected = False  # If connect success.
        self._subscribers = []  # e.g. `[(event, callback, multi), ...]`
        self._event_handler = {}  # e.g. `{"exchange:routing_key": [callback_function, ...]}`

        # Register a loop run task to check TCP connection's healthy.
        LoopRunTask.register(self._check_connection, 10)
    
        # Create MQ connection.
      asyncio.get_event_loop().run_until_complete(self.connect())
```

首先从配置文件中拿到了rabbitmq的host和port，以及username和password，这些字段都是我们在连接rabbitmq服务时需要提供的参数，protocol和channel是我们在连接服务器成功之后，返回的可操作对象，需要维护用于提供给后面的操作使用。subscribes是一个保存事件与callback函数消费关系的元组，这个元祖可以通过subscibe函数事件和callback函数对进行添加，在multi表示该callback函数是否订阅了多个交易对或者多个平台，在通过aioamqp进行绑定消费时，我们是通过subscribes这个元组获取我们需要绑定的队列和以及相应callback函数，event_handler是一个维护一个队列存在多个消费者字典，exchange:routing_key指定唯一队列，后面紧跟一个元组保存了该队列所有的callback function。

接下来构造器注册了一个循环任务每隔10秒检查与rabbitmq服务器的连接，最后，构造器去尝试连接rabbitmq服务器。

```python
async def connect(self, reconnect=False):
    """Connect to RabbitMQ server and create default exchange.

    Args:
        reconnect: If this invoke is a re-connection ?
    """
    logger.info("host:", self._host, "port:", self._port, caller=self)
    if self._connected:
        return

    # Create a connection.
    try:
        transport, protocol = await aioamqp.connect(host=self._host, port=self._port, login=self._username,
                                                    password=self._password, login_method="PLAIN")
    except Exception as e:
        logger.error("connection error:", e, caller=self)
        return
    finally:
        if self._connected:
            return
    channel = await protocol.channel()
    self._protocol = protocol
    self._channel = channel
    self._connected = True
    logger.info("Rabbitmq initialize success!", caller=self)

    # Create default exchanges.
    exchanges = ["Orderbook", "Kline"]
    for name in exchanges:
        await self._channel.exchange_declare(exchange_name=name, type_name="fanout")
    logger.debug("create default exchanges success!", caller=self)

    if reconnect:
        self._bind_and_consume()
    else:
        # Maybe we should waiting for all modules to be initialized successfully.
        asyncio.get_event_loop().call_later(5, self._bind_and_consume)
```

connect函数完成了以下几件事：
```python
transport, protocol = await aioamqp.connect(host=self._host, port=self._port, login=self._username,
                                                    password=self._password, login_method="PLAIN")

```
```python
channel = await protocol.channel()
```

拿到protocol和channel对象，表示与rabbitmq服务器连接成功，并保存维护两个对象。
```python
		#Create default exchanges.
		exchanges = ["Orderbook", "Kline"，"trade",]
    for name in exchanges:
        await 			  	                  				self._channel.exchange_declare(exchange_name=name, type_name="topic")     
    logger.debug("create default exchanges success!", caller=self)
```

创建默认的交换机，默认的交换机根据数据类型进行创建，有orderbook类型，kline类型，trade类型。
```python
if reconnect:
        self._bind_and_consume()
else:
        # Maybe we should waiting for all modules to be initialized successfully.
        asyncio.get_event_loop().call_later(5, self._bind_and_consume
```

最后我们在5s后进行callback消息队列的消费绑定，延迟5s的原因是，当我们启动aioquant框架时，AIOQuant类在初始化时就创建了一个eventcenter对象，此时我们时通过subscribes这个元组进行绑定，由于在初始化时就进行connect操作，subscribes中并没有添加任何信息，所以我们需要等待一小段时间，等策略的callback函数与对应event添加到subscribes元组中，如果是reconnect当然就不需要等待。

```python
@async_method_locker("EventCenter.subscribe")
async def subscribe(self, event: Event, callback=None, multi=False):
    """Subscribe a event.

    Args:
        event: Event type.
        callback: Asynchronous callback.
        multi: If subscribe multiple channel(routing_key) ?
    """
    logger.info("NAME:", event.name, "EXCHANGE:", event.exchange, "QUEUE:", event.queue, "ROUTING_KEY:",
                event.routing_key, caller=self)
    self._subscribers.append((event, callback, multi))

async def publish(self, event):
    """Publish a event.

    Args:
        event: A event to publish.
    """
    if not self._connected:
        logger.warn("RabbitMQ not ready right now!", caller=self)
        return
    data = event.dumps()
    await self._channel.basic_publish(payload=data, exchange_name=event.exchange, routing_key=event.routing_key)
```

接下来我们看下在事件中心如何订阅和发布，订阅过程非常简单，将event和对应的消费者callback函数添加到subscribes元组中，不过需要注意的是，这里用到了异步锁，因为在程序运行初期可能有多个event和对应callback函数同时访问subscribes元组，防止出错，这里使用锁机制，同时只有一个event和对应callback能访问subscribes元组，进行添加操作。publish函数不做过多介绍，完成对数据的压缩然后根据传入的event的exchange和routingkey进行发布。

接下来我们来看将event和对应的callback函数加入subscribes元组中后，如何使用subscribes中的数据在rabbitmq中进行绑定消费操作。

```python
def _bind_and_consume(self):
    async def do_them():
        for event, callback, multi in self._subscribers:
            await self._initialize(event, callback, multi)
    SingleTask.run(do_them)

async def _initialize(self, event: Event, callback=None, multi=False):
    if event.queue:
        await self._channel.queue_declare(queue_name=event.queue, auto_delete=True)
        queue_name = event.queue
    else:
        result = await self._channel.queue_declare(exclusive=True)
        queue_name = result["queue"]
    await self._channel.queue_bind(queue_name=queue_name, exchange_name=event.exchange,
                                   routing_key=event.routing_key)
    await self._channel.basic_qos(prefetch_count=event.prefetch_count)
    if callback:
        if multi:
            await self._channel.basic_consume(callback=callback, queue_name=queue_name, no_ack=True)
            logger.info("multi message queue:", queue_name, caller=self)
        else:
            self._add_event_handler(event, callback)
            await self._channel.basic_consume(self._on_consume_event_msg, queue_name=queue_name)
            logger.info("queue:", queue_name, caller=self)


async def _on_consume_event_msg(self, channel, body, envelope, properties):
    try:
        key = "{exchange}:{routing_key}".format(exchange=envelope.exchange_name, routing_key=envelope.routing_key)
        funcs = self._event_handler[key]
        for func in funcs:
            SingleTask.run(func, channel, body, envelope, properties)
    except:
        logger.error("event handle error! body:", body, caller=self)
        return
    finally:
        await self._channel.basic_client_ack(delivery_tag=envelope.delivery_tag)  # response ack

def _add_event_handler(self, event: Event, callback):
    key = "{exchange}:{routing_key}".format(exchange=event.exchange, routing_key=event.routing_key)
    if key in self._event_handler:
        self._event_handler[key].append(callback)
    else:
        self._event_handler[key] = [callback]
    logger.debug("event handlers:", self._event_handler.keys(), caller=self)
```

在bind_consume函数中对subscribes元组中的每个item进行initialize操作，接下来我们看initialize函数，如果event中的queue字段值不为空，就在rabbitmq中声明一个event.queue的队列，如果event中的queue字段值为空，就声明一个默认的队列，随后我们声明的队列与event中的exchange进行绑定，并设定每次从消息队列中取数据的粒度，默认为1每次取一个，随后我们根据该event是否可能存在有多个消费者的情况选择使用 自身callback函数还是封装之后的_on_consume_event_msg函数，其中 _on_consume_event_msg函数作为rabbitmq的回调函数，在 _on_consume_event_msg中将字典event_handler中该队列的所有callback函数依次调用创建task， _add_event_handler即将该队列对应的一个callback加入到event_handler这个字典中。

