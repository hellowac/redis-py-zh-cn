连接到 Redis (Connecting to Redis)
######################################


通用客户端(Generic Client)
****************************

这是用于直接连接到标准 Redis 节点的客户端。

.. autoclass:: redis.Redis
   :members:


哨兵客户端(Sentinel Client)
******************************

Redis `Sentinel <https://redis.io/topics/sentinel>`_ 提供 Redis 的高可用性。有些命令只能在以哨兵模式运行的 Redis 节点上执行。连接到这些节点并对它们执行命令需要使用哨兵连接。

连接示例（假设 Redis 在下面列出的端口上存在）：

   >>> from redis import Sentinel
   >>> sentinel = Sentinel([('localhost', 26379)], socket_timeout=0.1)
   >>> sentinel.discover_master('mymaster')
   ('127.0.0.1', 6379)
   >>> sentinel.discover_slaves('mymaster')
   [('127.0.0.1', 6380)]

哨兵(Sentinel)
================
.. autoclass:: redis.sentinel.Sentinel
    :members:

哨兵连接池(SentinelConnectionPool)
============================================
.. autoclass:: redis.sentinel.SentinelConnectionPool
    :members:


集群客户端(Cluster Client)
****************************

此客户端用于连接到 Redis 集群。

RedisCluster
============
.. autoclass:: redis.cluster.RedisCluster
    :members:

ClusterNode
===========
.. autoclass:: redis.cluster.ClusterNode
    :members:


异步客户端(Async Client)
************************************

完整示例见： `这里 <examples/asyncio_examples.html>`__

此客户端用于异步与 Redis 通信。

.. autoclass:: redis.asyncio.client.Redis
    :members:


异步集群客户端
********************

RedisCluster (异步)
====================
.. autoclass:: redis.asyncio.cluster.RedisCluster
    :members:
    :member-order: bysource

ClusterNode (异步)
===================
.. autoclass:: redis.asyncio.cluster.ClusterNode
    :members:
    :member-order: bysource

ClusterPipeline (异步)
=======================
.. autoclass:: redis.asyncio.cluster.ClusterPipeline
    :members: execute_command, execute
    :member-order: bysource


连接(Connection)
********************

完整示例见： `这里 <examples/connection_examples.html>`__

连接(Connection)
====================
.. autoclass:: redis.connection.Connection
    :members:

连接 (异步)(Connection (Async))
====================================
.. autoclass:: redis.asyncio.connection.Connection
    :members:


连接池(Connection Pools)
********************************

完整示例见： `这里 <examples/connection_examples.html>`__

ConnectionPool
==============
.. autoclass:: redis.connection.ConnectionPool
    :members:

ConnectionPool (异步)
======================
.. autoclass:: redis.asyncio.connection.ConnectionPool
    :members: