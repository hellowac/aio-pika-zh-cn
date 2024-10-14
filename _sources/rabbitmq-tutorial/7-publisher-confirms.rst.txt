.. _issue: https://github.com/mosquito/aio-pika/issues
.. _pull request: https://github.com/mosquito/aio-pika/compare
.. _aio-pika: https://github.com/mosquito/aio-pika
.. _官方教程: https://www.rabbitmq.com/tutorials/tutorial-seven-php.html
.. _publisher-confirms:

发布确认
==================

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


`发布者确认 <https://www.rabbitmq.com/confirms.html#publisher-confirms>`_ 是 RabbitMQ 的一个扩展，用于实现可靠的消息发布。当在通道上启用发布者确认时，客户端发布的消息会被代理异步确认，这意味着它们已经在服务器端得到了处理。

概述
++++++++

在本教程中，我们将使用发布者确认来确保发布的消息安全地到达代理。我们将介绍几种使用发布者确认的策略，并解释它们的优缺点。

在通道上启用发布者确认
++++++++++++++++++++++++++++++++++++++++

发布者确认是对 AMQP 0.9.1 协议的 RabbitMQ 扩展。通过将 :code:`publisher_confirms` 参数设置为 :code:`True` 来在通道级别启用发布者确认，这也是默认设置。

.. code-block:: python

   channel = await connection.channel(
      publisher_confirms=True, # This is the default
   )

策略 #1: 单独发布消息
++++++++++++++++++++++

让我们从最简单的发布确认方法开始，即发布一条消息并同步等待其确认：

.. literalinclude:: examples/7-publisher-confirms/publish_individually.py
   :language: python
   :start-at: # Sending the messages
   :end-before: # Done sending messages

在前面的示例中，我们像往常一样发布一条消息，并使用 :code:`await` 关键字等待其确认。
当消息被确认时，:code:`await` 会立即返回。
如果消息在超时内未被确认，或者被拒绝（即代理由于某种原因无法处理它），:code:`await` 将引发异常。
:code:`aio_pika.connect()` 和 :code:`connection.channel()` 的 :code:`on_return_raises` 参数控制了当强制消息被返回时的行为。
异常的处理通常包括记录错误消息和/或重试发送消息。

不同的客户端库以不同的方式同步处理发布者确认，因此请确保仔细阅读您所使用的客户端的文档。

这种技术非常简单，但也有一个主要缺点：它**显著减慢了发布速度**，因为消息的确认会阻塞所有后续消息的发布。
这种方法无法提供每秒超过几百条发布消息的吞吐量。
尽管如此，这对某些应用来说可能已经足够了。

策略 #2: 批量发布消息
+++++++++++++++++++++

为了改进我们之前的示例，我们可以发布一批消息，并等待整批消息被确认。
以下示例使用了一批 100 条消息：

.. literalinclude:: examples/7-publisher-confirms/publish_batches.py
   :language: python
   :start-at: batchsize = 100
   :end-before: # Done sending messages

等待一批消息被确认相比于等待单条消息的确认显著提高了吞吐量（在远程 RabbitMQ 节点上可提高 20-30 倍）。
一个缺点是，在发生故障时我们无法确切知道出错的原因，因此可能需要将整批消息保留在内存中，以记录一些有意义的信息或重新发布这些消息。
并且这个解决方案仍然是同步的，因此它会阻塞消息的发布。

.. note::

   为了异步启动消息发送，使用 :code:`asyncio.create_task` 创建一个任务，以便由事件循环处理我们的函数执行。
   :code:`await asyncio.sleep(0)` 是必需的，以使事件循环切换到我们的协程。
   任何 :code:`await` 都可以满足这个需求。
   使用 :code:`async for` 和 :code:`async` 生成器时，也需要生成器通过 :code:`await` 进行控制流的转移，以启动消息发送。

   如果没有任务和 :code:`await`，消息发送只会在 :code:`asyncio.gather` 调用时启动。
   对于某些应用来说，这种行为可能是可以接受的。


策略 #3: 异步处理发布确认
+++++++++++++++++++++++++++++++++++++++++++++++++++++++

代理服务器异步确认已发布的消息，我们的辅助函数将发布消息并接收这些确认的通知：

.. literalinclude:: examples/7-publisher-confirms/publish_asynchronously.py
   :language: python
   :start-at: # List for storing tasks
   :end-at: await asyncio.gather(*tasks)

在 Python 3.11 中，可以使用 :code:`TaskGroup` 替代 :code:`list` 和 :code:`asyncio.gather`。

辅助函数发布消息并等待确认。
这样，辅助函数就知道确认、超时或拒绝属于哪条消息。 

.. literalinclude:: examples/7-publisher-confirms/publish_asynchronously.py
   :language: python
   :pyobject: publish_and_handle_confirm


总结
+++++++

确保已发布的消息成功发送到代理服务器在某些应用中可能至关重要。
发布确认是 RabbitMQ 的一项功能，可以帮助满足这一要求。
发布确认本质上是异步的，但也可以以同步的方式处理。
没有一种确定的方法来实现发布确认，这通常取决于应用程序和整体系统的约束。典型的技术包括：

* 单独发布消息，等待同步确认：简单，但吞吐量非常有限。
* 批量发布消息，等待批量的同步确认：简单，吞吐量合理，但当出现问题时很难进行推理。
* 异步处理：最佳性能和资源利用，在出现错误时具有良好的控制，但实现起来可能比较复杂。

.. note::

    本材料改编自 **rabbitmq.org** 上的 `官方教程`_ 。
