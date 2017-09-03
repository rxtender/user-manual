Let's consider the following situation: Your team develops a product that is
composed of two services, using different programming languages (javascript and
python). You use Reactive Streams in both components and you need to communicate
between them. This is exactly where RxTender can help you: Generate reactive
streams bindings between your services.

Your user story is the following one:

As an end user using a terminal console I use a command line application to
count from 1 to 10. Since such a computation is very complex, this CLI
application asks another service to do it.

You ended up with the following technical choices:

- The client application is developed in javascript/NodeJS.
- The server is developed in python.
- TCP is used as a transport protocol.
- JSON is used as a serialization protocol.
- JSON lines us used as a framing protocol.

## Stream Definition

The first thing to do is to write the stream specification. We want 3 arguments
to create our counter: Its initial value, its end vale, and its increase step.
The stream specification is written in an IDL whose syntax is heavily inspired
from rust. Save the following code to a file named "counter.rxt":

```
item CounterItem {
    value: i32;
}

stream Counter(start: i32, end: i32, step: i32) -> CounterItem;
```

First there is the definition of the type of items that are emitted on the
stream. They are named "CounterItem" and contain a single field name "value".
The type of this field is a signed 32bits integer (i32).

Then there is the definition of the stream. It is named "Counter". It takes 3
arguments as input: start, end, and step. This stream will emit items of type
CounterItem.

## The python server

Now let's code the python server, based on asyncio. We first start from a sample
asyncio TCP server:

```python
import asyncio

class CounterServerProtocol(asyncio.Protocol):
    def connection_made(self, transport):
        peername = transport.get_extra_info('peername')
        print('Connection from {}'.format(peername))

    def connection_lost(self, exc):
        print('connection lost')
        return

    def data_received(self, data):
        return

loop = asyncio.get_event_loop()
# Each client connection will create a new protocol instance
coro = loop.create_server(CounterServerProtocol, '127.0.0.1', 9999)
server = loop.run_until_complete(coro)

print('Serving on {}'.format(server.sockets[0].getsockname()))
try:
    loop.run_forever()
except KeyboardInterrupt:
    pass

server.close()
loop.run_until_complete(server.wait_closed())
loop.close()
```

Each time a client connects to the server, a new CounterServerProtocol object is
created and its "connection_made" method is called. The first thing to do is
importing the definitions we will need:

```diff
import asyncio

+from rx import Observable
+from counter_rxt import frame, unframe, Router, CounterItem
```

Note that we use RxPy here, but we could use other reactive stream libraries.
The we must provide a factory function to create the counter stream when a
request is received from the network:

```diff
+Router.set_Counter_factory(create_counter_stream)
loop = asyncio.get_event_loop()
# Each client connection will create a new protocol instance
```

"Router.set_Counter_factory" is a static method of the Router class. RxTender
generates one of these methods per stream declared in the rxt file. We will
write the create_counter_stream function later. We will now create a Router
object each time a new connection is made:

```diff
def connection_made(self, transport):
    peername = transport.get_extra_info('peername')
    print('Connection from {}'.format(peername))
+    self.router = Router(FramedTransport(transport))
+    self.frame_context = ''
```

The router object contains the implementation that routes network messages
to/from streams. It is also in charge of (de)serializing these messages. Its
constructor takes a transport object as an argument. This transport object must
implement a "write" method to write data on the network connection. Note that we
did not use the "transport" argument directly: Since we use TCP (a stream based
protocol), we need add framing on top of it so that we receive messages
correctly. This is what is done by the FramedTransport class:

```python
class FramedTransport(object):
    def __init__(self, transport):
        self.transport = transport

    def write(self, data):
        self.transport.write(frame(data).encode())
```

In the write method, each message is "framed" with the JSON-lines protocol
before being sent. On the other way, all received packets are "unframed" before
they are provided to the Router:

```diff
def data_received(self, data):
+    message = data.decode()
+    self.frame_context, packets = unframe(self.frame_context, message)
+    for packet in packets:
+        self.router.on_message(packet)
```

From that point we created a Router, it receives all messages from the network
and can send messages. We finally need to implement create_counter_stream that
will be called each time a new stream subscription request comes:

```python
def delete_counter_subscription(stream):
    stream = None

def create_counter_stream(start, end, step):
    source = Observable.from_(range(start,end+1, step)).map(
        lambda i: CounterItem(i))
    return lambda n,c,e: subscribe_counter_stream(source, n, c, e), lambda: delete_counter_subscription(source)

def subscribe_counter_stream(stream, next, completed, error):
    stream.subscribe(
        lambda i: next(i),
        lambda e: error(e),
        lambda: completed())
```

create_counter_stream takes 3 arguments as input: The ones that we specified in the rxt file. It returns 2 functions:

- A function to subscribe to this stream
- A function to delete this stream

The subscription function is called with 3 function arguments (next, error,
completed). Each time the corresponding event occurs on the source stream, then
these functions must be called to forward the event on the transport layer.

Here is the complete code of the server. Save it to a file named "server.py":

```python
import asyncio

from rx import Observable
from counter_rxt import frame, unframe, Router, CounterItem

class FramedTransport(object):
    def __init__(self, transport):
        self.transport = transport

    def write(self, data):
        self.transport.write(frame(data).encode())

class CounterServerProtocol(asyncio.Protocol):
    def connection_made(self, transport):
        peername = transport.get_extra_info('peername')
        print('Connection from {}'.format(peername))
        self.router = Router(FramedTransport(transport))
        self.frame_context = ''

    def connection_lost(self, exc):
        print('connection lost')
        return

    def data_received(self, data):
        message = data.decode()
        self.frame_context, packets = unframe(self.frame_context, message)
        for packet in packets:
            self.router.on_message(packet)

def delete_counter_subscription(stream):
    stream = None

def create_counter_stream(start, end, step):
    source = Observable.from_(range(start,end+1, step)).map(
        lambda i: CounterItem(i))
    return lambda n,c,e: subscribe_counter_stream(source, n, c, e), lambda: delete_counter_subscription(source)

def subscribe_counter_stream(stream, next, completed, error):
    stream.subscribe(
        lambda i: next(i),
        lambda e: error(e),
        lambda: completed())

Router.set_Counter_factory(create_counter_stream)
loop = asyncio.get_event_loop()
# Each client connection will create a new protocol instance
coro = loop.create_server(CounterServerProtocol, '127.0.0.1', 9999)
server = loop.run_until_complete(coro)

print('Serving on {}'.format(server.sockets[0].getsockname()))
try:
    loop.run_forever()
except KeyboardInterrupt:
    pass

server.close()
loop.run_until_complete(server.wait_closed())
loop.close()
```


## NodeJS client

For the nodejs client, we start with a TCP client app:

```javascript
var net = require('net');

var HOST = '127.0.0.1';
var PORT = 9999;

var client = new net.Socket();
client.connect(PORT, HOST, function() {
    console.log('CONNECTED TO: ' + HOST + ':' + PORT);
});

client.on('data', function(data) {
});

client.on('close', function() {
    console.log('Connection closed');
});
```

Now we can create an observable when the client is connected. We first need to
import the generated functions we will use:

```javascript
import {
  frame, unframe, Router, createCounterObservable
} from './counter_rxt.js';
```

And implement our logic in the connect callback:

```javascript
var router = null;
var context = '';
client.connect(PORT, HOST, function() {
    console.log('CONNECTED TO: ' + HOST + ':' + PORT);
    router = new Router({"write": (d) => {
      client.write(frame(d));
    }});

    console.log('creating observable');
    createCounterObservable(router, 1, 10, 1)
    .subscribe(
      (i) => { console.log('tick: ' + i.value); },
      (e) => { console.log('stream error'); },
      () => {
        console.log('completed');
        process.exit();
      }
    );
});
```

We fist create a router. The router takes a transport object as an argument.  In
its write function we frame the data before sending it on the network link.

Then we create an Observable. The returned observable is an RxJS Observable. So
we can use it as any other stream. Here we directly subscribe to it and print
each received item.

The last thing to do is implementing the data reception callback:

```javascript
client.on('data', function(data) {
    const result = unframe(context, data.toString());
    context = result.context;
    result.packets.forEach( (e) => {
      router.onMessage(e);
    })
});
```

Each received data is unframed, and each unframed packet is notified to the
router.

Here is the complete code of the client. Save it to a file named
"client.es6.js":

```javascript
import {
  frame, unframe, Router,
  createCounterObservable
} from './counter_rxt.js';

var net = require('net');

var HOST = '127.0.0.1';
var PORT = 9999;

var client = new net.Socket();

var router = null;
var context = '';
client.connect(PORT, HOST, function() {
    console.log('CONNECTED TO: ' + HOST + ':' + PORT);
    router = new Router({"write": (d) => {
      client.write(frame(d));
    }});

    console.log('creating observable');
    createCounterObservable(router, 1, 10, 1)
    .subscribe(
      (i) => { console.log('tick: ' + i.value); },
      (e) => { console.log('stream error'); },
      () => {
        console.log('completed');
        process.exit();
      }
    );
});

client.on('data', function(data) {
    const result = unframe(context, data.toString());
    context = result.context;
    result.packets.forEach( (e) => {
      router.onMessage(e);
    })
});

client.on('close', function() {
    console.log('Connection closed');
});
```

## Running it all

Now that all code is there, let's start both components. We will first Build the
python bindings. First install the base backend on your system:

```shell
pip3 install rxt-backend-base
```

Then generate the proxy code corresponding to the Counter IDL:

```shell
rxtender \
--framing rxt_backend_base.python3.framing.newline \
--serialization rxt_backend_base.python3.serialization.json \
--stream rxt_backend_base.python3.stream \
--input counter.rxt > counter_rxt.py
```

rxtender is invoked with 3 generation arguments (framing, serialization, and
stream). Each parameter corresponds to an available code generator in the
backend.

Then we build the javascript bindings. In order to ease generation and
translation from es6 to es5 we use npm with babel:

Javascript NPM package.json:

```json
{
  "name": "counter-example",
  "version": "0.2.0",
  "description": "es2015/python3 rxtender counter example",
  "license": "MIT",
  "main": "client.js",
  "scripts": {
    "generate:counter": "rxtender --framing rxt_backend_base.es2015.framing.newline --serialization rxt_backend_base.es2015.serialization.json --stream rxt_backend_base.es2015.stream --stream rxt_backend_base.es2015_rxjs.stream --input counter.rxt > counter_rxt.es6.js",
    "build:counter": "npm run generate:counter && babel --presets es2015 counter_rxt.es6.js --out-file counter_rxt.js",
    "build:client": "npm run build:counter && babel --presets es2015 client.es6.js --out-file client.js",
    "build": "npm run build:client",
    "start": "node client.js"
  },
  "dependencies": {},
  "devDependencies": {
    "babel-cli": "^6.24.1",
    "babel-core": "^6.25.0",
    "babel-preset-es2015": "^6.24.1",
    "rxjs": "^5.4.3"
  }
}
```

You can see here that 2 stream arguments are provided to rxtender: es2015.stream
and es2015_rxjs.stream. The first one generates generic es2015 stream bindings
while the second one generates constructor functions for RxJS
Observables (The createCounterObservable function that we used to create the
stream). To generate the code, install npm dependencies and run rxtender:

```shell
npm install
npm run build
```


Now everything is ready, we can start the python server:

```shell
python3 server.py
```

and run the javascript client:

```shell
npm start
```

You should get the following output:

```shell
> counter-example@0.2.0 start /Users/bipbip/Documents/devel/rxtender/test
> node client.js

CONNECTED TO: 127.0.0.1:9999
creating observable
tick: 1
tick: 2
tick: 3
tick: 4
tick: 5
tick: 6
tick: 7
tick: 8
tick: 9
tick: 10
completed
```

Yeah! With only few lines of codes you used reactive streams to communicate
between two processes written in different programming languages. You can now
add other streams in the rxt file. All these streams will be multiplexed in
the same socket. Moreover, if you want to change the serialization or framing
protocol, then you do not need to change anything in your code: Just select
other ones when invoking rxtender.
