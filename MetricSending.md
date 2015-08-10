## Introduction ##

Sending metric needs to be easy.  That's the premise we set out when we were designing the solution.  After ops install the box, anyone should be able to start sending metric by looking an example.  We made the choice of creating a port listening daemon to accept metric so other app/script can be MQ unaware and still send metric.  This way, all they need is to connect to a port and send those [metrics](MetricFormat.md).


## Details ##

We send our metrics in two ways:
  1. Send via local daemon, MQ independent.
  1. Send using RabbitMQ library to send directly to message exchange.

### Send to regular tcp port ###

Create a daemon to listen for metric string and forward it to configured RabbitMQ server and exchange.  Admob has released an implementation written in Python.  You can check it out from launchpad

bzr = bazaar source control
```
bzr branch lp:~zirpu/graphite/admob.branch
cd admob.branch/graphite_local_proxy
make pkg
dpkg -i ../graphite-local-proxy_1.0.0-0_all.deb
```

Once package is installed, you'll need to edit /etc/default/graphite\_local\_proxy for rabbitmq server, vhost, exchange, and etc.

Once you have daemon up and running(default listening port is 2003), you can send metric with simple netcat

In Shell
```
echo '1min.system.cpu.prct_idle 80 1283207115' | nc localhost 2003
```

In Python
```
#!/usr/bin/python           # This is server.py file

import socket               # Import socket module

s = socket.socket()         # Create a socket object
host = 'localhost' # Get local machine name
port = 2003                # Reserve a port for your service.

s.connect((host, port))
s.send('1min.system.cpu.prct_idle 80 1283207115')
s.close                     # Close the socket when done
```

In Java
```
package com.test.simplesock.client;

import java.io.*;
import java.net.*;
/**
 * Very simple socket server client example written as static methods for simplicity
 * in adding to an existing project.
 */
public class Client {

    public static String sendString(String request) {
    	return sendString("localhost", 2003, request);
    }
    public static String sendString(String server, int port, String request) {
    	return (String) send(server, port, (Object) request);
    }
    public static Object send(String server, int port, Object request) {
    	Object response;

        try {
            Socket socket = new Socket(server, port);
            ObjectOutputStream out = new ObjectOutputStream(socket.getOutputStream());
            ObjectInputStream in = new ObjectInputStream(socket.getInputStream());

            out.writeObject(request);
            out.flush();

            response = in.readObject();
            
            out.close();
            in.close();
            socket.close();
        } 
        catch (Exception e) {
        	throw new RuntimeException(e);
        }        
        return response;
    }
}
```
### Send using RabbitMQ client ###

Many language has a client library, check this [Rabbitmq page](http://www.rabbitmq.com/how.html) for detail info.