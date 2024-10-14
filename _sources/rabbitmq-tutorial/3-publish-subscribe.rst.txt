.. _issue: https://github.com/mosquito/aio-pika/issues
.. _pull request: https://github.com/mosquito/aio-pika/compare
.. _aio-pika: https://github.com/mosquito/aio-pika
.. _官方教程: https://www.rabbitmq.com/tutorials/tutorial-three-python.html
.. _publish-subscribe:

发布/订阅
=================

.. warning::

    这是从 `官方教程`_ 移植的测试版本。如果你发现了错误，请为我创建 `issue`_ 或 `pull request`_。


.. note::
    
    使用 `aio-pika`_ 异步 Python 客户端。

.. note::

    **前提条件**

    本教程假设 RabbitMQ `已安装`_ 并在本地以标准端口（`5672`）运行。
    如果你使用的是不同的主机、端口或凭据，则需要调整连接设置。

    .. _已安装: https://www.rabbitmq.com/download.html

    **寻求帮助的途径**

    如果在完成本教程时遇到困难，可以通过邮件列表 `联系我们`_ 。

    .. _联系我们: https://groups.google.com/forum/#!forum/rabbitmq-users

在 :ref:`上一个教程 <work-queues>` 中，我们创建了一个工作队列。工作队列的假设是每个任务会被分配给一个工人。在这一部分，我们将做一些完全不同的事情——我们将消息传递给多个消费者。这个模式被称为“发布/订阅”。

为了说明这个模式，我们将构建一个简单的日志系统。它将由两个程序组成——第一个程序将发出日志消息，第二个程序将接收并打印这些消息。

在我们的日志系统中，每个正在运行的接收器程序副本都将收到消息。这样我们就可以运行一个接收器并将日志写入磁盘，同时也可以运行另一个接收器并在屏幕上查看日志。

本质上，发布的日志消息将广播给所有接收器。


交换机(Exchanges)
+++++++++++++++++

在教程的前几部分中，我们将消息发送到队列并从队列接收消息。现在是时候介绍 Rabbit 中的完整消息传递模型了。

让我们快速回顾一下前面的教程内容：

* 生产者是发送消息的用户应用程序。
* 队列是存储消息的缓冲区。
* 消费者是接收消息的用户应用程序。

RabbitMQ 消息传递模型的核心思想是生产者从不直接将消息发送到队列。实际上，生产者通常甚至不知道消息是否会被传递到某个队列。

相反，生产者只能将消息发送到交换机。交换机本身非常简单。一方面它从生产者接收消息，另一方面它将消息推送到队列中。交换机必须确切地知道如何处理接收到的消息。它应该将消息添加到某个特定队列吗？还是应该添加到多个队列？或者应该丢弃？这些规则由交换机的类型来定义。

.. image:: /_static/tutorial/exchanges.svg
   :align: center

有几种可用的交换机类型：`DIRECT`、`TOPIC`、`HEADERS` 和 `FANOUT`（参见 :class:`aio_pika.ExchangeType` ）。
我们将重点关注最后一种——扇出型（fanout）。让我们创建一个这种类型的交换机，并将其命名为 `logs`：

.. literalinclude:: examples/3-publish-subscribe/emit_log.py
   :language: python
   :lines: 15-17

扇出型交换机非常简单。顾名思义，它会将接收到的所有消息广播到所有它知道的队列。这正是我们日志系统所需要的。

.. note::

    **列出交换机**

    要列出服务器上的交换机，你可以运行非常有用的 rabbitmqctl 命令::

        $ sudo rabbitmqctl list_exchanges
        Listing exchanges ...
        logs      fanout
        amq.direct      direct
        amq.topic       topic
        amq.fanout      fanout
        amq.headers     headers
        ...done.

    在这个列表中，有一些 `amq.*` 交换机以及默认的（未命名的）交换机。这些交换机是默认创建的，但目前你可能不需要使用它们。

    **无名交换机**

    在教程的前几部分中，我们对交换机一无所知，但仍然能够将消息发送到队列。这是因为我们使用了默认交换机，我们通过空字符串 ("") 来标识它。

    回忆一下我们之前如何发布消息：

    .. code-block:: python

        await channel.default_exchange.publish(
            Message(message_body),
            routing_key='hello',
        )

    `exchange` 参数是交换机的名称。空字符串表示默认或无名交换机：消息将被路由到由 `routing_key` 指定名称的队列（如果存在）。


现在，我们可以将消息发布到我们命名的交换机：

.. literalinclude:: examples/3-publish-subscribe/emit_log.py
   :language: python
   :lines: 19-29


临时队列  
++++++++++++++++

你可能还记得之前我们使用的是有指定名称的队列（还记得 `hello` 和 `task_queue` 吗？）。能够命名队列对我们至关重要——我们需要将工人指向同一个队列。当你想在生产者和消费者之间共享队列时，给队列命名是很重要的。

但对于我们的日志系统来说情况不同。我们想听到所有的日志消息，而不仅仅是其中的一部分。我们也只关心当前流动的消息，而不是旧的消息。为了解决这个问题，我们需要两样东西。

首先，每当我们连接到 Rabbit 时，我们需要一个全新的、空的队列。为此，我们可以创建一个随机名称的队列，或者更好的是——让服务器为我们选择一个随机队列名称。我们可以通过不提供 `declare_queue` 的队列参数来实现这一点：

.. code-block:: python

    queue = await channel.declare_queue()

其次，一旦我们断开消费者连接，该队列应该被删除。为此有一个独占标志：

.. literalinclude:: examples/3-publish-subscribe/receive_logs.py
   :language: python
   :lines: 26

绑定  
++++++++

.. image:: /_static/tutorial/bindings.svg  
   :align: center

我们已经创建了一个扇出交换机和一个队列。现在我们需要告诉交换机将消息发送到我们的队列。交换机与队列之间的这种关系称为绑定。

.. literalinclude:: examples/3-publish-subscribe/receive_logs.py  
   :language: python  
   :lines: 21-29

从现在开始，日志交换机将把消息添加到我们的队列中。

.. note::

    **列出绑定**

    你可以使用 `rabbitmqctl list_bindings` 来列出现有的绑定，没错，就是它。


综合起来  
+++++++++++++++++++++++

.. image:: /_static/tutorial/python-three-overall.svg  
   :align: center

发送日志消息的生产者程序与之前的教程没有太大区别。最重要的变化是我们现在希望将消息发布到我们的 `logs` 交换机，而不是无名交换机。发送时我们需要提供 `routing_key`，但对于扇出型交换机，该值会被忽略。以下是 :download:`emit_log.py <examples/3-publish-subscribe/emit_log.py>` 脚本的代码：

.. literalinclude:: examples/3-publish-subscribe/emit_log.py  
   :language: python

如你所见，建立连接后，我们声明了交换机。这一步是必要的，因为向不存在的交换机发布消息是被禁止的。

如果没有队列绑定到交换机，消息将会丢失，但这对我们来说没问题；如果没有消费者在监听，我们可以安全地丢弃消息。

以下是 :download:`receive_logs.py <examples/3-publish-subscribe/receive_logs.py>` 脚本的代码：

.. literalinclude:: examples/3-publish-subscribe/receive_logs.py  
   :language: python

我们完成了。如果你想将日志保存到文件中，只需打开控制台并输入::

    $ python receive_logs.py > logs_from_rabbit.log

如果你想在屏幕上查看日志，打开一个新的终端并运行::

    $ python receive_logs.py

当然，要发送日志，请输入::

    $ python emit_log.py

使用 *rabbitmqctl list_bindings* 你可以验证代码确实按照我们预期创建了绑定和队列。运行两个 *receive_logs.py* 程序时，你应该看到类似的内容::

    $ sudo rabbitmqctl list_bindings
    Listing bindings ...
    logs    exchange        amq.gen-JzTY20BRgKO-HjmUJj0wLg  queue           []
    logs    exchange        amq.gen-vso0PVvyiRIL2WoV3i48Yg  queue           []
    ...done.

结果的解释很简单：来自 `logs` 交换机的数据被发送到两个由服务器分配名称的队列。这正是我们所期望的。

要了解如何监听一部分消息，让我们继续学习 :ref:`教程 4 <routing>`。

.. note::

    本材料采用自 **rabbitmq.org** 上的 `官方教程`_ 。
