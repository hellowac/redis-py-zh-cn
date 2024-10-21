RedisJSON 命令
========================

这些是与 `RedisJSON 模块 <https://redisjson.io>`_ 交互的命令。下面是一个简短的示例，以及命令本身的文档。

**创建一个 json 对象**

.. code-block:: python

    import redis
    r = redis.Redis()
    r.json().set("mykey", ".", {"hello": "world", "i am": ["a", "json", "object!"]})

有关如何结合搜索和 json 的示例可在 `此处 <examples/search_json_examples.html>`_ 找到。

.. automodule:: redis.commands.json.commands
    :members: JSONCommands