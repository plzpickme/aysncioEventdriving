# 第一节 main

```python
from aioquant import quant
```

在引入模块时，会执行模块中的__ init __.py文件，在aioquant模块的init.py文件中有`quant = AIOQuant()`即初始化创建了一个AIOQuant对象quant

```python
if __name__ == "__main__":
    config_file = sys.argv[1]
    quant.start(config_file, initialize)  
```

[quant.shart()](./quant.md) 传入两个参数：        config_file 为配置文件的相对路径    initialize为初始化函数

```python
def initialize():
    from strategy.strategy import MyStrategy
    MyStrategy()
```

initialize（）初始化函数  初始化函数做了两件事： 引入了mystrategy类并创建了一个mystrategy实例对象（调用了类的构造器函数）

