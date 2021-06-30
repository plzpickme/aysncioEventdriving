# 第三节 heartbeat

heartbeat模块作为整个异步事件驱动数字货币回测交易框架的基础服务，主要提供给框架两个功能，第一个功能是提供心跳功能即当主循环启动后就根据设定的时间间隔在控制台打印当前已经启动时间，目的是当系统在后台运行时也能被观察到其运行状态。同时heartbeat同时为框架提供实现循环任务的能力，当在heartbeat模块中被注册成为一个循环任务，heartbeat将每隔一个任务循环时间间隔就在事件主循环中创建一个该任务。

```python
class HeartBeat(object):
    """Server heartbeat.
    """

    def __init__(self):
        self._count = 0  # Heartbeat count.
        self._interval = 1  # Heartbeat interval(second).
        self._print_interval = config.heartbeat.get("interval", 0)  # Printf heartbeat information interval(second).
        self._tasks = {}  # Loop run tasks with heartbeat service. `{task_id: {...}}`
```

heartbeat类中维护了4个字段，count是目前心态次数的总计数，interval是每个心跳之间的间隔时间，通常为1sec，printinterval是每隔多少个心跳打印一次信息到控制台，tasks为在heartbeat中注册的循环任务列表，task中的任务会根据它们自身的是任务间隔，被循环创建。

```python
def register(self, func, interval=1, *args, **kwargs):
    """Register an asynchronous callback function.

    Args:
        func: Asynchronous callback function.
        interval: Loop callback interval(second), default is `1s`.

    Returns:
        task_id: Task id.
    """
    t = {
        "func": func,
        "interval": interval,
        "args": args,
        "kwargs": kwargs
    }
    task_id = tools.get_uuid1()
    self._tasks[task_id] = t
    return task_id

def unregister(self, task_id):
    """Unregister a task.

    Args:
        task_id: Task id.
    """
    if task_id in self._tasks:
        self._tasks.pop(task_id)
```

register函数是heartbeat模块中为注册循环任务提供的函数，unrigester函数即将注册在循环任务列表中的任务解除注册。当调用regester函数后，函数将随机获取taskid，获取到id后将任务（包含任务名，循环调用间隔，参数）和任务id一起加入tasks 中。

```python
def ticker(self):
    """Loop run ticker per self._interval.
    """
    self._count += 1

    if self._print_interval > 0:
        if self._count % self._print_interval == 0:
            logger.info("do server heartbeat, count:", self._count, caller=self)

    # Later call next ticker.
    asyncio.get_event_loop().call_later(self._interval, self.ticker)

    # Exec tasks.
    for task_id, task in self._tasks.items():
        interval = task["interval"]
        if self._count % interval != 0:
            continue
        func = task["func"]
        args = task["args"]
        kwargs = task["kwargs"]
        kwargs["task_id"] = task_id
        kwargs["heart_beat_count"] = self._count
        asyncio.get_event_loop().create_task(func(*args, **kwargs))
```

ticker函数是heartbeat的核心函数，每一次调用count会加1，如果调用次数是printinterval是整倍数，ticker将打印消息到控制台上。然后ticker函数将在一个interval之后调用自己asyncio.get_event_loop().call_later(self._interval, self.ticker)实现心跳功能。随后将从tasks中逐一对每一个task进行访问，判断是否已经距离任务上一次调用相隔了设定的interval，若已经到了设定的interval即在主循环中创建task，调用任务。
