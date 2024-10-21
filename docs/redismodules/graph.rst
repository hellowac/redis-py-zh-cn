RedisGraph 命令
================================

这些是与 `RedisGraph 模块 <https://redisgraph.io>`_ 交互的命令。下面是一个简短的示例，以及命令本身的文档。

**创建一个图，添加两个节点(Create a graph, adding two nodes)**

.. code-block:: python

    import redis
    from redis.graph.node import Node

    john = Node(label="person", properties={"name": "John Doe", "age": 33}
    jane = Node(label="person", properties={"name": "Jane Doe", "age": 34}

    r = redis.Redis()
    graph = r.graph()
    graph.add_node(john)
    graph.add_node(jane)
    graph.add_node(pat)
    graph.commit()

.. automodule:: redis.commands.graph.node
    :members: Node

.. automodule:: redis.commands.graph.edge
    :members: Edge

.. automodule:: redis.commands.graph.commands
    :members: GraphCommands