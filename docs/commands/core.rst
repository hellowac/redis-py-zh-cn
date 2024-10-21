核心命令(Core Commands)
===========================

以下函数可用于复制其对应的 `Redis 命令 <https://redis.io/commands>`_ 。通常，它们可以作为你 Redis 连接中的函数使用。最简单的示例如下：

在 Redis 中获取和设置数据::

   import redis
   r = redis.Redis(decode_responses=True)
   r.set('mykey', 'thevalueofmykey')
   r.get('mykey')

.. autoclass:: redis.commands.core.CoreCommands
   :inherited-members: