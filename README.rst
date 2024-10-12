.. _documentation: https://aio-pika.readthedocs.org/
.. _adopted official RabbitMQ tutorial: https://aio-pika.readthedocs.io/en/latest/rabbitmq-tutorial/1-introduction.html


aio-pika
========

.. image:: https://readthedocs.org/projects/aio-pika/badge/?version=latest
    :target: https://aio-pika.readthedocs.org/
    :alt: ReadTheDocs

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


一个基于 `aiormq`_ 的封装，适用于 asyncio 和人类使用。

请查看`文档`_中的示例和教程。

如果你是 RabbitMQ 的新手，请从 `官方推荐的 RabbitMQ 教程`_ 开始。

.. _aiormq: http://github.com/mosquito/aiormq/

.. note::
    自版本 ``5.0.0`` 起，该库不再使用 ``pika`` 作为 AMQP 连接器。  ``5.0.0`` 以下的版本包含或需要 ``pika`` 的源代码。

.. note::
    版本 7.0.0 对 API 进行了重大更改，迁移提示请参阅 CHANGELOG.md。


功能
--------

* 完全异步的 API。
* 面向对象的 API。
* 使用 `connect_robust` 进行透明的自动重连并完全恢复状态（例如，已声明的队列或交换机，消费状态和绑定）。
* 兼容 Python 3.7+。
* 对于 Python 3.5 用户，可以通过 `aio-pika<7` 获取 aio-pika。
* 透明的 `publisher confirms`_ 支持。
* 支持 `事务`_ 。
* 完整的类型提示覆盖。


.. _事务: https://www.rabbitmq.com/semantics.html
.. _publisher confirms: https://www.rabbitmq.com/confirms.html


安装
------------

.. code-block:: shell

    pip install aio-pika


使用样例
-------------

Simple 消费者:

.. code-block:: python

    import asyncio
    import aio_pika
    import aio_pika.abc


    async def main(loop):
        # 也可以使用给定的参数进行连接。
        # aio_pika.connect_robust(host="host", login="login", password="password")
        # 您只能选择一个选项来创建连接，url 或基于 kw 的参数。
        connection = await aio_pika.connect_robust(
            "amqp://guest:guest@127.0.0.1/", loop=loop
        )

        async with connection:
            queue_name = "test_queue"

            # 创建通道
            channel: aio_pika.abc.AbstractChannel = await connection.channel()

            # 声明队列
            queue: aio_pika.abc.AbstractQueue = await channel.declare_queue(
                queue_name,
                auto_delete=True
            )

            async with queue.iterator() as queue_iter:
                # __aexit__ 之后取消消费(consuming)
                async for message in queue_iter:
                    async with message.process():
                        print(message.body)

                        if queue.name in message.body.decode():
                            break


    if __name__ == "__main__":
        loop = asyncio.get_event_loop()
        loop.run_until_complete(main(loop))
        loop.close()

Simple 发布者:

.. code-block:: python

    import asyncio
    import aio_pika
    import aio_pika.abc


    async def main(loop):
        # 显式类型注解
        connection: aio_pika.RobustConnection = await aio_pika.connect_robust(
            "amqp://guest:guest@127.0.0.1/", loop=loop
        )

        routing_key = "test_queue"

        channel: aio_pika.abc.AbstractChannel = await connection.channel()

        await channel.default_exchange.publish(
            aio_pika.Message(
                body='Hello {}'.format(routing_key).encode()
            ),
            routing_key=routing_key
        )

        await connection.close()


    if __name__ == "__main__":
        loop = asyncio.get_event_loop()
        loop.run_until_complete(main(loop))
        loop.close()


获取单个消息样例:

.. code-block:: python

    import asyncio
    from aio_pika import connect_robust, Message


    async def main(loop):
        connection = await connect_robust(
            "amqp://guest:guest@127.0.0.1/",
            loop=loop
        )

        queue_name = "test_queue"
        routing_key = "test_queue"

        # 创建通道
        channel = await connection.channel()

        # 声明交换机
        exchange = await channel.declare_exchange('direct', auto_delete=True)

        # 声明队列
        queue = await channel.declare_queue(queue_name, auto_delete=True)

        # 绑定队列
        await queue.bind(exchange, routing_key)

        await exchange.publish(
            Message(
                bytes('Hello', 'utf-8'),
                content_type='text/plain',
                headers={'foo': 'bar'}
            ),
            routing_key
        )

        # 接收消息
        incoming_message = await queue.get(timeout=5)

        # 确认消息
        await incoming_message.ack()

        await queue.unbind(exchange, routing_key)
        await queue.delete()
        await connection.close()


    if __name__ == "__main__":
        loop = asyncio.get_event_loop()
        loop.run_until_complete(main(loop))

`文档`_ 中有更多样例以及RabbitMQ指南.

同样参考
==========

`aiormq`_
---------

`aiormq` 是一个纯 Python AMQP 客户端库。它位于 **aio-pika** 的底层，当您真正喜欢使用协议底层时可能会用到它。

以下示例演示了用户 API。

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
--------------------------------------

**PATIO** 是 Python Asynchronous Tasks for AsyncIO 的缩写，是一个易于扩展的库，用于分布式任务执行，类似于 Celery，但其主要设计方法是面向 asyncio。

**patio-rabbitmq** 让你能够通过非常简单的实现使用 *基于 RabbitMQ 的 RPC* 服务：

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

以及调用侧可以这样写:

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
---------------

**FastStream** 是一个功能强大且易于使用的 Python 库，用于构建与事件流交互的异步服务。

如果你不需要深入了解 **RabbitMQ** 的细节，你可以使用更高层的 **FastStream** 接口：

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

此外，**FastStream** 通过 **pydantic** 验证消息，生成项目的 **AsyncAPI** 规范，支持内存测试、RPC 调用等功能。

实际上，它是 **aio-pika** 之上的高级封装，因此你可以同时利用这两个库的优势。

`python-socketio`_
------------------

`Socket.IO`_ 是一种传输协议，使客户端（通常是但不限于网页浏览器）与服务器之间能够进行实时的双向事件通信。该包提供了客户端和服务器的 Python 实现，分别有标准版和 asyncio 版。

此外，该包还适用于通过 **aio-pika** 适配器在 **RabbitMQ** 上构建消息服务：

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

客户端可以通过以下方式调用 `chat_message`：

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
----------------------------------------

**Taskiq** 是一个用于 Python 的异步分布式任务队列。该项目受到 Celery 和 Dramatiq 等大型项目的启发。但 Taskiq 可以发送和运行同步与异步函数。

该库还为你提供了 **aio-pika** 代理来运行任务。

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
-------

Rasa Open Source 是构建聊天和基于语音的 AI 助手最受欢迎的开源框架，下载量超过 2500 万次。

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

以及你自己的自定义对话渠道或语音助手，如：

* Alexa Skills
* Google Home Actions

**Rasa** 帮助你构建能够进行多层次对话的上下文助手，实现丰富的互动。为了让人类与上下文助手进行有意义的交流，助手需要能够利用上下文来扩展之前讨论过的内容——**Rasa** 使你能够构建能够以可扩展方式实现这一点的助手。

它还使用 **aio-pika** 与 **RabbitMQ** 进行深层交互！

版本
==========

该软件遵循 `语义版本控制`_。


参与贡献
----------------

构建开发环境
__________________________________

克隆项目:

.. code-block:: shell

    git clone https://github.com/mosquito/aio-pika.git
    cd aio-pika

创建一个针对 `aio-pika`_ 的虚拟环境:

.. code-block:: shell

    python3 -m venv env
    source env/bin/activate

安装针对 `aio-pika`_ 的依赖:

.. code-block:: shell

    pip install -e '.[develop]'


运行测试
_____________

**注意：要在本地运行测试，你需要运行一个 RabbitMQ 实例，使用默认的用户名/密码（guest/guest）和端口（5672）。**

Makefile 提供了一个命令来运行适当的 RabbitMQ Docker 镜像：

.. code-block:: bash

    make rabbitmq

要测试请运行:

.. code-block:: bash

    make test


编辑文档
_____________________

要在浏览器中快速查看文档，请尝试：

.. code-block:: bash

    nox -s docs -- serve

创建合并请求
______________________

翻译：

请随时提交拉取请求，但你应该描述你的使用案例并添加一些示例。

更改应遵循一些简单的规则：

* 当你的更改破坏公共 API 时，必须增加主版本号。
* 当你的更改对公共 API 是安全的（例如，添加了一个具有默认值的参数）时。
* 你必须添加测试用例（参见 `tests/` 文件夹）。
* 你必须添加文档字符串。
* 欢迎将自己添加到 `“感谢”`_ 部分。


.. _"thank's to" section: https://github.com/mosquito/aio-pika/blob/master/docs/source/index.rst#thanks-for-contributing
.. _“感谢”: https://github.com/mosquito/aio-pika/blob/master/docs/source/index.rst#thanks-for-contributing
.. _Semantic Versioning: http://semver.org/
.. _语义版本控制: http://semver.org/
.. _aio-pika: https://github.com/mosquito/aio-pika/
.. _faststream: https://github.com/airtai/faststream
.. _patio: https://github.com/patio-python/patio
.. _patio-rabbitmq: https://github.com/patio-python/patio-rabbitmq
.. _Socket.IO: https://socket.io/
.. _python-socketio: https://python-socketio.readthedocs.io/en/latest/intro.html
.. _taskiq: https://github.com/taskiq-python/taskiq
.. _taskiq-aio-pika: https://github.com/taskiq-python/taskiq-aio-pika
.. _Rasa: https://rasa.com/docs/rasa/
