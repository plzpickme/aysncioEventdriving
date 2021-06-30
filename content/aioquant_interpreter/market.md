# 第四节 market

market模块和envent模块是我们理解整体框架的核心以及最重要的部分，涉及事件驱动的核心思想和架构。

首先我们来看market模块，market模块中有4个类分别是orderbook类，kline类，和trade类，看下其中一个orderbook类，其他类类似。

```python
class Orderbook:
    """Orderbook object.

    Args:
        platform: Exchange platform name, e.g. `binance` / `bitmex`.
        symbol: Trade pair name, e.g. `ETH/BTC`.
        asks: Asks list, e.g. `[[price, quantity], [...], ...]`
        bids: Bids list, e.g. `[[price, quantity], [...], ...]`
        timestamp: Update time, millisecond.
    """

    def __init__(self, platform=None, symbol=None, asks=None, bids=None, timestamp=None):
        """Initialize."""
        self.platform = platform
        self.symbol = symbol
        self.asks = asks
        self.bids = bids
        self.timestamp = timestamp

    @property
    def data(self):
        d = {
            "platform": self.platform,
            "symbol": self.symbol,
            "asks": self.asks,
            "bids": self.bids,
            "timestamp": self.timestamp
        }
        return d

    @property
    def smart(self):
        d = {
            "p": self.platform,
            "s": self.symbol,
            "a": self.asks,
            "b": self.bids,
            "t": self.timestamp
        }
        return d

    def load_smart(self, d):
        self.platform = d["p"]
        self.symbol = d["s"]
        self.asks = d["a"]
        self.bids = d["b"]
        self.timestamp = d["t"]
        return self

    def __str__(self):
        info = json.dumps(self.data)
        return info

    def __repr__(self):
        return str(self)
```

这三个类并没有实际参与到系统架构中，它们最重要的功能还是提供归一化的数据保存类型，这三个类分别代表了orderbook数据，kline数据，trade数据标准化数据保持类型，主要函数仅是get和set类方法，这里就不做过多介绍。

```python
# 订阅行情
Market(const.MARKET_TYPE_ORDERBOOK, const.BINANCE, self.symbol, self.on_event_orderbook_update)
```

我们看到demo_strategy中核心代码即为创建了一个Market对象，其中指定了market_type为orderbook类型，平台为binance，symbol交易对，和最重要的orderbook更新的回调callback函数，即该函数是当新的orderbook数据被送入程序后，on_event_orderbook_update函数会被调用去处理新的orderbook数据，接下来我们深入maket和event模块中，探究on_event_orderbook_update函数是如何被调用的。

```python
class Market:
    """Subscribe Market.

    Args:
        market_type: Market data type,
            MARKET_TYPE_TRADE = "trade"
            MARKET_TYPE_ORDERBOOK = "orderbook"
            MARKET_TYPE_KLINE = "kline"
            MARKET_TYPE_KLINE_5M = "kline_5m"
            MARKET_TYPE_KLINE_15M = "kline_15m"
        platform: Exchange platform name, e.g. `binance` / `bitmex`.
        symbol: Trade pair name, e.g. `ETH/BTC`.
        callback: Asynchronous callback function for market data update.
                e.g. async def on_event_kline_update(kline: Kline):
                        pass
    """

    def __init__(self, market_type, platform, symbol, callback):
        """Initialize."""
        if platform == "#" or symbol == "#":
            multi = True
        else:
            multi = False
        if market_type == const.MARKET_TYPE_ORDERBOOK:
            from aioquant.event import EventOrderbook
            EventOrderbook(Orderbook(platform, symbol)).subscribe(callback, multi)
        elif market_type == const.MARKET_TYPE_TRADE:
            from aioquant.event import EventTrade
            EventTrade(Trade(platform, symbol)).subscribe(callback, multi)
        elif market_type in [
            const.MARKET_TYPE_KLINE, const.MARKET_TYPE_KLINE_3M, const.MARKET_TYPE_KLINE_5M,
            const.MARKET_TYPE_KLINE_15M, const.MARKET_TYPE_KLINE_30M, const.MARKET_TYPE_KLINE_1H,
            const.MARKET_TYPE_KLINE_3H, const.MARKET_TYPE_KLINE_6H, const.MARKET_TYPE_KLINE_12H,
            const.MARKET_TYPE_KLINE_1D, const.MARKET_TYPE_KLINE_3D, const.MARKET_TYPE_KLINE_1W,
            const.MARKET_TYPE_KLINE_15D, const.MARKET_TYPE_KLINE_1MON, const.MARKET_TYPE_KLINE_1Y]:
            from aioquant.event import EventKline
            EventKline(Kline(platform, symbol, kline_type=market_type)).subscribe(callback, multi)
        else:
            logger.error("market_type error:", market_type, caller=self)
```

我们看到market类在构建对象时主要做了两件事，首先判定market的multi值，multi值是用于指明需要的市场数据是不不止一个交易所的一个交易对。然后根据传入参数market_type去创建相对应的数据类型Orderbook(platform, symbol)参数为传入的platform和symbol，获得的对象将被作为参数传入对应数据类型的事件类中，获得对应数据类型的事件类的对象，如果订阅的是orderbook数据即EventOrderbook(Orderbook(platform, symbol))，在获得对应数据类型的事件类的对象后，该对象调用了subscribe方法订阅了传入的回调函数callback作为event的消费者和订阅者，我们将在event模块中详细介绍是如何实现的。
