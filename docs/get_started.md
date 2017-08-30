## Python server

```shell
rxtender \
--framing rxt_backend_base.python3.framing.newline \
--serialization rxt_backend_base.python3.serialization.json \
--stream rxt_backend_base.python3.stream \
--input counter.rxt > counter_rxt.py
```

```python
import asyncio

class CounterServerProtocol(asyncio.Protocol):
    def connection_made(self, transport):
        peername = transport.get_extra_info('peername')
        print('Connection from {}'.format(peername))
        self.transport = transport

    def connection_lost(self, exc):
        return

    def data_received(self, data):
        message = data.decode()
        print('Data received: {!r}'.format(message))

loop = asyncio.get_event_loop()
# Each client connection will create a new protocol instance
coro = loop.create_server(CounterServerProtocol, '127.0.0.1', 9999)
server = loop.run_until_complete(coro)

# Serve requests until Ctrl+C is pressed
print('Serving on {}'.format(server.sockets[0].getsockname()))
try:
    loop.run_forever()
except KeyboardInterrupt:
    pass

# Close the server
server.close()
loop.run_until_complete(server.wait_closed())
loop.close()
```

create the message router:

```python
def connection_made(self, transport):
    [...]
    self.router = Router(transport
```

add framing support:

```python
class FramedTransport(object):
    def __init__(self, tramsport):
        self.transport = transport

    def write(self, data):
        self.transport.write(frame(data))
```


## NodeJS client

```javascript
var net = require('net');

var HOST = '127.0.0.1';
var PORT = 9999;

var client = new net.Socket();
client.connect(PORT, HOST, function() {
    console.log('CONNECTED TO: ' + HOST + ':' + PORT);
});

client.on('data', function(data) {
    console.log('DATA: ' + data);
});

client.on('close', function() {
    console.log('Connection closed');
});
```
