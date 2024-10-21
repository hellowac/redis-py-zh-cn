集群(Clustering)
====================

redis-py 现在支持集群模式，并为 `Redis
Cluster <https://redis.io/topics/cluster-tutorial>`__ 提供客户端。

集群客户端基于 Grokzen 的 `redis-py-cluster <https://github.com/Grokzen/redis-py-cluster>`__，已添加错误修复，现在取代了该库。对这些更改的支持要归功于他的贡献。

要了解有关 Redis 集群的更多信息，请参阅 `Redis 集群规范 <https://redis.io/topics/cluster-spec>`__。

`创建集群 <#creating-clusters>`__ \| `指定目标节点 <#specifying-target-nodes>`__ \| `多键命令 <#multi-key-commands>`__ \| `已知的 发布/订阅(PubSub)
限制 <#known-pubsub-limitations>`__


创建集群(Creating clusters)
----------------------------------

将 redis-py 连接到 Redis 集群实例至少需要一个用于集群发现的节点。创建集群实例有多种方式：

- 使用 ``host`` 和 ``port`` 参数：

.. code:: python

   >>> from redis.cluster import RedisCluster as Redis
   >>> rc = Redis(host='localhost', port=6379)
   >>> print(rc.get_nodes())
       [[host=127.0.0.1,port=6379,name=127.0.0.1:6379,server_type=primary,redis_connection=Redis<ConnectionPool<Connection<host=127.0.0.1,port=6379,db=0>>>], [host=127.0.0.1,port=6378,name=127.0.0.1:6378,server_type=primary,redis_connection=Redis<ConnectionPool<Connection<host=127.0.0.1,port=6378,db=0>>>], [host=127.0.0.1,port=6377,name=127.0.0.1:6377,server_type=replica,redis_connection=Redis<ConnectionPool<Connection<host=127.0.0.1,port=6377,db=0>>>]]

- 使用 Redis URL 规范：

.. code:: python

   >>> from redis.cluster import RedisCluster as Redis
   >>> rc = Redis.from_url("redis://localhost:6379/0")

- 直接通过 ClusterNode 类：

.. code:: python

   >>> from redis.cluster import RedisCluster as Redis
   >>> from redis.cluster import ClusterNode
   >>> nodes = [ClusterNode('localhost', 6379), ClusterNode('localhost', 6378)]
   >>> rc = Redis(startup_nodes=nodes)

在创建 :py:class:`~redis.cluster.RedisCluster` 实例时，它会首先尝试与提供的启动节点之一建立连接。如果没有启动节点可达，将抛出 :py:class:`~redis.exceptions.RedisClusterException` 。在与集群节点之一建立连接后， :py:class:`~redis.cluster.RedisCluster` 实例将被初始化为三个缓存：一个槽缓存，它将 16384 个槽映射到处理它们的节点，一个节点缓存，包含集群所有节点的 :py:class:`~redis.cluster.ClusterNode` 对象（名称、主机、端口、Redis 连接），以及一个命令缓存，包含通过 Redis ``COMMAND`` 输出检索的所有服务器支持的命令。有关更多信息，请参见下方的 *RedisCluster 特定选项*。

:py:class:`~redis.cluster.RedisCluster` 实例可以直接用于执行 Redis 命令。当通过集群实例执行命令时，目标节点将被内部确定。当使用基于键的命令时，目标节点将是持有该键槽的节点。集群管理命令和其他非基于键的命令都有一个名为 ``target_nodes`` 的参数，您可以在此指定要在哪些节点上执行命令。在没有 ``target_nodes`` 的情况下，命令将执行在默认集群节点上。作为集群实例初始化的一部分，集群的默认节点是从集群的主节点中随机选择的，并将在重新初始化时更新。使用 :py:func:`~redis.cluster.RedisCluster.get_default_node`，您可以获取集群的默认节点，或者可以使用 ``set_default_node`` 方法更改它。

``target_nodes`` 参数在下一节 :ref:`指定目标节点 <specifying-target-nodes>` 中进行了说明。

.. code:: python

   >>> # target-nodes: 持有 'foo1' 键槽的节点
   >>> rc.set('foo1', 'bar1')
   >>> # target-nodes: 持有 'foo2' 键槽的节点
   >>> rc.set('foo2', 'bar2')
   >>> # target-nodes: 持有 'foo1' 键槽的节点
   >>> print(rc.get('foo1'))
   b'bar'
   >>> # target-node: 默认节点
   >>> print(rc.keys())
   [b'foo1']
   >>> # target-node: 默认节点
   >>> rc.ping()

.. _specifying-target-nodes:

指定目标节点(Specifying Target Nodes)
----------------------------------------------

如上所述，所有非基于键的 :py:class:`~redis.cluster.RedisCluster` 命令都接受关键字参数 ``target_nodes``，该参数指定应在哪个节点上执行命令。最佳实践是使用 :py:class:`~redis.cluster.RedisCluster` 类的节点标志: ``PRIMARIES`` 、``REPLICAS`` 、``ALL_NODES``、``RANDOM`` 来指定目标节点。当节点标志与命令一起传递时，它将被内部解析为相关节点。如果在执行命令期间集群的节点拓扑发生变化，客户端将能够根据新的拓扑再次解析节点标志，并尝试重试执行命令。

.. code:: python

   >>> from redis.cluster import RedisCluster as Redis
   >>> # 在集群的所有节点上运行 cluster-meet 命令
   >>> rc.cluster_meet('127.0.0.1', 6379, target_nodes=Redis.ALL_NODES)
   >>> # ping 所有副本
   >>> rc.ping(target_nodes=Redis.REPLICAS)
   >>> # ping 一个随机节点
   >>> rc.ping(target_nodes=Redis.RANDOM)
   >>> # 从所有集群节点获取键
   >>> rc.keys(target_nodes=Redis.ALL_NODES)
   [b'foo1', b'foo2']
   >>> # 在所有主节点上执行 bgsave
   >>> rc.bgsave(Redis.PRIMARIES)

如果您想在特定 节点/节点组 上执行命令，而该 节点/节点组 未被节点标志覆盖，也可以直接传递 ``ClusterNodes`` 。但是，如果由于集群拓扑变化而导致命令执行失败，则不会进行重试尝试，因为传递的目标节点可能不再有效，并将返回相关的集群或连接错误。

.. code:: python

   >>> node = rc.get_node('localhost', 6379)
   >>> # 仅获取该特定节点的键
   >>> rc.keys(target_nodes=node)
   >>> # 从一部分主节点获取 Redis 信息
   >>> subset_primaries = [node for node in rc.get_primaries() if node.port > 6378]
   >>> rc.info(target_nodes=subset_primaries)

此外， :py:class:`~redis.cluster.RedisCluster` 实例可以查询特定节点的 :py:class:`~redis.Redis` 实例并直接在该节点上执行命令。然而， Redis 客户端不处理集群故障和重试。

.. code:: python

   >>> cluster_node = rc.get_node(host='localhost', port=6379)
   >>> print(cluster_node)
   [host=127.0.0.1,port=6379,name=127.0.0.1:6379,server_type=primary,redis_connection=Redis<ConnectionPool<Connection<host=127.0.0.1,port=6379,db=0>>>]
   >>> r = cluster_node.redis_connection
   >>> r.client_list()
   [{'id': '276', 'addr': '127.0.0.1:64108', 'fd': '16', 'name': '', 'age': '0', 'idle': '0', 'flags': 'N', 'db': '0', 'sub': '0', 'psub': '0', 'multi': '-1', 'qbuf': '26', 'qbuf-free': '32742', 'argv-mem': '10', 'obl': '0', 'oll': '0', 'omem': '0', 'tot-mem': '54298', 'events': 'r', 'cmd': 'client', 'user': 'default'}]
   >>> # Get the keys only for that specific node
   >>> r.keys()
   [b'foo1']

多键命令(Multi-key Commands)
------------------------------------

Redis 在集群模式下支持多键命令，例如集合类型的 **并集** 或 **交集** 、 **mset** 和 **mget** ，只要所有键都哈希到同一个槽(slot)。通过使用 :py:class:`~redis.cluster.RedisCluster` 客户端，可以使用已知的函数（例如 ``mget`` 、 ``mset`` ）执行原子多键操作。但是，必须确保所有键映射到相同的槽(slot)，否则将抛出 :py:class:`~redis.exceptions.RedisClusterException`。Redis 集群实现了一种称为哈希标签的概念，可以用来强制某些键存储在同一个哈希槽中，参见 `Keys hash tag <https://redis.io/topics/cluster-spec#keys-hash-tags>`__。您还可以对某些多键操作使用 非原子操作，并传递未映射到相同槽(slot)的键。客户端将会将这些键映射到相关槽(relevant slots)，并将命令发送给这些槽的节点所有者(node owners)。非原子操作根据它们的哈希值对键进行分批处理，然后将每批分别发送到槽的所有者(node owners)。

.. code:: python

   # 原子操作可以在所有键映射到同一槽时使用
   >>> rc.mset({'{foo}1': 'bar1', '{foo}2': 'bar2'})
   >>> rc.mget('{foo}1', '{foo}2')
   [b'bar1', b'bar2']
   # 非原子多键操作将键拆分到不同的槽
   >>> rc.mset_nonatomic({'foo': 'value1', 'bar': 'value2', 'zzz': 'value3'})
   >>> rc.mget_nonatomic('foo', 'bar', 'zzz')
   [b'value1', b'value2', b'value3']

**集群 发布/订阅(PubSub):**

当创建 :py:class:`redis.cluster.ClusterPubSub` 实例时，如果未指定节点，将在第一次命令执行时透明地选择一个节点用于 pubsub 连接。该节点的确定过程如下: 1. 对请求中的频道名称进行哈希以找到其键槽(keyslot)；2. 选择一个处理该键槽(keyslot)的节点：如果 ``read_from_replicas`` 设置为 ``true``，则可以选择副本。

已知的 发布/订阅(PubSub) 限制
-----------------------------------------

模式订阅和发布由于键槽(keyslot)的问题，目前无法正常工作。如果我们对像 fo\* 这样的模式进行哈希，将会得到该字符串的一个键槽(keyslot)，但基于该模式的频道名称有无数种可能性——这是无法事先知道的。该功能并未被禁用，但目前不推荐使用这些命令。有关更多信息，请参见 `redis-py-cluster documentation <https://redis-py-cluster.readthedocs.io/en/stable/pubsub.html>`__。

.. code:: python

   >>> p1 = rc.pubsub()
   # p1 连接将设置为持有 'foo' 键槽(keyslot)的节点
   >>> p1.subscribe('foo')
   # p2 连接将设置为节点 'localhost:6379'
   >>> p2 = rc.pubsub(rc.get_node('localhost', 6379))

**只读模式(Read Only Mode)**

默认情况下，Redis 集群在访问副本节点时始终返回 ``MOVE`` 重定向响应。您可以通过触发 ``READONLY`` 模式来克服此限制并扩展读取命令。

要启用 ``READONLY`` 模式，请将 ``read_from_replicas=True`` 传递给 :py:class:`~redis.cluster.RedisCluster` 构造函数。当设置为 ``true`` 时，读取命令将在主节点及其副本之间以轮询的方式分配。

可以通过调用 ``readonly()`` 方法并将 ``target_nodes=``replicas```` 来在运行时设置 ``READONLY`` 模式，而可以通过调用 ``readwrite()`` 方法恢复读写访问。

.. code:: python

   >>> from cluster import RedisCluster as Redis
   # 使用 'debug' 日志级别打印执行命令的节点
   >>> rc_readonly = Redis(startup_nodes=startup_nodes,
   ...                     read_from_replicas=True)
   >>> rc_readonly.set('{foo}1', 'bar1')
   >>> for i in range(0, 4):
   ...     # 以轮询方式将读取命令分配给槽的主机
   ...     rc_readonly.get('{foo}1')
   # set 命令将仅被定向到槽的主节点
   >>> rc_readonly.set('{foo}2', 'bar2')
   # 重置 READONLY 标志
   >>> rc_readonly.readwrite(target_nodes='replicas')
   # 现在 get 命令将仅被定向到槽的主节点
   >>> rc_readonly.get('{foo}1')