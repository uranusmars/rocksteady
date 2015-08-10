## Testing ##

For simple metric sending/receiving test, following instruction can be useful.

Rocksteady comes with a pair of publisher and consumer python scripts to test AMQP service.  Inside [\*scripts/\*](http://rocksteady.googlecode.com/svn/trunk/scripts/) directory, modify setting.py if you use a different credential or exchange for RabbitMQ, otherwise it will work with default installation of RabbitMQ

These scripts needs amqplib  python module, debian can installed via

`apt-get install python-amqplib`

Start publishing message by running
```
python amqp_publisher.py
```

Take a look of amqp\_publisher.py, you can easily change the metric name.  The default metric will work with example in other Wiki.

Then run consumer to see the messages are going through MQ correctly.
```
python amqp_consumer.py
```

if your instance of Rocksteady with default config is running, it should be making minutely aggregation once a minute.