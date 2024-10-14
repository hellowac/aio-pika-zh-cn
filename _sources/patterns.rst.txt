.. _aio-pika: https://github.com/mosquito/aio-pika


模式和辅助工具
++++++++++++++++++++

.. note:: 自 `aio-pika>=1.7.0` 起可用

`aio-pika`_ 包含一些用于创建分布式系统的有用模式(patterns)。


.. _patterns-worker:

Master/Worker
~~~~~~~~~~~~~

实现 Master/Worker 模式的辅助工具。这适用于在多个工作者之间平衡任务。

Master 节点创建任务:

.. literalinclude:: examples/master.py
   :language: python


Worker 代码:

.. literalinclude:: examples/worker.py
   :language: python

一个或多个工作者执行任务。


.. _patterns-rpc:

RPC
~~~

实现远程过程调用(Remote Procedure Call - RPC)模式的辅助工具。这适用于在多个工作者之间平衡任务。

调用者创建任务并等待结果：

.. literalinclude:: examples/rpc-caller.py
   :language: python


一个或多个被调用者执行任务：

.. literalinclude:: examples/rpc-callee.py
   :language: python

扩展
~~~~~~~~~

这两种模式的序列化行为可以通过继承和重新定义方法 :func:`aio_pika.patterns.base.serialize` 和 :func:`aio_pika.patterns.base.deserialize` 来改变。

下面的示例演示了这一点：

.. literalinclude:: examples/extend-patterns.py
   :language: python
