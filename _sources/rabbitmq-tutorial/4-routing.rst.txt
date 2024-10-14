.. _issue: https://github.com/mosquito/aio-pika/issues
.. _pull request: https://github.com/mosquito/aio-pika/compare
.. _aio-pika: https://github.com/mosquito/aio-pika
.. _官方教程: https://www.rabbitmq.com/tutorials/tutorial-four-python.html
.. _routing:

路由
=======

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


在 :ref:`上一教程 <publish-subscribe>` 中，我们构建了一个简单的日志系统，能够将日志消息广播给多个接收者。

在本教程中，我们将为其添加一个功能——我们将使订阅只接收部分消息成为可能。例如，我们可以将只有关键错误消息定向到日志文件（以节省磁盘空间），同时仍然能够在控制台上打印所有日志消息。


绑定
++++++++

在之前的示例中，我们已经创建了绑定。你可能还记得类似的代码：

.. code-block:: python

    async def main():
        ...

        # 将队列绑定到交换机
        await queue.bind(logs_exchange)

    ...


绑定是交换机和队列之间的关系。可以简单理解为：队列对来自该交换机的消息感兴趣。

绑定可以接受一个额外的 *routing_key* 参数。为了避免与 *basic_publish* 参数混淆，我们将其称为 *绑定键*。以下是如何使用键创建绑定的示例：

.. code-block:: python

    async def main():
        ...

        # 将队列绑定到交换机
        await queue.bind(logs_exchange,
                         routing_key="black")

    ...


*绑定键* 的含义取决于交换机的类型。我们之前使用的 *扇出型* 交换机会简单地忽略这个值。

直连交换机  
+++++++++++++++

我们在前一教程中的日志系统将所有消息广播给所有消费者。现在我们想扩展功能，允许根据消息的严重性进行过滤。例如，我们可能希望将日志消息写入磁盘的脚本只接收关键错误，而不浪费磁盘空间存储警告或信息日志消息。

我们之前使用的是扇出型交换机，它的灵活性有限——只能进行无脑的广播。

我们将改用直连交换机。直连交换机背后的路由算法很简单——消息会被路由到绑定键与消息的路由键完全匹配的队列。

举个例子，看看以下设置：

.. image:: /_static/tutorial/direct-exchange.svg  
   :align: center

在这个设置中，我们看到直连交换机 X 与两个队列绑定。第一个队列的绑定键是 *orange*，第二个队列有两个绑定，一个绑定键为 *black*，另一个为 *green*。

在这样的设置中，带有路由键 *orange* 的消息将被路由到队列 *Q1*。带有路由键 *black* 或 *green* 的消息将被路由到 *Q2*。所有其他消息将被丢弃。


多重绑定  
+++++++++++++++++

.. image:: /_static/tutorial/direct-exchange-multiple.svg  
   :align: center

将多个队列与相同的绑定键绑定是完全合法的。在我们的示例中，我们可以添加一个绑定，将交换机 *X* 与队列 *Q1* 通过绑定键 *black* 绑定。在这种情况下，直连交换机将像扇出交换机一样工作，并将消息广播到所有匹配的队列。带有路由键 *black* 的消息将会同时被发送到 *Q1* 和 *Q2*。


发送日志  
+++++++++++++

我们将使用这种模型来构建日志系统。与使用 *fanout* 不同，我们会将消息发送到 *direct* 交换机。我们将日志的严重性作为 *routing key* 提供，这样接收脚本就可以选择它想要接收的严重性。首先让我们专注于发送日志。

像往常一样，我们首先需要创建一个交换机：

.. code-block:: python

    from aio_pika import ExchangeType

    async def main():
        ...

        direct_logs_exchange = await channel.declare_exchange(
            'logs', ExchangeType.DIRECT
        )

现在我们可以发送消息了：

.. code-block:: python

    async def main():
        ...

        await direct_logs_exchange.publish(
            Message(message_body),
            routing_key=severity,
        )

为简化起见，我们假设 `'severity'` 可以是 `'info'`、`'warning'` 或 `'error'` 之一。

订阅  
+++++++++++

接收消息的方式与之前的教程相同，唯一的例外是——我们将为每个感兴趣的严重性创建一个新的绑定。

.. code-block:: python

    async def main():
        ...

        # 声明队列
        queue = await channel.declare_queue(exclusive=True)

        # 将队列绑定到交换机
        await queue.bind(direct_logs_exchange,
                         routing_key=severity)

    ...


综合起来  
+++++++++++++++++++++++

.. image:: /_static/tutorial/python-four.svg  
   :align: center  

简化后的代码 :download:`receive_logs_direct_simple.py <examples/4-routing/receive_logs_direct_simple.py>`:

.. literalinclude:: examples/4-routing/receive_logs_direct_simple.py  
   :language: python  

代码 :download:`emit_log_direct.py <examples/4-routing/emit_log_direct.py>`:

.. literalinclude:: examples/4-routing/emit_log_direct.py  
   :language: python  

.. note::

   基于回调的代码 :download:`receive_logs_direct.py <examples/4-routing/receive_logs_direct.py>`:

   .. literalinclude:: examples/4-routing/receive_logs_direct.py  
      :language: python  

如果您只想将 *'warning'* 和 *'error'*（而不是 *'info'*）日志消息保存到文件中，只需打开控制台并输入::

    $ python receive_logs_direct_simple.py warning error > logs_from_rabbit.log  

如果您希望在屏幕上查看所有日志消息，请打开一个新终端并执行::

    $ python receive_logs_direct.py info warning error
     [*] Waiting for logs. To exit press CTRL+C 

例如，要发送错误日志消息，只需输入::

    $ python emit_log_direct.py error "Run. Run. Or it will explode."  
    [x] Sent 'error':'Run. Run. Or it will explode.'

继续阅读 :ref:`教程 5 <topics>`，了解如何根据模式监听消息。

.. note::

    该材料改编自 **rabbitmq.org** 上的 `官方教程`_ 。