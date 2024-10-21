# redis-py

Redis 键值存储的 Python 接口。

[![CI](https://github.com/redis/redis-py/workflows/CI/badge.svg?branch=master)](https://github.com/redis/redis-py/actions?query=workflow%3ACI+branch%3Amaster)
[![文档](https://readthedocs.org/projects/redis/badge/?version=stable&style=flat)](https://redis-py.readthedocs.io/en/stable/)
[![MIT 许可](https://img.shields.io/badge/license-MIT-blue.svg)](./LICENSE)
[![pypi](https://badge.fury.io/py/redis.svg)](https://pypi.org/project/redis/)
[![预发布版本](https://img.shields.io/github/v/release/redis/redis-py?include_prereleases&label=latest-prerelease)](https://github.com/redis/redis-py/releases)
[![codecov](https://codecov.io/gh/redis/redis-py/branch/master/graph/badge.svg?token=yenl5fzxxr)](https://codecov.io/gh/redis/redis-py)

[安装](#installation) |  [使用](#usage) | [高级主题](#advanced-topics) | [贡献](https://github.com/redis/redis-py/blob/master/CONTRIBUTING.md)

---------------------------------------------

**注意：** redis-py 5.0 将是最后一个支持 Python 3.7 的版本，因为该版本已到达[生命周期终点](https://devguide.python.org/versions/)。redis-py 5.1 将支持 Python 3.8+。

---------------------------------------------

## 如何使用 Redis？

[在 Redis 大学免费学习](https://redis.io/university/)

[试用 Redis 云](https://redis.io/try-free/)

[深入开发者教程](https://redis.io/learn)

[加入 Redis 社区](https://redis.io/community/)

[在 Redis 工作](https://redis.io/careers/)

## 安装

通过 docker 启动 Redis：

``` bash
docker run -p 6379:6379 -it redis/redis-stack:latest
```

要安装 redis-py，只需执行以下命令：

``` bash
$ pip install redis
```

为了获得更高的性能，您可以安装支持 hiredis 的 redis-py，它提供了一个编译的响应解析器，*对于大多数情况*，无需更改代码。默认情况下，如果 hiredis 版本 >= 1.0 可用，redis-py 将尝试使用它来解析响应。

``` bash
$ pip install "redis[hiredis]"
```

在寻找处理对象映射的高级库？请参阅 [redis-om-python](https://github.com/redis/redis-om-python)！

## 支持的 Redis 版本

该库的最新版本支持 Redis 版本 [5.0](https://github.com/redis/redis/blob/5.0/00-RELEASENOTES)、[6.0](https://github.com/redis/redis/blob/6.0/00-RELEASENOTES)、[6.2](https://github.com/redis/redis/blob/6.2/00-RELEASENOTES)、[7.0](https://github.com/redis/redis/blob/7.0/00-RELEASENOTES)、[7.2](https://github.com/redis/redis/blob/7.2/00-RELEASENOTES) 和 [7.4](https://github.com/redis/redis/blob/7.4/00-RELEASENOTES)。

下表展示了最新库版本与 Redis 版本的兼容性。

| 库版本 | 支持的 Redis 版本 |
|-----------------|-------------------|
| 3.5.3 | <= 6.2 版本系列 |
| >= 4.5.0 | 5.0 到 7.0 版本 |
| >= 5.0.0 | 5.0 到当前版本 |


翻译如下：

## 使用方法

### 基本示例

``` python
>>> import redis
>>> r = redis.Redis(host='localhost', port=6379, db=0)
>>> r.set('foo', 'bar')
True
>>> r.get('foo')
b'bar'
```

上述代码连接到 `localhost` 上的端口 6379，在 Redis 中设置一个值，并检索该值。所有响应在 Python 中均以字节形式返回，若要接收解码后的字符串，设置 *decode_responses=True*。关于此选项及更多连接选项，请参阅[这些示例](https://redis.readthedocs.io/en/stable/examples.html)。

#### RESP3 支持
要启用 RESP3 支持，请确保客户端至少为 5.0 版本，并将连接对象中的协议设置为 *protocol=3*。

``` python
>>> import redis
>>> r = redis.Redis(host='localhost', port=6379, db=0, protocol=3)
```

### 连接池

默认情况下，redis-py 使用连接池来管理连接。每个 Redis 类的实例都会获得自己的连接池。不过，你可以定义自己的 [redis.ConnectionPool](https://redis.readthedocs.io/en/stable/connections.html#connection-pools)。

``` python
>>> pool = redis.ConnectionPool(host='localhost', port=6379, db=0)
>>> r = redis.Redis(connection_pool=pool)
```

或者，你可以查看 [异步连接](https://redis.readthedocs.io/en/stable/examples/asyncio_examples.html)、[集群连接](https://redis.readthedocs.io/en/stable/connections.html#cluster-client)，甚至是 [异步集群连接](https://redis.readthedocs.io/en/stable/connections.html#async-cluster-client)。

### Redis 命令

redis-py 内置支持所有[开箱即用的 Redis 命令](https://redis.io/commands)。这些命令使用原始的 Redis 命令名（如 `HSET`、`HGETALL` 等）进行暴露，除了某些被语言保留的关键字（例如 `del`）。完整的命令集可以在[此处](https://github.com/redis/redis-py/tree/master/redis/commands)或[文档](https://redis.readthedocs.io/en/stable/commands.html)中找到。

## 高级主题

[官方 Redis 命令文档](https://redis.io/commands)很好地解释了每个命令的详细信息。redis-py 尽量遵守官方命令语法，但有一些例外：

-   **MULTI/EXEC**：这些命令作为 Pipeline 类的一部分实现。默认情况下，执行时会将 Pipeline 包裹在 MULTI 和 EXEC 语句中，可以通过指定 `transaction=False` 来禁用此行为。更多关于 Pipeline 的内容请见下文。

-   **SUBSCRIBE/LISTEN**：与 Pipeline 类似，PubSub 被实现为一个单独的类，因为它将底层连接置于无法执行非 PubSub 命令的状态。调用 Redis 客户端的 pubsub 方法将返回一个 PubSub 实例，在该实例中可以订阅频道并监听消息。只能从 Redis 客户端调用 `PUBLISH`（详情请参阅[关于 issue #151 的评论](https://github.com/redis/redis-py/issues/151#issuecomment-1545015)）。

有关更多详细信息，请参阅[高级主题页面](https://redis.readthedocs.io/en/stable/advanced_features.html)上的文档。

翻译如下：

### 管道（Pipelines）

以下是一个 [Redis 管道](https://redis.io/docs/manual/pipelining/) 的基本示例。管道是一种通过批量发送 Redis 命令并以列表形式接收其结果来优化往返调用的方法。

``` python
>>> pipe = r.pipeline()
>>> pipe.set('foo', 5)
>>> pipe.set('bar', 18.5)
>>> pipe.set('blee', "hello world!")
>>> pipe.execute()
[True, True, True]
```

### 发布/订阅（PubSub）

以下示例展示了如何使用 [Redis 发布/订阅（Pub/Sub）](https://redis.io/docs/manual/pubsub/) 订阅特定频道。

``` python
>>> r = redis.Redis(...)
>>> p = r.pubsub()
>>> p.subscribe('my-first-channel', 'my-second-channel', ...)
>>> p.get_message()
{'pattern': None, 'type': 'subscribe', 'channel': b'my-second-channel', 'data': 1}
```

--------------------------

### 作者

redis-py 由 [Redis Inc](https://redis.io) 开发和维护。你可以在[这里](https://github.com/redis/redis-py)找到它，或从 [pypi](https://pypi.org/project/redis/) 下载。

特别感谢：

-   Andy McCurdy (<sedrik@gmail.com>)，redis-py 的原作者。
-   Ludovico Magnocavallo，原 Python Redis 客户端的作者，其中的一些 socket 代码仍在使用。
-   Alexander Solovyov，提出了通用响应回调系统的想法。
-   Paul Hubbard，为最初的打包提供支持。

[![Redis](./docs/_static/logo-redis.svg)](https://redis.io)
