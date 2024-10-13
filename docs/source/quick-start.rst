快速开始
+++++++++++

一些有用的示例

Simple 消费者
~~~~~~~~~~~~~~~

.. literalinclude:: examples/simple_consumer.py
   :language: python

Simple 发布者
~~~~~~~~~~~~~~~~

.. literalinclude:: examples/simple_publisher.py
   :language: python

异步消息处理
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. literalinclude:: examples/simple_async_consumer.py
   :language: python


使用 RabbitMQ 事务
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. literalinclude:: examples/simple_publisher_transactions.py
   :language: python

获取单个消息示例
~~~~~~~~~~~~~~~~~~~~~~~~~~

.. literalinclude:: examples/main.py
   :language: python

设置 logging 级别
~~~~~~~~~~~~~~~~~

有时你只想查看调试日志，但当你直接调用 `logging.basicConfig(logging.DEBUG)` 时，会为所有记录器设置调试日志级别，包括所有 aio_pika 的模块。如果你想独立设置日志级别，可以参考以下示例：

.. literalinclude:: examples/log-level-set.py
   :language: python

Tornado 示例
~~~~~~~~~~~~~~~

.. literalinclude:: examples/tornado-pubsub.py
   :language: python

外部凭证示例
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. literalinclude:: examples/external-credentials.py
   :language: python

连接池
~~~~~~~~~~~~~~~~~~

.. literalinclude:: examples/pooling.py
   :language: python
