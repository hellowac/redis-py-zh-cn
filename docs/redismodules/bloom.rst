RedisBloom 命令
====================

以下是与 `RedisBloom 模块 <https://redisbloom.io>`_ 交互的命令。下面是一个简要的示例，以及相关命令的文档。

**创建并添加到布隆过滤器(Create and add to a bloom filter)**

.. code-block:: python

    import redis
    r = redis.Redis()
    r.bf().create("bloom", 0.01, 1000)
    r.bf().add("bloom", "foo")

**创建并添加到布谷鸟过滤器(Create and add to a cuckoo filter)**

.. code-block:: python

    import redis
    r = redis.Redis()
    r.cf().create("cuckoo", 1000)
    r.cf().add("cuckoo", "filter")

**创建 Count-Min Sketch 并获取信息(Create Count-Min Sketch and get information)**

.. code-block:: python

    import redis
    r = redis.Redis()
    r.cms().initbydim("dim", 1000, 5)
    r.cms().incrby("dim", ["foo"], [5])
    r.cms().info("dim")

**创建一个 topk 列表，并访问结果(Create a topk list, and access the results)**

.. code-block:: python

    import redis
    r = redis.Redis()
    r.topk().reserve("mytopk", 3, 50, 4, 0.9)
    r.topk().info("mytopk")

.. automodule:: redis.commands.bf.commands
    :members: BFCommands, CFCommands, CMSCommands, TOPKCommands