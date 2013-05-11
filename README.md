# websocket-driver [![Build Status](https://travis-ci.org/faye/websocket-driver-ruby.png)](https://travis-ci.org/faye/websocket-driver-ruby)

This module provides a complete implementation of the WebSocket protocols that
can be hooked up to any TCP library. It aims to simplify things by decoupling
the protocol details from the I/O layer, such that users only need to implement
code to stream data in and out of it without needing to know anything about how
the protocol actually works. Think of it as a complete WebSocket system with
pluggable I/O.

Due to this design, you get a lot of things for free. In particular, if you
hook this module up to some I/O object, it will do all of this for you:

* Select the correct server-side driver to talk to the client
* Generate and send both server- and client-side handshakes
* Recognize when the handshake phase completes and the WS protocol begins
* Negotiate subprotocol selection based on `Sec-WebSocket-Protocol`
* Buffer sent messages until the handshake process is finished
* Deal with proxies that defer delivery of the draft-76 handshake body
* Notify you when the socket is open and closed and when messages arrive
* Recombine fragmented messages
* Dispatch text, binary, ping and close frames
* Manage the socket-closing handshake process
* Automatically reply to ping frames with a matching pong
* Apply masking to messages sent by the client

This library was originally extracted from the [Faye](http://faye.jcoglan.com)
project but now aims to provide simple WebSocket support for any Ruby server or
I/O system.


## Installation

```
$ gem install websocket-driver
```


## Usage

To build either a server-side or client-side socket, the only requirement is
that you supply a `socket` object with these methods:

* `socket.url` - returns the full URL of the socket as a string.
* `socket.write(string)` - writes the given string to a TCP stream.

Server-side sockets require one additional method:

* `socket.env` - returns a Rack-style env hash that will contain some of the
  following fields. Their values are strings containing the value of the named
  header, unless stated otherwise.
  * `HTTP_CONNECTION`
  * `HTTP_HOST`
  * `HTTP_ORIGIN`
  * `HTTP_SEC_WEBSOCKET_KEY`
  * `HTTP_SEC_WEBSOCKET_KEY1`
  * `HTTP_SEC_WEBSOCKET_KEY2`
  * `HTTP_SEC_WEBSOCKET_PROTOCOL`
  * `HTTP_SEC_WEBSOCKET_VERSION`
  * `HTTP_UPGRADE`
  * `rack.input`, an `IO` object representing the request body
  * `REQUEST_METHOD`, the request's HTTP verb


### Server-side

To handle a server-side WebSocket connection, you need to check whether the
request is a WebSocket handshake, and if so create a protocol driver for it.
You must give the driver an object with the `env`, `url` and `write` methods.
A simple example might be:

```ruby
require 'websocket/driver'
require 'eventmachine'

class WS
  attr_reader :env, :url

  def initialize(env)
    @env = env

    secure = Rack::Request.new(env).ssl?
    scheme = secure ? 'wss:' : 'ws:'
    @url = scheme + '//' + env['HTTP_HOST'] + env['REQUEST_URI']

    @driver = WebSocket::Driver.rack(self)

    env['rack.hijack'].call
    @io = env['rack.hijack_io']

    EM.attach(@io, Reader) { |conn| conn.driver = @driver }

    @driver.start
  end

  def write(string)
    @io.write(string)
  end

  module Reader
    attr_writer :driver

    def receive_data(string)
      @driver.parse(string)
    end
  end
end
```

To explain what's going on here: the `WS` class implements the `env`, `url` and
`write(string)` methods as required. When instantiated with a Rack environment,
it stores the environment and infers the complete URL from it.  Having set up
the `env` and `url`, it asks `WebSocket::Driver` for a server-side driver for
the socket. Then it uses the Rack hijack API to gain access to the TCP stream,
and uses EventMachine to stream in incoming data from the client, handing
incoming data off to the driver for parsing. Finally, we tell the driver to
`start`, which will begin sending the handshake response.  This will invoke the
`WS#write` method, which will send the response out over the TCP socket.

Having defined this class we could use it like this when handling a request:

```ruby
if WebSocket::Driver.websocket?(env)
  socket = WS.new(env)
end
```

The driver API is described in full below.


### Client-side

Similarly, to implement a WebSocket client you need an object with `url` and
`write` methods. Once you have one such object, you ask for a driver for it:

```ruby
driver = WebSocket::Driver.client(socket)
```

After this you use the driver API as described below to process incoming data
and send outgoing data.


### Driver API

Drivers are created using one of the following methods:

```ruby
driver = WebSocket::Driver.rack(socket, options)
driver = WebSocket::Driver.client(socket, options)
```

The `rack` method returns a driver chosen using the socket's `env`. The
`client` method always returns a driver for the RFC version of the protocol
with masking enabled on outgoing frames.

The `options` argument is optional, and is a hash. It may contain the following
keys:

* `:protocols` - an array of strings representing acceptable subprotocols for
  use over the socket. The driver will negotiate one of these to use via the
  `Sec-WebSocket-Protocol` header if supported by the other peer.

All drivers respond to the following API methods, but some of them are no-ops
depending on whether the client supports the behaviour.

Note that most of these methods are commands: if they produce data that should
be sent over the socket, they will give this to you by calling
`socket.write(string)`.

#### `driver.on('open') { |event| }`

Sets the callback block to execute when the socket becomes open.

#### `driver.on('message') { |event| }`

Sets the callback block to execute when a message is received. `event` will
have a `data` attribute containing either a string in the case of a text
message or an array of integers in the case of a binary message.

#### `driver.on('error') { |event| }`

Sets the callback to execute when a protocol error occurs due to the other peer
sending an invalid byte sequence. `event` will have a `message` attribute
describing the error.

#### `driver.on('close') { |event| }`

Sets the callback block to execute when the socket becomes closed. The `event`
object has `code` and `reason` attributes.

#### `driver.start`

Initiates the protocol by sending the handshake - either the response for a
server-side driver or the request for a client-side one. This should be the
first method you invoke.  Returns `true` iff a handshake was sent.

#### `driver.parse(string)`

Takes a string and parses it, potentially resulting in message events being
emitted (see `on('message')` above) or in data being sent to `socket.write`.
You should send all data you receive via I/O to this method.

#### `driver.text(string)`

Sends a text message over the socket. If the socket handshake is not yet
complete, the message will be queued until it is. Returns `true` if the message
was sent or queued, and `false` if the socket can no longer send messages.

#### `driver.binary(array)`

Takes an array of byte-sized integers and sends them as a binary message. Will
queue and return `true` or `false` the same way as the `text` method. It will
also return `false` if the driver does not support binary messages.

#### `driver.ping(string = '', &callback)`

Sends a ping frame over the socket, queueing it if necessary. `string` and the
`callback` block are both optional. If a callback is given, it will be invoked
when the socket receives a pong frame whose content matches `string`. Returns
`false` if frames can no longer be sent, or if the driver does not support
ping/pong.

#### `driver.close`

Initiates the closing handshake if the socket is still open. For drivers with
no closing handshake, this will result in the immediate execution of the
`on('close')` callback. For drivers with a closing handshake, this sends a
closing frame and `emit('close')` will execute when a response is received or a
protocol error occurs.

#### `driver.version`

Returns the WebSocket version in use as a string. Will either be `hixie-75`,
`hixie-76` or `hybi-$version`.

#### `driver.protocol`

Returns a string containing the selected subprotocol, if any was agreed upon
using the `Sec-WebSocket-Protocol` mechanism. This value becomes available
after `emit('open')` has fired.


## License

(The MIT License)

Copyright (c) 2010-2013 James Coglan

Permission is hereby granted, free of charge, to any person obtaining a copy of
this software and associated documentation files (the 'Software'), to deal in
the Software without restriction, including without limitation the rights to
use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies
of the Software, and to permit persons to whom the Software is furnished to do
so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED 'AS IS', WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
