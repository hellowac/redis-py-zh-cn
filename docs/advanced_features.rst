高级功能(Advanced Features)
==================================

关于线程的说明(A note about threading)  
--------------------------------------------

Redis 客户端实例可以安全地在多个线程之间共享。在内部，连接实例仅在执行命令时从连接池中获取，并在执行完命令后立即返回到连接池中。命令执行时不会修改客户端实例的状态。

然而，有一个例外情况：Redis 的 SELECT 命令。SELECT 命令允许你切换当前连接使用的数据库。该数据库将保持选中状态，直到选择了另一个数据库或关闭连接。这会导致一个问题，即某些连接可能会返回到连接池中时仍连接到不同的数据库。

因此，redis-py 不在客户端实例上实现 SELECT 命令。如果你在同一个应用程序中使用多个 Redis 数据库，则应该为每个数据库创建一个单独的客户端实例（可能还需要单独的连接池）。

将 PubSub 或 Pipeline 对象在线程之间传递是不安全的。

管道(Pipelines)
------------------

默认管道(Default pipelines)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

管道 (Pipelines) 是 Redis 基类的子类，提供了将多个命令缓冲到服务器的单个请求中的支持。通过减少客户端与服务器之间来回发送的 TCP 数据包数量，它们可以显著提高一组命令的执行性能。

使用管道非常简单：

.. code:: python

   >>> r = redis.Redis(...)
   >>> r.set('bing', 'baz')
   >>> # 使用 pipeline() 方法创建一个管道实例
   >>> pipe = r.pipeline()
   >>> # 以下 SET 命令将被缓冲
   >>> pipe.set('foo', 'bar')
   >>> pipe.get('bing')
   >>> # EXECUTE 调用会将所有缓冲的命令发送到服务器，并返回
   >>> # 一个响应列表，每个命令对应一个响应。
   >>> pipe.execute()
   [True, b'baz']

为了便于使用，所有缓冲到管道中的命令都会返回管道对象本身。因此，可以像这样进行链式调用：

.. code:: python

   >>> pipe.set('foo', 'bar').sadd('faz', 'baz').incr('auto_number').execute()
   [True, True, 6]

此外，管道还可以确保缓冲的命令以原子方式作为一个组执行。这是默认行为。如果你想禁用管道的原子性，但仍然希望缓冲命令，可以关闭事务。

.. code:: python

   >>> pipe = r.pipeline(transaction=False)

一个常见问题是，需要原子事务但需要在事务中使用 Redis 中的值。例如，假设 INCR 命令不存在，我们需要在 Python 中构建一个原子的 INCR。

一个非常天真的实现是 GET 值，在 Python 中递增它，然后将新值 SET 回去。然而，这并不是原子的，因为多个客户端可能同时执行此操作，每个客户端都会从 GET 获取相同的值。

此时可以使用 WATCH 命令。WATCH 允许在启动事务之前监视一个或多个键。如果这些键在事务执行之前发生了变化，则整个事务将被取消，并抛出 WatchError。为了实现我们自己的客户端 INCR 命令，可以这样做：

.. code:: python

   >>> with r.pipeline() as pipe:
   ...     while True:
   ...         try:
   ...             # 监视持有序列值的键
   ...             pipe.watch('OUR-SEQUENCE-KEY')
   ...             # 在 WATCH 后，管道将进入立即执行模式，
   ...             # 直到我们告诉它再次开始缓冲命令。
   ...             # 这使我们可以获取序列的当前值
   ...             current_value = pipe.get('OUR-SEQUENCE-KEY')
   ...             next_value = int(current_value) + 1
   ...             # 现在我们可以使用 MULTI 将管道重新置于缓冲模式
   ...             pipe.multi()
   ...             pipe.set('OUR-SEQUENCE-KEY', next_value)
   ...             # 最后，执行管道（SET 命令）
   ...             pipe.execute()
   ...             # 如果在执行过程中没有抛出 WatchError，那么
   ...             # 我们所做的一切都是原子操作。
   ...             break
   ...        except WatchError:
   ...             # 在我们开始 WATCH 和管道执行之间，另一个客户端
   ...             # 必须更改了 'OUR-SEQUENCE-KEY'。
   ...             # 我们最好重试。
   ...             continue

请注意，由于管道在 WATCH 期间必须绑定到单个连接，因此必须小心确保通过调用 reset() 方法将连接返回到连接池。如果管道作为上下文管理器使用（如上例所示），reset() 将自动调用。当然，你也可以通过显式调用 reset() 手动完成此操作：

.. code:: python

   >>> pipe = r.pipeline()
   >>> while True:
   ...     try:
   ...         pipe.watch('OUR-SEQUENCE-KEY')
   ...         ...
   ...         pipe.execute()
   ...         break
   ...     except WatchError:
   ...         continue
   ...     finally:
   ...         pipe.reset()

一个名为 "transaction" 的便捷方法存在，用于处理监视错误的所有样板代码。它接收一个可调用对象，该对象应期望一个管道对象作为参数，并且可以传入任何数量的要监视的键。我们上面的客户端 INCR 命令可以这样重写，代码更易于阅读：

.. code:: python

   >>> def client_side_incr(pipe):
   ...     current_value = pipe.get('OUR-SEQUENCE-KEY')
   ...     next_value = int(current_value) + 1
   ...     pipe.multi()
   ...     pipe.set('OUR-SEQUENCE-KEY', next_value)
   >>>
   >>> r.transaction(client_side_incr, 'OUR-SEQUENCE-KEY')
   [True]

请确保在传递给 Redis.transaction 的可调用对象中，在任何写入命令之前调用 pipe.multi()。

集群中的管道(Pipelines in clusters)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:class:`~redis.asyncio.cluster.ClusterPipeline` 是 :class:`~redis.cluster.RedisCluster` 的一个子类，提供了在集群模式下对 Redis 管道的支持。当调用 execute() 命令时，所有命令都会按照将要执行的节点进行分组，然后由各自的节点并行执行。管道实例将在所有节点响应之前等待，才会将结果返回给调用者。命令响应以列表形式返回，顺序与发送顺序相同。管道可以显著提高 Redis 集群的吞吐量，通过大幅减少客户端与服务器之间的网络往返次数。

.. code:: python

   >>> with rc.pipeline() as pipe:
   ...     pipe.set('foo', 'value1')
   ...     pipe.set('bar', 'value2')
   ...     pipe.get('foo')
   ...     pipe.get('bar')
   ...     print(pipe.execute())
   [True, True, b'value1', b'value2']
   ...     pipe.set('foo1', 'bar1').get('foo1').execute()
   [True, b'bar1']

请注意：
- RedisCluster 管道目前仅支持基于键的命令。
- 管道从集群的参数中获取 ‘read_from_replicas’ 的值。因此，如果集群实例中启用了从副本读取，则管道也会将读取命令指向副本。
- 在集群模式下，不支持 ‘transaction’ 选项。在非集群模式下，执行管道时可以使用 ‘transaction’ 选项。这将使用 MULTI/EXEC 命令包装管道命令，并有效地将管道命令转换为单个事务块。这意味着所有命令将按顺序执行，而不会受到其他客户端的干扰。然而，在集群模式下，这是不可能的，因为命令根据其各自的目标节点进行分区。这意味着我们无法将管道命令转变为一个事务块，因为在大多数情况下，它们被拆分为多个较小的管道。

发布/订阅(Publish / Subscribe)
--------------------------------------

redis-py 包含一个 PubSub 对象，用于订阅频道并监听新消息。创建一个 PubSub 对象非常简单。

.. code:: python

   >>> r = redis.Redis(...)
   >>> p = r.pubsub()

一旦创建了 PubSub 实例，就可以订阅频道和模式。

.. code:: python

   >>> p.subscribe('my-first-channel', 'my-second-channel', ...)
   >>> p.psubscribe('my-*', ...)

现在，PubSub 实例已订阅这些频道/模式。可以通过从 PubSub 实例读取消息来查看订阅确认。

.. code:: python

   >>> p.get_message()
   {'pattern': None, 'type': 'subscribe', 'channel': b'my-second-channel', 'data': 1}
   >>> p.get_message()
   {'pattern': None, 'type': 'subscribe', 'channel': b'my-first-channel', 'data': 2}
   >>> p.get_message()
   {'pattern': None, 'type': 'psubscribe', 'channel': b'my-*', 'data': 3}

从 PubSub 实例读取的每条消息将是一个包含以下键的字典。

-  **type**: 以下之一：'subscribe'、'unsubscribe'、'psubscribe'、'punsubscribe'、'message'、'pmessage'
-  **channel**: [取消]订阅的频道或发布消息的频道
-  **pattern**: 匹配已发布消息频道的模式。对于 'pmessage' 类型，其他情况将为 None。
-  **data**: 消息数据。对于 [un]subscribe 消息，此值将是当前连接订阅的频道和模式的数量。对于 [p]message 消息，此值将是实际发布的消息。

现在我们发送一条消息。

.. code:: python

   # publish 方法返回匹配的频道和模式
   # 订阅的数量。'my-first-channel' 匹配
   # 'my-first-channel' 订阅和 'my-*' 模式订阅，
   # 因此此消息将被送达 2 个频道/模式。
   >>> r.publish('my-first-channel', 'some data')
   2
   >>> p.get_message()
   {'channel': b'my-first-channel', 'data': b'some data', 'pattern': None, 'type': 'message'}
   >>> p.get_message()
   {'channel': b'my-first-channel', 'data': b'some data', 'pattern': b'my-*', 'type': 'pmessage'}

取消订阅的方式与订阅相同。如果没有参数传递给 [p]unsubscribe，则将取消所有频道或模式的订阅。

.. code:: python

   >>> p.unsubscribe()
   >>> p.punsubscribe('my-*')
   >>> p.get_message()
   {'channel': b'my-second-channel', 'data': 2, 'pattern': None, 'type': 'unsubscribe'}
   >>> p.get_message()
   {'channel': b'my-first-channel', 'data': 1, 'pattern': None, 'type': 'unsubscribe'}
   >>> p.get_message()
   {'channel': b'my-*', 'data': 0, 'pattern': None, 'type': 'punsubscribe'}

redis-py 还允许你注册回调函数来处理已发布的消息。消息处理程序接受一个参数，即消息，这也是一个与上面示例相同的字典。要使用消息处理程序订阅频道或模式，可以将频道或模式名称作为关键字参数传入，其值为回调函数。

当在频道或模式上读取带有消息处理程序的消息时，将创建消息字典并将其传递给消息处理程序。在这种情况下，从 get_message() 返回一个 None 值，因为消息已经被处理。

.. code:: python

   >>> def my_handler(message):
   ...     print('MY HANDLER: ', message['data'])
   >>> p.subscribe(**{'my-channel': my_handler})
   # 读取订阅确认消息
   >>> p.get_message()
   {'pattern': None, 'type': 'subscribe', 'channel': b'my-channel', 'data': 1}
   >>> r.publish('my-channel', 'awesome data')
   1
   # 为了使消息处理程序工作，我们需要告诉实例读取数据。
   # 这可以通过多种方式完成（详见下文）。我们暂时将使用
   # 熟悉的 get_message() 函数
   >>> message = p.get_message()
   MY HANDLER:  awesome data
   # 请注意，my_handler 回调打印了上面的字符串。
   # `message` 为 None，因为消息已由我们的处理程序处理。
   >>> print(message)
   None

如果您的应用程序对（有时嘈杂的）订阅/取消订阅确认消息不感兴趣，可以通过将 ignore_subscribe_messages=True 传递给 r.pubsub() 来忽略它们。这将导致所有订阅/取消订阅消息被读取，但它们不会冒泡到您的应用程序中。

.. code:: python

   >>> p = r.pubsub(ignore_subscribe_messages=True)
   >>> p.subscribe('my-channel')
   >>> p.get_message()  # 隐藏订阅消息并返回 None
   >>> r.publish('my-channel', 'my data')
   1
   >>> p.get_message()
   {'channel': b'my-channel', 'data': b'my data', 'pattern': None, 'type': 'message'}

读取消息有三种不同的策略。

上述示例使用了 pubsub.get_message()。在幕后，get_message() 使用系统的 'select' 模块快速轮询连接的套接字。如果有可供读取的数据，get_message() 将读取它，格式化消息并返回或传递给消息处理程序。如果没有数据可供读取，get_message() 将立即返回 None。这使得将其集成到您应用程序中的现有事件循环中变得微不足道。

.. code:: python

   >>> while True:
   >>>     message = p.get_message()
   >>>     if message:
   >>>         # 处理消息
   >>>     time.sleep(0.001)  # 对系统友好 :)

旧版本的 redis-py 仅使用 pubsub.listen() 读取消息。listen() 是一个生成器，阻塞直到有消息可用。如果您的应用程序不需要做其他事情，只需接收和处理来自 redis 的消息，listen() 是一种简单的启动方式。

.. code:: python

   >>> for message in p.listen():
   ...     # 处理消息

第三个选项是在单独的线程中运行事件循环。pubsub.run_in_thread() 创建一个新线程并启动事件循环。线程对象将返回给 run_in_thread() 的调用者。调用者可以使用 thread.stop() 方法关闭事件循环和线程。在幕后，这只是 get_message() 的一个包装，它在单独的线程中运行，实际上为您创建了一个微小的非阻塞事件循环。run_in_thread() 还接受一个可选的 sleep_time 参数。如果指定，事件循环将在每次循环迭代中调用 time.sleep()。

注意：由于我们在单独的线程中运行，因此无法处理未通过注册的消息处理程序自动处理的消息。因此，redis-py 阻止您在订阅没有消息处理程序附加的模式或频道时调用 run_in_thread()。

.. code:: python

   >>> p.subscribe(**{'my-channel': my_handler})
   >>> thread = p.run_in_thread(sleep_time=0.001)
   # 事件循环现在在后台运行，处理消息
   # 当需要关闭时...
   >>> thread.stop()

run_in_thread 还支持一个可选的异常处理程序，允许您捕获在工作线程中发生的异常并进行适当处理。异常处理程序将以异常本身、pubsub 对象和 run_in_thread 返回的工作线程作为参数。

.. code:: python

   >>> p.subscribe(**{'my-channel': my_handler})
   >>> def exception_handler(ex, pubsub, thread):
   >>>     print(ex)
   >>>     thread.stop()
   >>>     thread.join(timeout=1.0)
   >>>     pubsub.close()
   >>> thread = p.run_in_thread(exception_handler=exception_handler)

PubSub 对象遵循与其创建时的客户端实例相同的编码语义。任何 Unicode 的频道或模式将在发送到 Redis 之前使用客户端指定的字符集进行编码。如果客户端的 decode_responses 标志设置为 False（默认值），则消息字典中的 'channel'、'pattern' 和 'data' 值将是字节字符串（在 Python 2 中为 str，在 Python 3 中为 bytes）。如果客户端的 decode_responses 为 True，则 'channel'、'pattern' 和 'data' 值将使用客户端的字符集自动解码为 Unicode 字符串。

PubSub 对象记住它们订阅的频道和模式。如果发生断开连接，例如网络错误或超时，PubSub 对象将在重新连接时重新订阅所有先前的频道和模式。在客户端断开连接期间发布的消息无法交付。当您完成使用 PubSub 对象时，请调用其 .close() 方法以关闭连接。

.. code:: python

   >>> p = r.pubsub()
   >>> ...
   >>> p.close()

PUBSUB 子命令集 CHANNELS、NUMSUB 和 NUMPAT 也受支持：

.. code:: python

   >>> r.pubsub_channels()
   [b'foo', b'bar']
   >>> r.pubsub_numsub('foo', 'bar')
   [(b'foo', 9001), (b'bar', 42)]
   >>> r.pubsub_numsub('baz')
   [(b'baz', 0)]
   >>> r.pubsub_numpat()
   1204

共享 pubsub(Sharded pubsub)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

`Sharded pubsub <https://redis.io/docs/interact/pubsub/#:~:text=Sharded%20Pub%2FSub%20helps%20to,the%20shard%20of%20a%20cluster.>`_ 是 Redis 7.0 引入的一项功能，并在 5.0 版本的 redis-py 中得到完全支持。它通过将消息分片到拥有分片通道槽的节点，帮助扩展集群模式下的 pub/sub 使用。在这里，集群确保发布的分片消息被转发到适当的节点。客户端通过连接到负责该槽的主节点或其任何副本来订阅频道。

这利用了 Redis 中的 `SSUBSCRIBE <https://redis.io/commands/ssubscribe>`_ 和 `SPUBLISH <https://redis.io/commands/spublish>`_ 命令。

以下是一个简化的示例：

.. code:: python

    >>> from redis.cluster import RedisCluster, ClusterNode
    >>> r = RedisCluster(startup_nodes=[ClusterNode('localhost', 6379), ClusterNode('localhost', 6380)])
    >>> p = r.pubsub()
    >>> p.ssubscribe('foo')
    >>> # 假设某人通过发布沿通道发送了一条消息
    >>> message = p.get_sharded_message()

同样，可以通过将节点传递给 get_sharded_message 来获取已发送到特定节点的分片 pubsub 消息：

.. code:: python

    >>> from redis.cluster import RedisCluster, ClusterNode
    >>> first_node = ClusterNode('localhost', 6379)
    >>> second_node = ClusterNode('localhost', 6380)
    >>> r = RedisCluster(startup_nodes=[first_node, second_node])
    >>> p = r.pubsub()
    >>> p.ssubscribe('foo')
    >>> # 假设某人通过发布沿通道发送了一条消息
    >>> message = p.get_sharded_message(target_node=second_node)


监控(Monitor)
~~~~~~~~~~~~

redis-py 包含一个 Monitor 对象，可以流式传输 Redis 服务器处理的每个命令。使用 Monitor 对象上的 listen() 阻塞直到接收到命令。

.. code:: python

   >>> r = redis.Redis(...)
   >>> with r.monitor() as m:
   >>>     for command in m.listen():
   >>>         print(command)
