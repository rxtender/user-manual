This is the "not so short" getting started tutorial of RxTender. In this
tutorial you will learn to use reactive streams to communicate between python
and javascript code.

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

The first thing to do is to write the stream specifications. We want 3 arguments
to create our counter: Its initial value, its end value, and its increase step.
The stream specification is written in an IDL whose syntax is heavily inspired
from rust. Save the following code to a file named "counter.rxt":

```
struct CounterItem {
    value: i32;
}

struct CounterError {
    message :string;
}

stream Counter(start: i32, end: i32, step: i32) -> Stream<CounterItem, CounterError>;
```

First there is the definition of the type of items that are emitted on the
stream. They are named "CounterItem" and contain a single field named "value".
The type of this field is a signed 32 bits integer (i32).

Then there is the definition of the error type associated to the stream. The
COunterError struct contains a single string field named "message".

Finally there is the definition of the stream. It is named "Counter". It takes 3
creation arguments: start, end, and step. This stream will emit items of type
CounterItem, and raise errors of type CounterError.

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

For the nodejs client, we start with a TCP client, implemented as a function
that returns a stream of stream:

```javascript
import {Observable, Subject} from 'rxjs';
import {Socket} from 'net';
let connection = [];

function createConnection(id, host, port) {
  const state$ = Observable.create(stateObserver => {
    let dataObserver = null;
    const data$ = Observable.create(observer => {
      dataObserver = observer;
    });

    connection[id] = new Socket();
    connection[id].setEncoding('utf8');
    connection[id].connect(port, host, function() {
        console.log('CONNECTED TO: ' + host + ':' + port);
        stateObserver.next({
          "linkId": id,
          "stream": data$
        })
    });

    connection[id].on('data', function(data) {
      dataObserver.next(data);
    });

    connection[id].on('close', function() {
        dataObserver.complete();
        stateObserver.complete();
        connection.splice(id, 1);
    });
  });

  return state$;
}
```

This function returns a stream that emits a stream when the connection is
established. This latter stream then emits items each time some data is received
on the socket. This function is used from a factory function that takes another
stream as input:

```javascript
function tcpClient(sink$) {
  sink$.subscribe( (i) => {
    connection[i.linkId].write(i.data);
  });

  return {
    "connect" : createConnection
  };
}
```

Each item of the sink stream contains the data to write on the tcp socket.
We will also use log function:

```javascript
function consoleDriver(sink$) {
  sink$.subscribe( (i) => {
    console.log('console: ' + i);
  });
}
```

Now we can implement our logic:

```javascript
import { router } from './counter_rxt.js';

function main(sources) {
  const linkRcv$ = sources.LINK.connect('counter', 'localhost', 9999);
  const returnChannel$ = sources.ROUTER.linkData();
  const console$ = sources.ROUTER.link()
    .map( i => {
      return sources.ROUTER.Counter(i.linkId, 1,10,1)
    })
    .mergeAll()
    .map( i => i.value);;

  return {
   ROUTER: linkRcv$,
   LINK: returnChannel$,
   CONSOLE: console$
  };
}
```

The sources parameter contains objects returned by the tcp client factory, the
console factory, and the router factory. This main function creates a Counter
observable when the tcp connection is established, i.e. when an item is emitted
on the sources.ROUTER.link() stream. The result of the counter is returned in
the CONSOLE field, and will be connected to the console output.

All streams are finally connected together:

```javascript
const consoleProxy$ = new Subject();
const routerProxy$ = new Subject();
const linkProxy$ = new Subject();

const sources = {
  CONSOLE: consoleDriver(consoleProxy$),
  ROUTER: router(routerProxy$),
  LINK: tcpClient(linkProxy$)
};

const sinks = main(sources);

sinks.ROUTER.subscribe(routerProxy$);
sinks.LINK.subscribe(linkProxy$);
sinks.CONSOLE.subscribe(consoleProxy$);
```

Here is the complete code of the client. Save it to a file named
"client.es6.js":

```javascript
import {Observable, Subject} from 'rxjs';
import {
  router,
} from './counter_rxt.js';
import {Socket} from 'net';

let connection = [];

function createConnection(id, host, port) {
  const state$ = Observable.create(stateObserver => {
    let dataObserver = null;
    const data$ = Observable.create(observer => {
      dataObserver = observer;
    });

    connection[id] = new Socket();
    connection[id].setEncoding('utf8');
    connection[id].connect(port, host, function() {
        console.log('CONNECTED TO: ' + host + ':' + port);
        stateObserver.next({
          "linkId": id,
          "stream": data$
        })
    });

    connection[id].on('data', function(data) {
      dataObserver.next(data);
    });

    connection[id].on('close', function() {
        dataObserver.complete();
        stateObserver.complete();
        connection.splice(id, 1);
    });
  });

  return state$;
}

function tcpClient(sink$) {
  sink$.subscribe( (i) => {
    connection[i.linkId].write(i.data);
  });

  return {
    "connect" : createConnection
  };
}

function consoleDriver(sink$) {
  sink$.subscribe( (i) => {
    console.log('console: ' + i);
  });
}

function main(sources) {
  const linkRcv$ = sources.LINK.connect('counter', 'localhost', 9999);
  const returnChannel$ = sources.ROUTER.linkData();
  const console$ = sources.ROUTER.link()
    .map( i => {
      return sources.ROUTER.Counter(i.linkId, 1,10,1)
    })
    .mergeAll()
    .map( i => i.value);;

  return {
   ROUTER: linkRcv$,
   LINK: returnChannel$,
   CONSOLE: console$
  };
}

const consoleProxy$ = new Subject();
const routerProxy$ = new Subject();
const linkProxy$ = new Subject();

const sources = {
  CONSOLE: consoleDriver(consoleProxy$),
  ROUTER: router(routerProxy$),
  LINK: tcpClient(linkProxy$)
};

const sinks = main(sources);

sinks.ROUTER.subscribe(routerProxy$);
sinks.LINK.subscribe(linkProxy$);
sinks.CONSOLE.subscribe(consoleProxy$);
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
--input counter.rxt --output counter_rxt.py
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
    "generate:counter": "rxtender --framing rxt_backend_base.es2015.framing.newline --serialization rxt_backend_base.es2015.serialization.json --stream rxt_backend_base.es2015_rxjs.stream --input counter.rxt --output counter_rxt.es6.js",
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

The stream argument provided to rxtender generates an es2015 rxjs router
function. Since we use a TCP connection, and TCP is a stream protocol, we need
some framing to encapsulate each item of the stream. Here we use new lines
framing, as specified with the framing argument. With the serialization argument
we asked rxtender to generate json serialization of all structs defined in the
IDL. To generate the code, install npm dependencies and run rxtender:

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
