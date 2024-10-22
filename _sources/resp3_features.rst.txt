RESP 3 功能(RESP 3 Features)
==============================

从 5.0 版本开始，redis-py 支持 `RESP 3 标准 <https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md>`_。实际上，这意味着使用 RESP 3 的客户端将更快、更高效，因为在客户端中发生的类型转换更少。这也意味着新的响应类型，如双精度数、简单字符串、映射和布尔值变得可用。

连接(Connecting)
----------------------

启用 RESP 3 与在 redis-py 中进行其他连接并无不同。在所有情况下，都必须通过设置 `protocol=3` 来扩展连接类型。以下是一些启用 RESP 3 连接的基础示例。

使用标准连接，但指定 RESP 3：

.. code:: python

    >>> import redis
    >>> r = redis.Redis(host='localhost', port=6379, protocol=3)
    >>> r.ping()

或者使用 URL 方案：

.. code:: python

    >>> import redis
    >>> r = redis.from_url("redis://localhost:6379?protocol=3")
    >>> r.ping()

使用异步连接并指定 RESP 3：

.. code:: python

    >>> import redis.asyncio as redis
    >>> r = redis.Redis(host='localhost', port=6379, protocol=3)
    >>> await r.ping()

异步客户端使用 URL 方案：

.. code:: python

    >>> import redis.asyncio as Redis
    >>> r = redis.from_url("redis://localhost:6379?protocol=3")
    >>> await r.ping()

连接到启用 RESP 3 的 OSS Redis 集群：

.. code:: python

    >>> from redis.cluster import RedisCluster, ClusterNode
    >>> r = RedisCluster(startup_nodes=[ClusterNode('localhost', 6379), ClusterNode('localhost', 6380)], protocol=3)
    >>> r.ping()

推送通知(Push notifications)  
------------------------------------

推送通知是 redis 发送带外数据的一种方式。RESP 3 协议包括一个 `push 类型 <https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#push-type>`_，它允许我们的客户端拦截这些带外消息。默认情况下，客户端将记录简单的消息，但 redis-py 提供了使用自定义函数处理器的能力。

这意味着如果你想在接收到特定的推送通知时执行某些操作，可以在连接期间指定一个函数，如下所示：

.. code:: python

    >> from redis import Redis
    >>
    >> def our_func(message):
    >>    if message.find("This special thing happened"):
    >>        raise IOError("This was the message: \n" + message)
    >>
    >> r = Redis(protocol=3)
    >> p = r.pubsub(push_handler_func=our_func)

在上面的示例中，当收到推送通知时，如果出现特定文本，不是记录消息，而是抛出一个 IOError 。这例子展示了如何开始实现自定义消息处理程序。

客户端缓存(Client-side caching)
--------------------------------------

客户端缓存是一种用于创建高性能服务的技术。它利用应用程序服务器上的内存（通常与数据库节点分离）在应用程序端直接缓存一部分数据。有关更多信息，请查阅 `官方 Redis 文档 <https://redis.io/docs/latest/develop/use/client-side-caching/>`_ 。请注意，此功能仅在启用了 RESP3 协议的同步客户端中可用。支持独立模式、集群和哨兵客户端。

基本用法：

使用默认配置启用缓存：

.. code:: python

    >>> import redis
    >>> from redis.cache import CacheConfig
    >>> r = redis.Redis(host='localhost', port=6379, protocol=3, cache_config=CacheConfig())

相同的接口适用于 Redis 集群和哨兵。

使用自定义缓存实现启用缓存：

.. code:: python

    >>> import redis
    >>> from foo.bar import CacheImpl
    >>> r = redis.Redis(host='localhost', port=6379, protocol=3, cache=CacheImpl())

CacheImpl 应该实现 `redis.cache` 包中指定的 `CacheInterface`。

更全面的文档将很快在 `官方 Redis 文档 <https://redis.io/docs/latest/>`_ 中提供。
