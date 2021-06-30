# 第六节 tasks

task提供了两种任务类型供我们选择，looptask为循环任务，其底层是调用heartbeat模块实现，singletask是直接调用
```python
asyncio.get_event_loop().create_task(func(*args, **kwargs))
```
```python
asyncio.get_event_loop().call_later(delay, func, *args)
```

当任务为asyncio异步函数时，会被创建为coro协程对象时，使用creat_task函数，若为一般函数时使用call_later函数调用。同时singletask提供call_later和run两种方式，run为立即执行而call_later运行delay执行。

```python
def run(cls, func, *args, **kwargs):
    """Create a coroutine and execute immediately.

    Args:
        func: Asynchronous callback function.
    """
    asyncio.get_event_loop().create_task(func(*args, **kwargs))

@classmethod
def call_later(cls, func, delay=0, *args, **kwargs):
    """Create a coroutine and delay execute, delay time is seconds, default delay time is 0s.

    Args:
        func: Asynchronous callback function.
        delay: Delay time is seconds, default delay time is 0, you can assign a float e.g. 0.5, 2.3, 5.1 ...
    """
    if not inspect.iscoroutinefunction(func):
        asyncio.get_event_loop().call_later(delay, func, *args)
    else:
        def foo(f, *args, **kwargs):
            asyncio.get_event_loop().create_task(f(*args, **kwargs))
        asyncio.get_event_loop().call_later(delay, foo, func, *args)
```

而looptask是对heartbeat模块中循环任务注册的封装

```python
class LoopRunTask(object):
    """Loop run task.
    """

    @classmethod
    def register(cls, func, interval=1, *args, **kwargs):
        """Register a loop run.

        Args:
            func: Asynchronous callback function.
            interval: execute interval time(seconds), default is 1s.

        Returns:
            task_id: Task id.
        """
        task_id = heartbeat.register(func, interval, *args, **kwargs)
        return task_id

    @classmethod
    def unregister(cls, task_id):
        """Unregister a loop run task.

        Args:
            task_id: Task id.
        """
        heartbeat.unregister(task_id)
```
