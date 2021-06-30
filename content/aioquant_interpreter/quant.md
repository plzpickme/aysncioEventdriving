# 第二节 quant 

```python
class AIOQuant:
    """Asynchronous event I/O driven quantitative trading framework.
    """

    def __init__(self) -> None:
        self.loop = None
        self.event_center = None
```

AIOQuant对象维护了两个重要的两个对象，loop对象和event_center对象，loop对象为整个框架提供异步支持，event_center对象为消息中心，为框架提供消息服务。

```python
def start(self, config_file=None, entrance_func=None) -> None:
    """Start the event loop."""
    def keyboard_interrupt(s, f):
        print("KeyboardInterrupt (ID: {}) has been caught. Cleaning up...".format(s))
        self.stop()
    signal.signal(signal.SIGINT, keyboard_interrupt)

    self._initialize(config_file)
    if entrance_func:
        if inspect.iscoroutinefunction(entrance_func):
            self.loop.create_task(entrance_func())
        else:
            entrance_func()

    logger.info("start io loop ...", caller=self)
    self.loop.run_forever()

def stop(self) -> None:
    """Stop the event loop."""
    logger.info("stop io loop.", caller=self)
    # TODO: clean up running coroutine
    self.loop.stop()
```

AIOQuant主要有两个外部可访问的方法start（）和stop（），start（）方法是框架的启动方法，而stop（）函数是框架的终止方法。start方法主要进行初始化工作，由`self._initialize(config_file)`完成

```python
def _initialize(self, config_file):
    """Initialize."""
    self._get_event_loop()#获得事件循环，提供异步支持
    self._load_settings(config_file)#使用config_file文件对框架参数进行配置
    self._init_logger()#初始化日志系统
    self._init_event_center()#初始化事件中心
    self._do_heartbeat()#初始化服务心跳 标志服务处于运行状态
    return self
```

其中init_logger（）为初始化日志系统， do_heartbeat()是初始化心跳模块，作用是每秒打印当前系统已经运行时间，提示系统当前运行良好，后面源码就不在详细分析。

get_event_loop()拿到的loop对象，_init_event_center()拿到的事件中心对象和将被维护在AIOQuant内部，这也启动了异步事件框架最重要的两个结构。

stop()停止维护的loop对象，整个事件循环停止，框架停止运行。 
