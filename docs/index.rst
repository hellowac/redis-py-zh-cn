.. redis-py documentation master file, created by
   sphinx-quickstart on Thu Jul 28 13:55:57 2011.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

redis-py - Python Client for Redis
====================================

入门
****************

`redis-py <https://pypi.org/project/redis>`_ 需要一个正在运行的 Redis 服务器，并且要求 Python 版本为 3.7 以上。请参阅 `Redis 快速入门 <https://redis.io/topics/quickstart>`_ 获取 Redis 安装说明。

你可以通过 pip 来安装 `redis-py`：
``pip install redis``。

快速连接到 Redis
********************

有两种快速连接 Redis 的方法。

**假设你在 localhost:6379 上运行 Redis（默认端口）**

.. code-block:: python

   import redis
   r = redis.Redis()
   r.ping()

**在 foo.bar.com 上运行 Redis，端口为 12345**

.. code-block:: python

   import redis
   r = redis.Redis(host='foo.bar.com', port=12345)
   r.ping()

**另一个使用 foo.bar.com 和端口 12345 的示例**

.. code-block:: python

   import redis
   r = redis.from_url('redis://foo.bar.com:12345')
   r.ping()

之后，你可能会想 `执行 Redis 命令 <commands.html>`_ 。

.. toctree::
   :hidden:

   genindex

Redis 命令函数
***********************

.. toctree::
   :maxdepth: 3

   commands/index
   redismodules/index

模块文档
********************
.. toctree::
   :maxdepth: 1

   connections
   clustering
   exceptions
   backoff
   lock
   retry
   lua_scripting
   opentelemetry
   resp3_features
   advanced_features
   examples

贡献
*************

- `如何贡献 <https://github.com/redis/redis-py/blob/master/CONTRIBUTING.md>`_
- `问题追踪 <https://github.com/redis/redis-py/issues>`_
- `源代码 <https://github.com/redis/redis-py/>`_
- `发布历史 <https://github.com/redis/redis-py/releases/>`_

许可证
*******

本项目根据 `MIT 许可证 <https://github.com/redis/redis-py/blob/master/LICENSE>`_ 获得许可。