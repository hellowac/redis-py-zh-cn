重试工具(Retry Helpers)
##########################

.. automodule:: redis.retry
    :members:

在 Redis Standalone 中的重试(Retry in Redis Standalone)
*****************************************************************

.. code:: python

    >>> from redis.backoff import ExponentialBackoff
    >>> from redis.retry import Retry
    >>> from redis.client import Redis
    >>> from redis.exceptions import (
    >>>    BusyLoadingError,
    >>>    ConnectionError,
    >>>    TimeoutError
    >>> )
    >>>
    >>> # 使用指数退避策略进行3次重试
    >>> retry = Retry(ExponentialBackoff(), 3)
    >>> # 配置在自定义错误时进行重试的 Redis 客户端
    >>> r = Redis(host='localhost', port=6379, retry=retry, retry_on_error=[BusyLoadingError, ConnectionError, TimeoutError])
    >>> # 配置仅在超时错误时进行重试的 Redis 客户端
    >>> r_only_timeout = Redis(host='localhost', port=6379, retry=retry, retry_on_timeout=True)


从上述示例可以看出，Redis 客户端支持 3 个参数来配置重试行为：

* ``retry``: 带有 :ref:`backoff-label` 策略和最大重试次数的 :class:`~.Retry` 实例
* ``retry_on_error``: 重试的 :ref:`exceptions-label` 异常列表
* ``retry_on_timeout``: 如果为 ``True``，仅在 :class:`~.TimeoutError` 时重试

如果传入了 ``retry_on_error`` 或 ``retry_on_timeout``，但未提供 ``retry``，则默认使用 ``Retry(NoBackoff(), 1)`` （意味着在第一次失败后立即重试一次）。

在 Redis Cluster 中的重试(Retry in Redis Cluster)
****************************************************

.. code:: python

    >>> from redis.backoff import ExponentialBackoff
    >>> from redis.retry import Retry
    >>> from redis.cluster import RedisCluster
    >>>
    >>> # 使用指数退避策略进行3次重试
    >>> retry = Retry(ExponentialBackoff(), 3)
    >>> # 配置重试的 Redis Cluster 客户端
    >>> rc = RedisCluster(host='localhost', port=6379, retry=retry, cluster_error_retry_attempts=2)


Redis Cluster 中的重试行为与 Standalone 略有不同：

* ``retry``: 带有 :ref:`backoff-label` 策略和最大重试次数的 :class:`~.Retry` 实例，默认值为 ``Retry(NoBackoff(), 0)``
* ``cluster_error_retry_attempts``: 当遇到 :class:`~.TimeoutError`、:class:`~.ConnectionError` 或 :class:`~.ClusterDownError` 时，重试的次数，默认值为 ``3``

我们来看以下示例：

.. code:: python

    >>> from redis.backoff import ExponentialBackoff
    >>> from redis.retry import Retry
    >>> from redis.cluster import RedisCluster
    >>>
    >>> rc = RedisCluster(host='localhost', port=6379, 
                          retry=Retry(ExponentialBackoff(), 6), cluster_error_retry_attempts=1)
    >>> rc.set('foo', 'bar')

* 客户端库计算键 'foo' 的哈希槽。
* 根据哈希槽，确定要连接到哪个节点以执行命令。
* 在连接期间，抛出了 :class:`~.ConnectionError`。
* 由于我们设置了 ``retry=Retry(ExponentialBackoff(), 6)``，客户端会尝试重新连接该节点最多6次，每次尝试之间采用指数退避。
* 即便经过6次重试，客户端仍无法连接。
* 由于我们设置了 ``cluster_error_retry_attempts=1``，在放弃之前，客户端会启动集群更新，将失败的节点从启动节点中移除，并重新初始化集群。
* 集群重新初始化后，开始新一轮的重试循环，最多6次重试，每次之间采用指数退避。
* 如果客户端能够连接，一切正常；否则，由于已耗尽所有尝试，异常最终会抛给调用者。