Why sending metric passively?
  * One host polling thousands of hosts are inefficient and hard problem to solve.
  * It's hard to maintain list of hosts/metrics to pull.
  * With graphite local proxy install on every box, adding a new metric is done by sending a line of text to a localhost port, and graph shows up on the other end right away.

Why use RabbitMQ, why not send metrics to rocksteady directly?
  * One host can not keep up with thousands of metrics per second.
  * As shown in the graph, there are other consumers such as Graphite for the same metrics, so having a message queue that supports publish/subscribe model makes distribution very easy.
  * Easier to scale in the future.  Most messaging system supports key routing, that makes adding extra instances of rocksteady as simple as subscribing with new hash key.