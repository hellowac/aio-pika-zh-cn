.. aio-pika documentation master file, created by
   sphinx-quickstart on Fri Mar 31 17:03:20 2017.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

.. _aio-pika: https://github.com/mosquito/aio-pika
.. _asyncio: https://docs.python.org/3/library/asyncio.html
.. _aiormq: http://github.com/mosquito/aiormq/


欢迎来到 aio-pika 的文档
====================================

.. image:: https://coveralls.io/repos/github/mosquito/aio-pika/badge.svg?branch=master
    :target: https://coveralls.io/github/mosquito/aio-pika
    :alt: Coveralls

.. image:: https://github.com/mosquito/aio-pika/workflows/tests/badge.svg
    :target: https://github.com/mosquito/aio-pika/actions?query=workflow%3Atests
    :alt: Github Actions

.. image:: https://img.shields.io/pypi/v/aio-pika.svg
    :target: https://pypi.python.org/pypi/aio-pika/
    :alt: Latest Version

.. image:: https://img.shields.io/pypi/wheel/aio-pika.svg
    :target: https://pypi.python.org/pypi/aio-pika/

.. image:: https://img.shields.io/pypi/pyversions/aio-pika.svg
    :target: https://pypi.python.org/pypi/aio-pika/

.. image:: https://img.shields.io/pypi/l/aio-pika.svg
    :target: https://pypi.python.org/pypi/aio-pika/


`aio-pika`_ 是一个基于 `aiormq`_ 和 `asyncio`_ 的更人性化的包.


功能
++++++++

* 完全异步的 API。
* 面向对象的 API。
* 使用 `connect_robust` 实现透明的自动重连，完全状态恢复（例如，声明的队列或交换机、消费状态和绑定）。
* 兼容 Python 3.6 及以上版本。
* 对于 Python 3.5 用户，可以使用 `aio-pika<7`。
* 透明的 `publisher confirms`_ 支持。
* 支持 `Transactions`_ (事物)。
* 完全的类型提示覆盖。

.. _publisher confirms: https://www.rabbitmq.com/confirms.html
.. _Transactions: https://www.rabbitmq.com/semantics.html#tx

AMQP URL 参数
+++++++++++++++++++

URL 是配置连接的支持方式。为了自定义连接行为，您可以像传递查询字符串那样传递参数。

本文描述了这些参数的说明。

``aiormq`` 特定参数
~~~~~~~~~~~~~~~~~~~

* ``name`` (``str`` URL 编码) - 一个字符串，在 RabbitMQ 管理控制台和服务器日志中可见，便于诊断。

* ``cafile`` (``str``) - 证书授权文件的路径。

* ``capath`` (``str``) - 证书授权目录的路径。

* ``cadata`` (``str`` URL 编码) - URL 编码的 CA 证书内容。

* ``keyfile`` (``str``) - 客户端 SSL 私钥文件的路径。

* ``certfile`` (``str``) - 客户端 SSL 证书文件的路径。

* ``no_verify_ssl`` - 不验证服务器 SSL 证书。默认值为 ``0``，表示 ``False``，其他值表示 ``True``。

* ``heartbeat`` (``int`` 类似) - AMQP 心跳包之间的间隔（以秒为单位）。``0`` 表示禁用此功能。


``aio_pika.connect`` 函数和 ``aio_pika.Connection`` 类特定参数
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* ``interleave`` (``int`` 类似) - 控制当主机名解析为多个 IP 地址时的地址重排序。如果为 0 或未指定，则不进行重排序，地址按 ``getaddrinfo()`` 返回的顺序尝试。如果指定了正整数，则按地址族交错这些地址，该整数被解释为 `RFC 8305` 中定义的“首地址族计数”。如果未指定 ``happy_eyeballs_delay``，默认值为 ``0``；如果指定，则为 ``1``。

  .. note::

      对于使用一个 DNS 名称的 RabbitMQ 集群，具有多个 ``A``/``AAAA`` 记录，这个选项非常有用。

  .. warning::

      此选项由 ``asyncio.DefaultEventLoopPolicy`` 支持，并在 Python 3.8 及以后版本可用。

* ``happy_eyeballs_delay`` (``float`` 类似) - 如果给定，则为此连接启用 Happy Eyeballs。它应为一个浮点数，表示在开始下一个并行连接尝试之前，等待当前连接尝试完成的时间（以秒为单位）。这被称为 `RFC 8305` 中定义的“连接尝试延迟”。RFC 推荐的合理默认值是 ``0.25``（250 毫秒）。

  .. note::

      对于使用一个 DNS 名称的 RabbitMQ 集群，具有多个 ``A``/``AAAA`` 记录，这个选项非常有用。

  .. warning::

      此选项由 ``asyncio.DefaultEventLoopPolicy`` 支持，并在 Python 3.8 及以后版本可用。

``aio_pika.connect_robust`` 函数和 ``aio_pika.RobustConnection`` 类特定参数
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

对于 ``aio_pika.RobustConnection`` 类，适用所有与 ``aio_pika.Connection`` 相关的参数，如 ``name``/``interleave``/``happy_eyeballs_delay``，以及一些特定参数：

* ``reconnect_interval`` (``float`` 类似) - 重新建立连接的尝试间隔（以秒为单位），表示不超过此时间间隔进行重连尝试。

* ``fail_fast`` (``true``/``yes``/``y``/``enable``/``on``/``enabled``/``1`` 表示 ``True``，否则为 ``False``) - 在启动连接尝试时的特殊行为，如果尝试失败，则所有其他尝试将停止，并在连接阶段抛出异常。默认启用，如果你确定需要禁用此功能，请确保传递的 URL 实际可用。否则，程序将进入无休止的重连尝试，无法成功。

.. _RFC 8305: https://datatracker.ietf.org/doc/html/rfc8305.html


URL 示例
~~~~~~~~~~~~

* ``amqp://username:password@hostname/vhost?name=connection%20name&heartbeat=60&happy_eyeballs_delay=0.25``

* ``amqps://username:password@hostname/vhost?reconnect_interval=5&fail_fast=1``

* ``amqps://username:password@hostname/vhost?cafile=/path/to/ca.pem``

* ``amqps://username:password@hostname/vhost?cafile=/path/to/ca.pem&keyfile=/path/to/key.pem&certfile=/path/to/sert.pem``


安装
++++++++++++

使用pip:

.. code-block:: shell

    pip install aio-pika


使用git:

.. code-block:: shell

    # via pip
    pip install https://github.com/mosquito/aio-pika/archive/master.zip

    # manually
    git clone https://github.com/mosquito/aio-pika.git
    cd aio-pika
    python setup.py install


开发
+++++++++++

克隆项目:

.. code-block:: shell

    git clone https://github.com/mosquito/aio-pika.git
    cd aio-pika


创建一个属于 `aio-pika`_ 的虚拟环境:

.. code-block:: shell

    virtualenv -p python3.5 env

安装 `aio-pika`_ 的所有依赖:

.. code-block:: shell

    env/bin/pip install -e '.[develop]'

目录
+++++++++++++++++

.. toctree::
   :glob:
   :maxdepth: 3

   quick-start
   patterns
   rabbitmq-tutorial/index
   apidoc


感谢以下人员的贡献
+++++++++++++++++++++++

* `@mosquito`_ (author)
* `@decaz`_ (steel persuasiveness while code review)
* `@heckad`_ (bug fixes)
* `@smagafurov`_ (bug fixes)
* `@hellysmile`_ (bug fixes and ideas)
* `@altvod`_ (bug fixes)
* `@alternativehood`_ (bugfixes)
* `@cprieto`_ (bug fixes)
* `@akhoronko`_ (bug fixes)
* `@iselind`_ (bug fixes)
* `@DXist`_ (bug fixes)
* `@blazewicz`_ (bug fixes)
* `@chibby0ne`_ (bug fixes)
* `@jmccarrell`_ (bug fixes)
* `@taybin`_ (bug fixes)
* `@ollamh`_ (bug fixes)
* `@DriverX`_ (bug fixes)
* `@brianmedigate`_ (bug fixes)
* `@dan-stone`_ (bug fixes)
* `@Kludex`_ (bug fixes)
* `@bmario`_ (bug fixes)
* `@tzoiker`_ (bug fixes)
* `@Pehat`_ (bug fixes)
* `@WindowGenerator`_ (bug fixes)
* `@dhontecillas`_ (bug fixes)
* `@tilsche`_ (bug fixes)
* `@leenr`_ (bug fixes)
* `@la0rg`_ (bug fixes)
* `@SolovyovAlexander`_ (bug fixes)
* `@kremius`_ (bug fixes)
* `@zyp`_ (bug fixes)
* `@kajetanj`_ (bug fixes)
* `@Alviner`_ (moral support, debug sessions and good mood)
* `@Pavkazzz`_ (composure, and patience while debug sessions)
* `@bbrodriges`_ (supplying grammar while writing documentation)
* `@dizballanze`_ (review, grammar)

.. _@mosquito: https://github.com/mosquito
.. _@decaz: https://github.com/decaz
.. _@heckad: https://github.com/heckad
.. _@smagafurov: https://github.com/smagafurov
.. _@hellysmile: https://github.com/hellysmile
.. _@altvod: https://github.com/altvod
.. _@alternativehood: https://github.com/alternativehood
.. _@cprieto: https://github.com/cprieto
.. _@akhoronko: https://github.com/akhoronko
.. _@iselind: https://github.com/iselind
.. _@DXist: https://github.com/DXist
.. _@blazewicz: https://github.com/blazewicz
.. _@chibby0ne: https://github.com/chibby0ne
.. _@jmccarrell: https://github.com/jmccarrell
.. _@taybin: https://github.com/taybin
.. _@ollamh: https://github.com/ollamh
.. _@DriverX: https://github.com/DriverX
.. _@brianmedigate: https://github.com/brianmedigate
.. _@dan-stone: https://github.com/dan-stone
.. _@Kludex: https://github.com/Kludex
.. _@bmario: https://github.com/bmario
.. _@tzoiker: https://github.com/tzoiker
.. _@Pehat: https://github.com/Pehat
.. _@WindowGenerator: https://github.com/WindowGenerator
.. _@dhontecillas: https://github.com/dhontecillas
.. _@tilsche: https://github.com/tilsche
.. _@leenr: https://github.com/leenr
.. _@la0rg: https://github.com/la0rg
.. _@SolovyovAlexander: https://github.com/SolovyovAlexander
.. _@kremius: https://github.com/kremius
.. _@zyp: https://github.com/zyp
.. _@kajetanj: https://github.com/kajetanj
.. _@Alviner: https://github.com/Alviner
.. _@Pavkazzz: https://github.com/Pavkazzz
.. _@bbrodriges: https://github.com/bbrodriges
.. _@dizballanze: https://github.com/dizballanze


同样参考
++++++++

`aiormq`_
~~~~~~~~~

`aiormq` 是一个纯 Python 的 AMQP 客户端库。它在 **aio-pika** 的底层实现中，可以在你需要与协议进行底层交互时使用。以下示例演示了用户 API 的用法。

Simple 消费者:

.. code-block:: python

    import asyncio
    import aiormq

    async def on_message(message):
        """
        on_message doesn't necessarily have to be defined as async.
        Here it is to show that it's possible.
        """
        print(f" [x] Received message {message!r}")
        print(f"Message body is: {message.body!r}")
        print("Before sleep!")
        await asyncio.sleep(5)   # Represents async I/O operations
        print("After sleep!")

    async def main():
        # Perform connection
        connection = await aiormq.connect("amqp://guest:guest@localhost/")

        # Creating a channel
        channel = await connection.channel()

        # Declaring queue
        declare_ok = await channel.queue_declare('helo')
        consume_ok = await channel.basic_consume(
            declare_ok.queue, on_message, no_ack=True
        )

    loop = asyncio.get_event_loop()
    loop.run_until_complete(main())
    loop.run_forever()

Simple 发布者:

.. code-block:: python

    import asyncio
    from typing import Optional

    import aiormq
    from aiormq.abc import DeliveredMessage

    MESSAGE: Optional[DeliveredMessage] = None

    async def main():
        global MESSAGE
        body = b'Hello World!'

        # Perform connection
        connection = await aiormq.connect("amqp://guest:guest@localhost//")

        # Creating a channel
        channel = await connection.channel()
        declare_ok = await channel.queue_declare("hello", auto_delete=True)

        # Sending the message
        await channel.basic_publish(body, routing_key='hello')
        print(f" [x] Sent {body}")

        MESSAGE = await channel.basic_get(declare_ok.queue)
        print(f" [x] Received message from {declare_ok.queue!r}")

    loop = asyncio.get_event_loop()
    loop.run_until_complete(main())

    assert MESSAGE is not None
    assert MESSAGE.routing_key == "hello"
    assert MESSAGE.body == b'Hello World!'

`patio`_ 和 `patio-rabbitmq`_
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**PATIO** 是 “Python Asynchronous Tasks for AsyncIO(基于异步IO的python异步任务)” 的缩写——一个易于扩展的库，用于分布式任务执行，类似于 Celery，但主要设计为支持 asyncio。

**patio-rabbitmq** 让你能够使用 *基于 RabbitMQ 的 RPC* 服务，且实现非常简单：

.. code-block:: python

   from patio import Registry, ThreadPoolExecutor
   from patio_rabbitmq import RabbitMQBroker

   rpc = Registry(project="patio-rabbitmq", auto_naming=False)

   @rpc("sum")
   def sum(*args):
       return sum(args)

   async def main():
       async with ThreadPoolExecutor(rpc, max_workers=16) as executor:
           async with RabbitMQBroker(
               executor, amqp_url="amqp://guest:guest@localhost/",
           ) as broker:
               await broker.join()

调用方可以像这样编写：

.. code-block:: python

    import asyncio
    from patio import NullExecutor, Registry
    from patio_rabbitmq import RabbitMQBroker

    async def main():
        async with NullExecutor(Registry(project="patio-rabbitmq")) as executor:
            async with RabbitMQBroker(
                executor, amqp_url="amqp://guest:guest@localhost/",
            ) as broker:
                print(await asyncio.gather(
                    *[
                        broker.call("mul", i, i, timeout=1) for i in range(10)
                     ]
                ))


`FastStream`_
~~~~~~~~~~~~~~

**FastStream** 是一个强大且易于使用的 Python 库，用于构建与事件流交互的异步服务。

如果你不需要深入了解 **RabbitMQ** 的细节，可以使用更高层次的 **FastStream** 接口：

.. code-block:: python

   from faststream import FastStream
   from faststream.rabbit import RabbitBroker
   
   broker = RabbitBroker("amqp://guest:guest@localhost:5672/")
   app = FastStream(broker)
   
   @broker.subscriber("user")
   async def user_created(user_id: int):
       assert isinstance(user_id, int)
       return f"user-{user_id}: created"

   @app.after_startup
   async def pub_smth():
       assert (
           await broker.publish(1, "user", rpc=True)
       ) ==  "user-1: created"

此外，**FastStream** 通过 **pydantic** 验证消息，生成你的项目 **AsyncAPI** 规范，支持内存测试、RPC 调用等功能。

实际上，它是 **aio-pika** 之上的高层包装器，因此你可以同时利用这两个库的优势。

`python-socketio`_
~~~~~~~~~~~~~~~~~~

`Socket.IO`_ 是一种传输协议，能够实现客户端（通常是网页浏览器，但不局限于此）与服务器之间的实时双向事件驱动通信。此包提供了两种 Python 实现，分别为标准和 asyncio 版本。

此外，此包还适合通过 **aio-pika** 适配器构建基于 **RabbitMQ** 的消息服务：

.. code-block:: python

   import socketio
   from aiohttp import web

   sio = socketio.AsyncServer(client_manager=socketio.AsyncAioPikaManager())
   app = web.Application()
   sio.attach(app)

   @sio.event
   async def chat_message(sid, data):
       print("message ", data)

   if __name__ == '__main__':
       web.run_app(app)

客户端可以通过以下方式调用 `chat_message`:

.. code-block:: python

   import asyncio
   import socketio

   sio = socketio.AsyncClient()

   async def main():
       await sio.connect('http://localhost:8080')
       await sio.emit('chat_message', {'response': 'my response'})

   if __name__ == '__main__':
       asyncio.run(main())

`taskiq`_ 和 `taskiq-aio-pika`_
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Taskiq** 是一个用于 Python 的异步分布式任务队列。该项目受到大型项目如 Celery 和 Dramatiq 的启发，但 Taskiq 可以发送和运行同步与异步函数。

该库还为运行任务提供了 **aio-pika** 代理。

.. code-block:: python

   from taskiq_aio_pika import AioPikaBroker

   broker = AioPikaBroker()

   @broker.task
   async def test() -> None:
       print("nothing")

   async def main():
       await broker.startup()
       await test.kiq()

`Rasa`_
~~~~~~~

拥有超过 2500 万次下载，Rasa Open Source 是构建聊天和语音 AI 助手的最流行的开源框架。

使用 **Rasa**，你可以在以下平台上构建上下文助手：

* Facebook Messenger
* Slack
* Google Hangouts
* Webex Teams
* Microsoft Bot Framework
* Rocket.Chat
* Mattermost
* Telegram
* Twilio

你还可以创建自定义的对话渠道或语音助手，如：

* Alexa Skills
* Google Home Actions

**Rasa** 帮助你构建能够进行多层次对话的上下文助手，实现丰富的互动。为了让人类与上下文助手进行有意义的交流，助手需要能够利用上下文，基于之前讨论的内容进行扩展——**Rasa** 使你能够以可扩展的方式构建能够实现这一目标的助手。

它还使用 **aio-pika** 来与 **RabbitMQ** 深度交互！

版本控制
==========

本软件遵循 `语义化版本控制`_


.. _Semantic Versioning: http://semver.org/
.. _语义化版本控制: http://semver.org/
.. _faststream: https://github.com/airtai/faststream
.. _patio: https://github.com/patio-python/patio
.. _patio-rabbitmq: https://github.com/patio-python/patio-rabbitmq
.. _Socket.IO: https://socket.io/
.. _python-socketio: https://python-socketio.readthedocs.io/en/latest/intro.html
.. _taskiq: https://github.com/taskiq-python/taskiq
.. _taskiq-aio-pika: https://github.com/taskiq-python/taskiq-aio-pika
.. _Rasa: https://rasa.com/docs/rasa/
