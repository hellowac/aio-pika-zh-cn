.. _issue: https://github.com/mosquito/aio-pika/issues
.. _pull request: https://github.com/mosquito/aio-pika/compare
.. _aio-pika: https://github.com/mosquito/aio-pika
.. _official tutorial: https://www.rabbitmq.com/tutorials/tutorial-one-python.html
.. _官方教程: https://www.rabbitmq.com/tutorials/tutorial-one-python.html
.. _asyncio: https://docs.python.org/3/library/asyncio.html
.. _asyncio tutorial: https://pymotw.com/3/asyncio/coroutines.html
.. _asyncio 教程: https://pymotw.com/3/asyncio/coroutines.html
.. _introduction:


入门
============

.. warning::

   这是来自 `官方教程`_ 的 port 的 beta 版本。如果你发现错误，请创建 `issue`_ 或 `pull request`_ 。

   预计你对 `asyncio`_ 的基本知识有所了解。不过，以下示例可以按原样运行。你可以自由下载并测试这些示例，而无需进行任何更改（前提是你的 RabbitMQ 安装允许“guest”用户访问）。

   否则，我们建议阅读 `asyncio 教程`_ 。


.. note::

   **前置条件**

   本教程假设 RabbitMQ 已安装并在本地主机的标准端口 (`5672`) 上运行。如果你使用不同的主机、端口或凭据，连接设置需要进行相应调整。

   .. _installed: https://www.rabbitmq.com/download.html

   **寻求帮助的方式**

   如果你在学习本教程时遇到问题，可以通过邮件列表 `联系我们`_ 。

   .. _contact us: https://groups.google.com/forum/#!forum/rabbitmq-users
   .. _联系我们: https://groups.google.com/forum/#!forum/rabbitmq-users


RabbitMQ 是一个消息中介。其基本理念非常简单:它接受并转发消息。你可以把它想象成一个邮局:当你把邮件放入邮筒时，你可以很确信邮递员最终会将邮件送达收件人。用这个比喻来说，RabbitMQ 就是一个邮筒、一个邮局和一个邮递员。

RabbitMQ 与邮局之间的主要区别在于，它不处理纸张，而是接受、存储和转发二进制数据块 —— 消息(messages)。

RabbitMQ 和消息传递领域使用了一些术语。

* 生产意味着发送。发送消息的程序被称为生产者(producer)。

我们将用 "P" 来表示它:

.. image:: /_static/tutorial/producer.svg
   :align: center

* 队列是邮箱的名称。它存在于 RabbitMQ 中。尽管消息在 RabbitMQ 和你的应用程序之间流动，但它们只能存储在队列中。队列没有限制，可以存储任意数量的消息——本质上它是一个无限缓冲区。多个生产者可以发送消息到同一个队列，而多个消费者可以尝试从同一个队列接收数据。

队列将如下所示，队列的名称在其上方：

.. image:: /_static/tutorial/queue.svg
   :align: center

* 消费(Consuming)的含义类似于接收。消费者是一个主要等待接收消息的程序。

在我们的图示中用 "C" 表示它：

.. image:: /_static/tutorial/consumer.svg
   :align: center

.. note::
   
   请注意，生产者、消费者和中介不必位于同一台机器上；实际上，在大多数应用程序中，它们并不在同一台机器上。


Hello World!
++++++++++++

.. note::
   
   使用 `aio-pika`_ 异步python客户端

我们的“Hello World”示例不会太复杂——我们将发送一条消息，接收它并在屏幕上打印出来。为此，我们需要两个程序：一个发送消息，另一个接收并打印消息。

我们的整体设计将如下所示：

.. image:: /_static/tutorial/python-one-overall.svg
   :align: center

生产者将消息发送到 "hello" 队列，消费者从该队列接收消息。

.. note::

   **RabbitMQ 库**

   RabbitMQ 使用 AMQP 0.9.1，这是一种开放的通用消息传递协议。RabbitMQ 有 `许多不同语言`_ 的客户端。在本教程系列中，我们将使用 `aio-pika`_ 。要安装它，你可以使用 `pip`_ 包管理工具。

   .. _许多不同语言: https://www.rabbitmq.com/devtools.html
   .. _pip: https://pip.pypa.io/en/stable/quickstart/


发送
+++++++

.. image:: /_static/tutorial/sending.svg
   :align: center

我们的第一个程序 *send.py* 将向队列发送一条消息。我们首先需要做的是与 RabbitMQ 服务器建立连接。


.. literalinclude:: examples/1-introduction/send.py
   :language: python
   :lines: 5-12

我们现在已连接到本地机器上的中介——因此使用 localhost。如果我们想连接到其他机器上的中介，只需在此处指定其名称或 IP 地址。

接下来，在发送消息之前，我们需要确保接收队列存在。如果我们将消息发送到不存在的位置，RabbitMQ 将直接丢弃该消息。让我们创建一个将接收消息的队列，命名为 *hello*：

.. literalinclude:: examples/1-introduction/send.py
   :language: python
   :lines: 14-15

此时我们已准备好发送消息。我们的第一条消息将仅包含字符串 "Hello World!"，我们希望将其发送到 hello 队列。

在 RabbitMQ 中，消息不能直接发送到队列，它总是需要通过交换机传递。但我们不必纠结于这些细节 —— 你可以在 :ref:`本教程的第三部分 <publish-subscribe>` 中阅读更多关于交换机的信息。
我们现在需要知道的是如何使用一个由空字符串标识的默认交换机。这个交换机是特殊的 —— 它允许我们准确指定消息应该发送到哪个队列。队列名称需要在 *routing_key* 参数中指定：


.. literalinclude:: examples/1-introduction/send.py
   :language: python
   :lines: 17-21

在退出程序之前，我们需要确保网络缓冲区已刷新，并且我们的消息已实际发送到 RabbitMQ。我们可以通过温和地关闭连接来实现。在这个示例中使用了异步上下文管理器。

.. literalinclude:: examples/1-introduction/send.py
   :language: python
   :lines: 10-12

.. note::

    *发送失败！*

    如果这是你第一次使用 RabbitMQ，并且你没有看到“Sent(已发送)”消息，可能会让你感到困惑，想知道可能出错的地方。
    也许中介在没有足够可用磁盘空间的情况下启动（默认情况下至少需要 1Gb 的空闲空间），因此拒绝接收消息。
    请检查中介日志文件以确认，并在必要时减少限制。 `配置文件文档`_ 将告诉你如何设置 *disk_free_limit*。

    .. _配置文件文档: http://www.rabbitmq.com/configure.html#config-items


接收
+++++++++

.. image:: /_static/tutorial/receiving.svg
   :align: center

我们的第二个程序 *receive.py* 将从队列接收消息并在屏幕上打印出来。

首先，我们需要连接到 RabbitMQ 服务器。连接 RabbitMQ 的代码与之前相同。

下一步，就像之前一样，我们需要确保队列存在。使用 *queue_declare* 创建队列是幂等的 —— 我们可以多次运行该命令，只有一个队列会被创建。

.. literalinclude:: examples/1-introduction/receive.py
   :language: python
   :lines: 22-28

你可能会问，为什么我们再次声明队列 —— 我们在之前的代码中已经声明过。
我们可以避免这样做，如果我们确定队列已经存在。例如，如果先运行了 *send.py* 程序。
但我们还不确定哪个程序应该先运行。在这种情况下，最好在两个程序中都重复声明队列。


.. note::
    
    **列出队列**

    你可能希望查看 RabbitMQ 中有哪些队列以及它们中的消息数量。作为特权用户，你可以使用 rabbitmqctl 工具做到这一点：

    ::

        $ sudo rabbitmqctl list_queues
        Listing queues ...
        hello    0
        ...done.
        (omit sudo on Windows)

从队列接收消息很简单。它通过将一个 `回调函数` (callback function) 订阅到队列或使用 `simple get` 来实现。

每当我们收到一条消息时， `aio-pika`_ 库会调用这个回调函数。在我们的例子中，这个函数将打印消息的内容。

.. literalinclude:: examples/1-introduction/receive.py
   :language: python
   :pyobject: on_message

接下来，我们需要告诉 RabbitMQ 这个特定的回调函数应该接收来自 hello 队列的消息：

.. literalinclude:: examples/1-introduction/receive.py
   :language: python
   :pyobject: main

*no_ack* 参数将在 :ref:`后面介绍 <work-queues>`。

综合起来
+++++++++++++++++++++++

:download:`send.py <examples/1-introduction/send.py>` 的完整代码:

.. literalinclude:: examples/1-introduction/send.py
   :language: python

:download:`receive.py <examples/1-introduction/receive.py>` 的完整代码:

.. literalinclude:: examples/1-introduction/receive.py
   :language: python

现在我们可以在终端中尝试我们的程序。首先，让我们使用 *send.py* 程序发送一条消息::

     $ python send.py
     [x] Sent 'Hello World!'

生产者程序 *send.py* 在每次运行后都会停止。接下来，让我们接收这条消息::

     $ python receive.py
     [x] Received message IncomingMessage:{
      "app_id": null,
      "body_size": 12,
      "cluster_id": null,
      "consumer_tag": "ctag1.11fa33f5f4fa41f6a6488648181656e0",
      "content_encoding": null,
      "content_type": null,
      "correlation_id": "b'None'",
      "delivery_mode": 1,
      "delivery_tag": 1,
      "exchange": "",
      "expiration": null,
      "headers": null,
      "message_id": null,
      "priority": null,
      "redelivered": false,
      "reply_to": null,
      "routing_key": "hello",
      "synchronous": false,
      "timestamp": null,
      "type": "None",
      "user_id": null
     }
     Message body is: b'Hello World!'

太好了！我们成功通过 RabbitMQ 发送了第一条消息。如你所见，*receive.py* 程序不会退出。它将保持准备接收进一步的消息，可以使用 **Ctrl-C** 中断。

尝试在新终端中再次运行 *send.py*。

我们已经学习了如何从命名队列发送和接收消息。现在是时候继续到 :ref:`第 2 部分 <work-queues>` 并构建一个简单的工作队列。


.. note::

    该材料来自于 **rabbitmq.org** 的 `官方教程`_。
