.. _issue: https://github.com/mosquito/aio-pika/issues
.. _pull request: https://github.com/mosquito/aio-pika/compare
.. _aio-pika: https://github.com/mosquito/aio-pika
.. _syslog: http://en.wikipedia.org/wiki/Syslog
.. _官方教程: https://www.rabbitmq.com/tutorials/tutorial-five-python.html
.. _topics:

Topics
======

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


在 :ref:`上一教程 <routing>` 中，我们改进了我们的日志系统。我们不再使用仅能进行简单广播的 fanout 交换，而是使用了 direct 交换，从而获得了选择性接收日志的能力。

尽管使用 direct 交换改善了我们的系统，但它仍然存在一些限制——它无法基于多个标准进行路由。

在我们的日志系统中，我们可能希望不仅根据严重性订阅日志，还根据发出日志的来源进行订阅。您可能从 unix 工具 syslog_ 中了解到这个概念，它根据严重性（`info` / `warn` / `crit`...）和设施（`auth` / `cron` / `kern`...）路由日志。

这将为我们提供很大的灵活性——我们可能只想监听来自 'cron' 的关键错误，同时也希望接收来自 'kern' 的所有日志。

要在我们的日志系统中实现这一点，我们需要学习更复杂的主题交换（topic exchange）。

主题交换（Topic Exchange）
++++++++++++++++++++++++++

发送到主题交换的消息不能使用任意的 *routing_key* —— 它必须是由点分隔的单词列表。这些单词可以是任何内容，但通常会指定与消息相关的一些特征。以下是一些有效的 routing key 示例: `"stock.usd.nyse"`, `"nyse.vmw"`, `"quick.orange.rabbit"` 。routing key 中的单词可以有任意数量，最多限制为 255 字节。

绑定键（binding key）也必须采用相同的格式。主题交换背后的逻辑类似于直接交换——使用特定 routing key 发送的消息将被发送到所有与匹配绑定键相绑定的队列。然而，绑定键有两个重要的特殊情况：

* `*` （星号）可以替代一个单词。
* `#` （井号）可以替代零个或多个单词。

用一个示例来解释这一点是最简单的：

.. image:: /_static/tutorial/python-five.svg
   :align: center

在这个例子中，我们将发送描述动物的消息。这些消息将使用由三个单词（两个点）组成的 routing key 发送。routing key 中的第一个单词将描述速度，第二个单词描述颜色，第三个单词描述物种: `"<celerity>.<colour>.<species>"` 。

我们创建了三个绑定: *Q1* 使用绑定键 `"*.orange.*"` 绑定, *Q2* 使用绑定键 `"*.*.rabbit"` 和 `"lazy.#"` 绑定。

这些绑定可以总结如下：

* Q1 对所有橙色动物感兴趣。
* Q2 想了解有关兔子的所有信息，以及关于懒惰动物的所有信息。
* routing key 设置为 `"quick.orange.rabbit"` 的消息将被发送到两个队列。
  消息 `"lazy.orange.elephant"` 也会发送到两个队列。另一方面，`"quick.orange.fox"` 仅会发送到第一个队列，而 `"lazy.brown.fox"` 仅会发送到第二个队列。`"lazy.pink.rabbit"` 将仅发送到第二个队列一次，即使它匹配了两个绑定。`"quick.brown.fox"` 不匹配任何绑定，因此会被丢弃。

如果我们违反合同，发送一个包含一个或四个单词的消息，例如 `"orange"` 或 `"quick.orange.male.rabbit"`，会发生什么？这些消息将不匹配任何绑定，因此会丢失。

另一方面, `"lazy.orange.male.rabbit"` 尽管有四个单词，但会匹配最后一个绑定，并将被发送到第二个队列。

.. note::

    **主题交换**

    主题交换非常强大，可以像其他交换一样运行。

    当一个队列使用 `"#"` （井号）绑定键绑定时——它将接收所有消息，而不管 routing key 是什么——就像在 fanout 交换中一样。

    当在绑定中不使用特殊字符 `"*"` （星号）和 `"#"` （井号）时，主题交换将表现得就像直接交换一样。


综合起来
+++++++++++++++++

我们将在日志系统中使用主题交换。我们将以日志的 routing keys 有两个单词: `"<facility>.<severity>"` 为工作假设开始。

代码与 :ref:`前一个教程 <routing>` 中的几乎相同。

用于 :download:`emit_log_topic.py <examples/5-topics/emit_log_topic.py>` 的代码：

.. literalinclude:: examples/5-topics/emit_log_topic.py
   :language: python

用于 :download:`receive_logs_topic.py <examples/5-topics/receive_logs_topic.py>` 的代码：

.. literalinclude:: examples/5-topics/receive_logs_topic.py
   :language: python

要接收所有日志，请运行::

    python receive_logs_topic.py "#"

要接收来自设施 `"kern"` 的所有日志，请运行::

    python receive_logs_topic.py "kern.*"

或者如果您只想了解 `"critical"` 日志，请运行::

    python receive_logs_topic.py "*.critical"

您可以创建多个绑定::

    python receive_logs_topic.py "kern.*" "*.critical"

要发出 routing key 为 `"kern.critical"` 的日志，请输入::

    python emit_log_topic.py "kern.critical" "A critical kernel error"

玩得开心！请注意，代码对 routing 或 binding keys 并没有任何假设，您可能想尝试更多的 routing key 参数。

接下来请参阅 :ref:`教程 6 <rpc>` 来了解 RPC。

.. note::

    本材料摘自 **rabbitmq.org** 上的 `官方教程`_ 。
