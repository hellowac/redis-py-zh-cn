RedisTimeSeries 命令
=============================

这些是用于与 `RedisTimeSeries 模块 <https://redistimeseries.io>`_ 交互的命令。下面是一个简短的示例，以及命令本身的文档。

**创建一个保留时间为 5 秒的时间序列对象**

.. code-block:: python

    import redis
    r = redis.Redis()
    r.ts().create(2, retention_msecs=5000)

.. automodule:: redis.commands.timeseries.commands
    :members: TimeSeriesCommands