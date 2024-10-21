RediSearch 命令
======================

这些是与 `RediSearch 模块 <https://redisearch.io>`_ 交互的命令。下面是一个简短的示例，以及命令本身的文档。在下面的示例中，正在创建一个名为 *my_index* 的索引。当未指定索引名称时，将创建一个名为 *idx* 的索引。

**创建搜索索引并显示其信息**

.. code-block:: python

    import redis
    from redis.commands.search.field import TextField

    r = redis.Redis()
    index_name = "my_index"
    schema = (
        TextField("play", weight=5.0),
        TextField("ball"),
    )
    r.ft(index_name).create_index(schema)
    print(r.ft(index_name).info())


.. automodule:: redis.commands.search.commands
    :members: SearchCommands