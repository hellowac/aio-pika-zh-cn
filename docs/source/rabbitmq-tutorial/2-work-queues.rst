.. _issue: https://github.com/mosquito/aio-pika/issues
.. _pull request: https://github.com/mosquito/aio-pika/compare
.. _aio-pika: https://github.com/mosquito/aio-pika
.. _official tutorial: https://www.rabbitmq.com/tutorials/tutorial-two-python.html
.. _官方教程: https://www.rabbitmq.com/tutorials/tutorial-two-python.html
.. _work-queues:

工作队列
===========

.. warning::

   这是从 `官方教程`_ 移植的测试版。如果你发现错误，请为我创建 `issue`_ 或 `pull request`_。

   该实现是官方教程的一部分。
   自版本 1.7.0 起， `aio-pika`_ 提供了 :ref:`patterns 子模块 <patterns-worker>`。

   你可以在实际项目中使用 :class:`aio_pika.patterns.Master`。

.. note::

   使用 `aio-pika`_ 异步 Python 客户端。

.. note::
   
   **先决条件**

   本教程假设 RabbitMQ `已安装`_ 并在标准端口 (`5672`) 上在本地主机上运行。
   如果你使用不同的主机、端口或凭据，需要调整连接设置。

   .. _已安装: https://www.rabbitmq.com/download.html

   **寻求帮助**

   如果你在完成本教程时遇到问题，可以通过邮件列表 `联系我们`_。

   .. _联系我们: https://groups.google.com/forum/#!forum/rabbitmq-users

.. image:: /_static/tutorial/python-two.svg
   :align: center

在 :ref:`第一个教程 <introduction>` 中，我们编写了从命名队列发送和接收消息的程序。在本教程中，我们将创建一个工作队列，用于在多个工作者之间分配耗时的任务。

工作队列（也称为 *任务队列(Task Queues)*）的主要思想是避免立即执行资源密集型任务并等待其完成。相反，我们将任务安排在稍后完成。我们将任务封装为消息并发送到队列。后台运行的工作进程将弹出任务并最终执行工作。当运行多个工作者时，任务将被它们共享。

这个概念在 Web 应用程序中特别有用，因为在短暂的 HTTP 请求窗口内处理复杂任务几乎是不可能的。

准备工作
+++++++++++

在本教程的前一部分中，我们发送了一条包含 `"Hello World!"` 的消息。现在我们将发送表示复杂任务的字符串。由于我们没有真实的任务（例如要调整大小的图像或要渲染的 PDF 文件），我们将通过使用 `asyncio.sleep()` 函数来假装自己很忙。我们将字符串中的点的数量视为其复杂性；每个点将代表一秒钟的“工作”。例如，由 `Hello...` 描述的虚假任务将耗时三秒。

我们将稍微修改之前示例中的 *send.py* 代码，以允许从命令行发送任意消息。这个程序将任务调度到我们的工作队列中，因此我们将其命名为 *new_task.py*:

.. literalinclude:: examples/2-work-queues/new_task_initial.py
   :language: python
   :pyobject: main

我们的旧 *receive.py* 脚本也需要一些更改：它需要为消息体中的每个点假装工作一秒。它将从队列中弹出消息并执行任务，因此我们将其称为 *tasks_worker.py*:

.. code-block:: python

    async def on_message(message: IncomingMessage):
        print(" [x] Received %r" % message.body)
        await asyncio.sleep(message.body.count(b'.'))
        print(" [x] Done")


轮询调度(Round-robin dispatching)
++++++++++++++++++++++++++++++++++

使用任务队列的一个优点是能够轻松地并行化工作。如果我们积累了大量待处理工作，可以简单地增加更多工作者，从而轻松扩展。

首先，让我们尝试同时运行两个 *tasks_worker.py* 脚本。它们将从队列中获取消息，但具体是如何获取的呢？让我们来看看。

你需要打开三个控制台。两个将运行 ``tasks_worker.py`` 脚本。这两个控制台将是我们的两个消费者 - C1 和 C2。

::

    shell1$ python tasks_worker.py
    [*] Waiting for messages. To exit press CTRL+C

::

    shell2$ python tasks_worker.py
    [*] Waiting for messages. To exit press CTRL+C

在第三个控制台中，我们将发布新任务。启动消费者后，可以发布几条消息::

    shell3$ python new_task.py First message.
    shell3$ python new_task.py Second message..
    shell3$ python new_task.py Third message...
    shell3$ python new_task.py Fourth message....
    shell3$ python new_task.py Fifth message.....

让我们看看分发给我们工作者的消息::

    shell1$ python tasks_worker.py
     [*] Waiting for messages. To exit press CTRL+C
     [x] Received 'First message.'
     [x] Received 'Third message...'
     [x] Received 'Fifth message.....'

::

    shell2$ python tasks_worker.py
     [*] Waiting for messages. To exit press CTRL+C
     [x] Received 'Second message..'
     [x] Received 'Fourth message....'

默认情况下，RabbitMQ 会按顺序将每条消息发送到下一个消费者。平均而言，每个消费者将收到相同数量的消息。这种分发消息的方式称为轮询调度。尝试用三个或更多工作者进行测试。

消息确认
++++++++++++++++++++++

执行任务可能需要几秒钟。你可能会想，如果其中一个消费者开始执行一个长任务并且在任务未完成时崩溃，会发生什么？在我们当前的代码中，一旦RabbitMQ将消息发送给消费者，它会立即从内存中删除该消息。在这种情况下，如果你杀掉一个工作者，我们将丢失它刚处理的消息，还会失去所有分配给该特定工作者但尚未处理的消息。

但我们不希望丢失任何任务。如果一个工作者崩溃，我们希望将任务交给另一个工作者。

为了确保消息不会丢失，RabbitMQ支持消息确认。确认（acknowledgment）是从消费者发送回RabbitMQ的信号，告诉它某条消息已被接收和处理，可以安全删除。

如果一个消费者在未发送确认的情况下崩溃（例如，通道关闭、连接关闭或TCP连接丢失），RabbitMQ将理解该消息未被完全处理，并会将其重新排队。如果同时有其他消费者在线，它将迅速将消息重新投递给其他消费者。这样你可以确保即使工作者偶尔崩溃，也不会丢失任何消息。

RabbitMQ没有消息超时；当消费者崩溃时，RabbitMQ将重新投递该消息。即使处理消息的时间非常长，这也是可以的。

消息确认默认是启用的。在之前的示例中，我们通过 `no_ack=True` 标志显式地将其关闭。现在是时候移除这个标志，并在工作者完成任务后发送正确的确认了。

.. code-block:: python

    async def on_message(message: IncomingMessage):
        print(" [x] Received %r" % message.body)
        await asyncio.sleep(message.body.count(b'.'))
        print(" [x] Done")
        await message.ack()

.. code-block:: python

    # Declaring queue
        queue = await channel.declare_queue("hello")
    # Start listening the queue with name 'hello'
        await queue.consume(on_message)

或者使用特殊的上下文处理器：

.. literalinclude:: examples/2-work-queues/tasks_worker.py
   :language: python
   :lines: 7-11


如果上下文处理器捕获到异常，该消息将返回到队列中。

使用这段代码，我们可以确保即使在处理消息时使用CTRL+C杀掉工作者，也不会丢失任何消息。工作者崩溃后，所有未确认的消息将被重新投递。

.. note::
    
    **遗忘确认**

    漏掉确认是一个常见错误。这是一个简单的错误，但后果严重。当客户端退出时，消息会被重新投递（这可能看起来像随机重新投递），但RabbitMQ会消耗越来越多的内存，因为它无法释放任何未确认的消息。

    为了调试这种错误，你可以使用`rabbitmqctl`打印`messages_unacknowledged`字段::

        $ sudo rabbitmqctl list_queues name messages_ready messages_unacknowledged
        Listing queues ...
        hello    0       0
        ...done.


消息持久性
++++++++++++++++++

我们已经学习了如何确保即使消费者崩溃，任务也不会丢失。但如果RabbitMQ服务器停止运行，我们的任务仍然会丢失。

当RabbitMQ退出或崩溃时，它会忘记队列和消息，除非你告诉它不要丢失。为了确保消息不丢失，我们需要将队列和消息都标记为持久。

首先，我们需要确保RabbitMQ永远不会丢失我们的队列。为此，我们需要将其声明为*持久*：

.. code-block:: python

   queue = await channel.declare_queue("hello", durable=True)


虽然这条命令本身是正确的，但在我们的设置中无法工作。这是因为我们已经定义了一个名为hello的非持久队列。RabbitMQ不允许你用不同的参数重新定义一个已存在的队列，任何尝试这样做的程序都会返回错误。但有一个快速的解决方法——我们可以声明一个不同名称的队列，例如task_queue：

.. literalinclude:: examples/2-work-queues/tasks_worker.py
   :language: python
   :lines: 23-27

这个`queue_declare`的更改需要在生产者和消费者代码中都应用。

此时我们可以确定，即使RabbitMQ重启，task_queue队列也不会丢失。
现在，我们需要将我们的消息标记为持久，通过提供一个值为 `PERSISTENT` 的 `delivery_mode` 属性（参见枚举：:class:`aio_pika.DeliveryMode` ）。

.. literalinclude:: examples/2-work-queues/new_task.py
   :language: python
   :pyobject: main

.. note::

   **关于消息持久性**

   将消息标记为持久并不完全保证消息不会丢失。
   虽然这告诉RabbitMQ将消息保存到磁盘，但在RabbitMQ接受消息后尚未保存之前，仍然存在短暂的时间窗口。
   此外，RabbitMQ并不会对每条消息执行 fsync(2)操作 —— 它可能只是将消息保存到缓存中，而并非真正写入磁盘。
   持久性保证并不强，但对于我们简单的任务队列来说已经足够。如果你需要更强的保证，可以使用 `发布者确认`_ 。

   `aio-pika`_ 支持 `发布者确认`_ 功能，开箱即用。

   .. _发布者确认: https://www.rabbitmq.com/confirms.html


公平分配
+++++++++++++

你可能已经注意到，消息的分发仍然不完全符合我们的期望。
例如，在有两个工作者的情况下，当所有奇数消息都很重而偶数消息很轻时，一个工作者将会一直忙碌，
而另一个几乎没有任何工作。RabbitMQ对此一无所知，仍然会平均分配消息。

这种情况发生是因为RabbitMQ在消息进入队列时就分发消息。
它并不会考虑消费者未确认的消息数量，而是盲目地将每第n条消息分发给第n个消费者。


.. image:: /_static/tutorial/prefetch-count.svg
   :align: center


为了改善这一点，我们可以使用 `basic.qos` 方法，并设置 `prefetch_count=1` 。
这告诉RabbitMQ不要一次性给一个工作者超过一条消息。
换句话说，在一个工作者处理并确认之前的消息之前，不要向其分发新消息。相反，它会将消息分发给下一个没有在忙碌的工作者。

.. literalinclude:: examples/2-work-queues/tasks_worker.py
   :language: python
   :lines: 18-21


.. note::

    **关于队列大小的说明**

    如果所有工作者都在忙，你的队列可能会填满。你需要密切关注这一点，并可能考虑增加更多工作者，或者采取其他策略。


综合起来
+++++++++++++++++++++++

我们 :download:`new_task.py <examples/2-work-queues/new_task.py>` 脚本的最终代码:

.. literalinclude:: examples/2-work-queues/new_task.py
   :language: python

以及我们的 :download:`tasks_worker.py <examples/2-work-queues/tasks_worker.py>`:

.. literalinclude:: examples/2-work-queues/tasks_worker.py
   :language: python

通过使用消息确认和 `prefetch_count`，你可以设置一个工作队列。而持久性选项确保即使RabbitMQ重启，任务也不会丢失。

现在我们可以继续学习 :ref:`第三个教程 <publish-subscribe>`，了解如何将相同的消息传递给多个消费者。


.. note::

    这部分内容取自  **rabbitmq.org** 上的 `官方教程`_ .
