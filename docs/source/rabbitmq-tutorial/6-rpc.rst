.. _issue: https://github.com/mosquito/aio-pika/issues
.. _pull request: https://github.com/mosquito/aio-pika/compare
.. _aio-pika: https://github.com/mosquito/aio-pika
.. _官方教程: https://www.rabbitmq.com/tutorials/tutorial-six-python.html
.. _rpc:

远程过程调用(RPC)
===========================

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


在 :ref:`第二个教程 <work-queues>` 中，我们学习了如何使用 *工作队列* 在多个工作者之间分配耗时的任务。

但是，如果我们需要在远程计算机上运行一个函数并等待结果呢？嗯，那就是另一个故事。这个模式通常被称为 *远程过程调用（Remote procedure call - RPC）*。

在本教程中，我们将使用 RabbitMQ 构建一个 RPC 系统：一个客户端和一个可扩展的 RPC 服务器。由于我们没有任何值得分配的耗时任务，我们将创建一个返回 Fibonacci 数字的虚拟 RPC 服务。


客户端接口
++++++++++++++++

为了说明 RPC 服务如何使用，我们将创建一个简单的客户端类。它将暴露一个名为 `call` 的方法，该方法发送 RPC 请求并阻塞，直到收到答案：

```python
async def main():
    fibonacci_rpc = FibonacciRpcClient()
    result = await fibonacci_rpc.call(4)
    print("fib(4) is %r" % result)
```

.. note::

    **关于 RPC 的说明**

    尽管 RPC 是计算中一个相当常见的模式，但它常常受到批评。问题出现在程序员不知道函数调用是本地的还是一个慢速的 RPC。这样的混淆会导致系统不可预测，并为调试增加不必要的复杂性。滥用 RPC 可能导致不可维护的杂乱代码，而不是简化软件。

    鉴于此，请考虑以下建议：

    * 确保明确哪些函数调用是本地的，哪些是远程的。
    * 记录系统文档，明确组件之间的依赖关系。
    * 处理错误情况。当 RPC 服务器长时间宕机时，客户端应该如何反应？

    当不确定时，避免使用 RPC。如果可以，您应该使用异步管道——而不是 RPC 类似的阻塞，结果会异步推送到下一个计算阶段。

回调队列
++++++++++++++

一般来说，通过 RabbitMQ 实现 RPC 很简单。客户端发送请求消息，服务器回复响应消息。为了接收响应，客户端需要在请求中发送一个“回调(callback)”队列地址。让我们尝试一下：

.. code-block:: python

    async def main():
        ...

        # 用于结果的队列
        callback_queue = await channel.declare_queue(exclusive=True)

        await channel.default_exchange.publish(
            Message(
                request,
                reply_to=callback_queue.name
            ),
            routing_key='rpc_queue'
        )

        # ... 还有一些代码来从 callback_queue 读取响应消息 ...

    ...

.. note::

    **消息属性**

    AMQP 协议预定义了一组与消息相关的 14 个属性。大多数属性很少使用，以下属性是例外：

    * `delivery_mode`：将消息标记为持久（值为 2）或临时（任何其他值）。您可能还记得这个属性来自于 :ref:`第二个教程 <work-queues>`。
    * `content_type`：用于描述编码的 MIME 类型。例如，对于常用的 JSON 编码，设置此属性为 `application/json` 是一个好习惯。
    * `reply_to`：通常用于命名回调队列。
    * `correlation_id`：用于将 RPC 响应与请求关联。

    详细信息见 :class:`aio_pika.Message`。


关联 ID
++++++++++++++

在上面提出的方法中，我们建议为每个 RPC 请求创建一个回调队列。这是相当低效的，但幸运的是，有更好的方法 —— 我们可以为每个客户端创建一个单一的回调队列。

这就引出了一个新问题：在那个队列中收到响应后，无法明确响应属于哪个请求。这时就需要使用 `correlation_id` 属性。我们将为每个请求设置一个唯一值。稍后，当我们在回调队列中接收到消息时，我们会查看这个属性，并根据它能够将响应与请求匹配。如果看到一个未知的 `correlation_id` 值，我们可以安全地丢弃这条消息 —— 它不属于我们的请求。

你可能会问，为什么在回调队列中忽略未知消息，而不是出现错误？这是因为在服务器端可能发生竞争条件。虽然不太可能，但RPC服务器可能会在发送答案后、发送请求确认消息之前崩溃。如果发生这种情况，重启的RPC服务器将再次处理该请求。这就是为什么在客户端我们必须优雅地处理重复的响应，理想情况下，RPC 应该是幂等的。


总结
+++++++

我们的 RPC 工作流程如下：

* 当客户端启动时，它创建一个匿名的独占回调队列。
* 对于 RPC 请求，客户端发送一条消息，包含两个属性： `reply_to` ，设置为回调队列，以及 `correlation_id`，为每个请求设置一个唯一值。
* 请求被发送到 `rpc_queue` 队列。
* RPC 工作者（即服务器）在该队列上等待请求。当请求出现时，它执行任务并将结果消息发送回客户端，使用 `reply_to` 字段中的队列。
* 客户端在回调队列上等待数据。当消息出现时，它检查 `correlation_id` 属性。如果与请求中的值匹配，则将响应返回给应用程序。

综合起来
+++++++++++++++++++++++

:download:`rpc_server.py <examples/6-rpc/rpc_server.py>` 的代码:

.. literalinclude:: examples/6-rpc/rpc_server.py
   :language: python
   :linenos:

服务器代码相当简单明了：

* (34) 和往常一样，我们首先建立连接并声明队列。
* (6) 我们声明了我们的 Fibonacci 函数。它假设只有有效的正整数输入。（不要指望它能处理大数字，这可能是最慢的递归实现）。
* (15) 我们为 `basic_consume` 声明了一个回调，这是 RPC 服务器的核心。 当请求被接收时，它会执行这个回调。它完成工作并将响应发送回去。

:download:`rpc_client.py <examples/6-rpc/rpc_client.py>` 的代码:

.. literalinclude:: examples/6-rpc/rpc_client.py
   :language: python
   :linenos:


客户端代码稍微复杂一些：

* (15) 我们建立连接、通道，并声明一个独占的“回调”队列用于接收回复。
* (22) 我们订阅“回调”队列，以便接收 RPC 响应。
* (26) 每当响应被接收时，`on_response` 回调执行非常简单的任务，
  对于每条响应消息，它检查 `correlation_id` 是否是我们要找的那个。
  如果是，它将响应保存在 `self.response` 中并中断消费循环。
* (30) 接下来，我们定义了我们的主调用方法 - 它执行实际的 RPC 请求。
* (31) 在这个方法中，首先生成一个唯一的 `correlation_id` 编号并保存 - `on_response` 回调函数将使用这个值捕获适当的响应。
* (36) 接下来，我们发布请求消息，带有两个属性：`reply_to` 和 `correlation_id`。最后，我们将响应返回给用户。

我们的 RPC 服务现在准备就绪。我们可以启动服务器::

    $ python rpc_server.py
    [x] Awaiting RPC requests

要请求一个 Fibonacci 数字，请运行客户端::

    $ python rpc_client.py
    [x] Requesting fib(30)

所展示的设计并不是 RPC 服务的唯一实现方式，但它具有一些重要的优点：

如果 RPC 服务器太慢，你可以通过简单地运行另一个服务器来扩展。
尝试在新控制台中运行第二个 `rpc_server.py`。
在客户端方面，RPC 仅需发送和接收一条消息。
不需要像 `queue_declare` 这样的同步调用。因此，RPC 客户端
只需进行一次网络往返即可完成单个 RPC 请求。
我们的代码仍然相当简单，没有尝试解决更复杂（但重要）的问题，例如：

* 如果没有服务器在运行，客户端应该如何反应？
* 客户端是否应该对 RPC 设置某种超时？
* 如果服务器发生故障并引发异常，是否应该将其转发给客户端？
* 在处理之前，如何防止无效的传入消息（例如检查边界）？

.. note::

    如果你想实验一下，可能会发现 rabbitmq-management 插件对于查看队列非常有用。


.. note::

    这些材料是来自 **rabbitmq.org** 上的 `官方教程`_ 。
