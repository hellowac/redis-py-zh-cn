Lua脚本(Lua Scripting)
=============================

`Lua Scripting <#lua-lua-scripting-in-default-connections>`__ \|
`Pipelines <#pipelines>`__ \| `Cluster mode <#cluster-mode>`__

--------------

默认连接中的 Lua 脚本(Lua Scripting in default connections)
------------------------------------------------------------------------

redis-py 支持 EVAL、EVALSHA 和 SCRIPT 命令。然而，在实际使用中，这些命令有许多边缘情况，使得它们使用起来比较繁琐。因此，redis-py 提供了一个 Script 对象，使得脚本更加易于使用。（RedisClusters 对脚本的支持有限。）

要创建一个 Script 实例，可以在客户端实例上使用 :py:func:`~redis.commands.core.CoreCommands.register_script` 函数，传递 Lua 代码作为第一个参数。:py:func:`~redis.commands.core.CoreCommands.register_script` 会返回一个 Script 实例，您可以在整个代码中使用它。

以下是一个简单的 Lua 脚本，它接受两个参数：一个键名和一个乘数值。该脚本获取存储在键中的值，将其与乘数值相乘并返回结果。

.. code:: python

   >>> r = redis.Redis()
   >>> lua = """
   ... local value = redis.call('GET', KEYS[1])
   ... value = tonumber(value)
   ... return value * ARGV[1]"""
   >>> multiply = r.register_script(lua)

`multiply` 现在是一个 Script 实例，通过像函数一样调用它来执行。Script 实例接受以下可选参数：

-  **keys**: 脚本将访问的键名列表。这将成为 Lua 中的 KEYS 列表。
-  **args**: 参数值列表。这将成为 Lua 中的 ARGV 列表。
-  **client**: 一个 redis-py 的 Client 或 Pipeline 实例，将用于调用脚本。如果未指定 client ，将使用最初创建 Script 实例的客户端（即调用 `register_script` 的客户端）。

继续前面的示例：

.. code:: python

   >>> r.set('foo', 2)
   >>> multiply(keys=['foo'], args=[5])
   10

键 'foo' 的值被设置为 2。当调用 `multiply` 时，'foo' 键与乘数值 5 一起传递给脚本。Lua 执行该脚本并返回结果 10。

Script 实例可以使用不同的客户端实例执行，即使该客户端指向的是完全不同的 Redis 服务器。

.. code:: python

   >>> r2 = redis.Redis('redis2.example.com')
   >>> r2.set('foo', 3)
   >>> multiply(keys=['foo'], args=[5], client=r2)
   15

Script 对象确保 Lua 脚本被加载到 Redis 的脚本缓存中。如果出现 :py:exc:`~redis.exceptions.NoScriptError` 错误，它会重新加载脚本并重试执行。

管道 (Pipelines)
-----------------

脚本对象也可以在管道 (pipeline) 中使用。在调用脚本时，管道实例(pipeline instance) 应该作为 `client` 参数传递。在管道执行之前，会确保脚本已注册到 Redis 的脚本缓存中。

.. code:: python

   >>> pipe = r.pipeline()
   >>> pipe.set('foo', 5)
   >>> multiply(keys=['foo'], args=[5], client=pipe)
   >>> pipe.execute()
   [True, 25]

集群模式 (Cluster Mode)
------------------------

集群模式对 Lua 脚本的支持有限。

以下命令在集群模式下受支持，但有一些限制：

- ``EVAL`` 和 ``EVALSHA``: 根据键将命令发送到相关节点（即在 ``EVAL "<script>" num_keys key_1 ... key_n ...`` 中）。这些键 **必须** 全部位于同一个节点上。如果脚本不需要任何键， **命令将被发送到一个随机的（主）节点** 。
- ``SCRIPT EXISTS``: 命令会发送到所有主节点。结果是一个布尔值列表，对应于输入的 SHA 哈希值。每个布尔值表示 “脚本是否存在于每个节点上” 的逻辑与 (AND)。换句话说，只有当脚本在所有节点上都存在时，布尔值才为 True。
- ``SCRIPT FLUSH``: 命令会发送到所有主节点。结果是所有节点响应的布尔值的逻辑与 (AND)。
- ``SCRIPT LOAD``: 命令会发送到所有主节点。结果是 SHA1 摘要。

以下命令 **不受支持** ：

- ``EVAL_RO``
- ``EVALSHA_RO``

在集群模式中， **不支持** 在管道中使用脚本。
