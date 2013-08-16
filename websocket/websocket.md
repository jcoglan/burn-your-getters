!SLIDE title
# WebSockets
## Without reading the RFC


!SLIDE diagram

```
GET /eurucamp HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Host: localhost:6001
Origin: https://github.com
Sec-WebSocket-Key: 6V3e74K7aT/sxaUL/d8k0A==
Sec-WebSocket-Version: 13

81 8b 22 4f b2 dd 6a 2a de b1
4d 6f c5 b2 50 23 d6 ...
```


!SLIDE bullets diagram
# 6 frame types

```
* text              |
* binary            +-- application data
* continuation      |

* ping              |
* pong              +-- control frames
* close             |
```


!SLIDE

```
                      +------------+ |-- new() ---> +-------------------------+
                      |            | |--- << -----> |                         |
                      |            | | finished? -> |                         |
                      |            | |-- valid? --> |    Handshake::Server    |
+----+                |            | | leftovers -> |                         |
|    | <-- read() --| |            | |- version --> |                         |
|    |                |   Socket   | |-- to_s ----> +-------------------------+
| IO |                | Controller |
|    |                |            | |-- new() ---> +-------------------------+
|    | <-- write() -| |            | |--- << -----> | Frame::Incoming::Server |
+----+                |            | |-- next ----> +-------------------------+
                      |            |
                      |            | |-- new() ---> +-------------------------+
                      |            | |-- to_s ----> | Frame::Outgoing::Server |
                      +------------+                +-------------------------+
```


!SLIDE

```rb
class SocketController
  def initialize(io)
    @io        = io
    @handshake = WebSocket::Handshake::Server.new
    @queue     = []
    @state     = :connecting

    loop { parse(io.read) }
  end
```


!SLIDE

```rb
  def parse(data)
    case @state
    when :connecting
      @handshake << data
      return unless @handshake.finished?
      if @handshake.valid?
        leftovers = @handshake.leftovers
        @version  = @handshake.version
        @parser   = WebSocket::Frame::Incoming::Server.new(:version => @version)

        @io.write(@handshake.to_s)
        @queue.each &method(:send)
        @state = :connected
        parse(leftovers)
      end
    when :connected
      @parser << data
      while frame = @parser.next
        handle_frame(frame)
      end
    end
  end
```


!SLIDE

```rb
  def parse(data)
    case @state
    when :connecting # <---------------------- parser state
      @handshake << data
      return unless @handshake.finished?
      if @handshake.valid?
        leftovers = @handshake.leftovers
        @version  = @handshake.version
        @parser   = WebSocket::Frame::Incoming::Server.new(:version => @version)

        @io.write(@handshake.to_s)
        @queue.each &method(:send)
        @state = :connected # <--------------- parser state
        parse(leftovers)
      end
    when :connected # <----------------------- parser state
      @parser << data
      while frame = @parser.next
        handle_frame(frame)
      end
    end
  end
```


!SLIDE

```rb
  def parse(data)
    case @state
    when :connecting
      @handshake << data
      return unless @handshake.finished? # <-- protocol state
      if @handshake.valid? # <---------------- protocol state
        leftovers = @handshake.leftovers
        @version  = @handshake.version # <---- protocol state
        @parser   = WebSocket::Frame::Incoming::Server.new(:version => @version)

        @io.write(@handshake.to_s)
        @queue.each &method(:send)
        @state = :connected
        parse(leftovers)
      end
    when :connected
      @parser << data
      while frame = @parser.next
        handle_frame(frame)
      end
    end
  end
```


!SLIDE

```rb
  def parse(data)
    case @state
    when :connecting
      @handshake << data # <------------------ stream routing
      return unless @handshake.finished?
      if @handshake.valid?
        leftovers = @handshake.leftovers # <-- stream routing
        @version  = @handshake.version
        @parser   = WebSocket::Frame::Incoming::Server.new(:version => @version)

        @io.write(@handshake.to_s)
        @queue.each &method(:send)
        @state = :connected
        parse(leftovers) # <------------------ stream routing
      end
    when :connected
      @parser << data # <--------------------- stream routing
      while frame = @parser.next
        handle_frame(frame)
      end
    end
  end
```


!SLIDE

```rb
  def parse(data)
    case @state
    when :connecting
      @handshake << data
      return unless @handshake.finished?
      if @handshake.valid?
        leftovers = @handshake.leftovers
        @version  = @handshake.version
        @parser   = WebSocket::Frame::Incoming::Server.new(:version => @version)

        @io.write(@handshake.to_s)
        @queue.each &method(:send)
        @state = :connected
        parse(leftovers)
      end
    when :connected
      @parser << data
      while frame = @parser.next #
        handle_frame(frame)      # <---------- message polling
      end                        #
    end
  end
```


!SLIDE

```rb
  def send(message)
    case @state
    when :connecting
      @queue << message
    when :connected
      frame = WebSocket::Frame::Outgoing::Server.new(
        :version => @version,
        :type    => :text,
        :data    => message
      )
      @io.write(frame.to_s)
    end
  end
```


!SLIDE

```rb
  def send(message)
    case @state
    when :connecting # <--------------------- protocol state
      @queue << message
    when :connected # <---------------------- protocol state
      frame = WebSocket::Frame::Outgoing::Server.new(
        :version => @version,
        :type    => :text,
        :data    => message
      )
      @io.write(frame.to_s)
    end
  end
```


!SLIDE

```rb
  def send(message)
    case @state
    when :connecting
      @queue << message # <------------------ I/O sequencing
    when :connected
      frame = WebSocket::Frame::Outgoing::Server.new(
        :version => @version,
        :type    => :text,
        :data    => message
      )
      @io.write(frame.to_s)
    end
  end
```


!SLIDE

```rb
  def send(message)
    case @state
    when :connecting
      @queue << message
    when :connected
      frame = WebSocket::Frame::Outgoing::Server.new(
        :version => @version,
        :type    => :text,
        :data    => message
      )
      @io.write(frame.to_s) # <-------------- NO I/O sequencing
    end
  end
```


!SLIDE

```rb
  def send(message)
    case @state
    when :connecting
      @queue << message
    when :connected
      frame = WebSocket::Frame::Outgoing::Server.new(
        :version => @version, # <------------- protocol state
        :type    => :text,
        :data    => message
      )
      @io.write(frame.to_s)
    end
  end
```


!SLIDE

```rb
  def handle_message(message)
    case message.type
    when :text
      # ...
    when :binary
      # ...
    when :ping
      frame = WebSocket::Frame::Outgoing::Server.new(
        :version => @version,
        :type    => :pong,
        :data    => message.data
      )
      @io.write(frame.to_s)
    when :pong
      # ...
    when :close
      # ...
    end
  end
```


!SLIDE

```rb
  def handle_message(message)
    case message.type
    when :text
      # ...
    when :binary
      # ...
    when :ping # <--------------------------- protocol requirement
      frame = WebSocket::Frame::Outgoing::Server.new(
        :version => @version,
        :type    => :pong,
        :data    => message.data
      )
      @io.write(frame.to_s)
    when :pong # <--------------------------- protocol requirement
      # ...
    when :close # <-------------------------- protocol requirement
      # ...
    end
  end
```


!SLIDE

```rb
  def handle_message(message)
    case message.type
    when :text
      # ...
    when :binary
      # ...
    when :ping
      frame = WebSocket::Frame::Outgoing::Server.new(
        :version => @version, # <------------ protocol state
        :type    => :pong,
        :data    => message.data
      )
      @io.write(frame.to_s)
    when :pong
      # ...
    when :close
      # ...
    end
  end
```


!SLIDE

```rb
  def handle_message(message)
    case message.type
    when :text
      # ...
    when :binary
      # ...
    when :ping
      frame = WebSocket::Frame::Outgoing::Server.new(
        :version => @version,
        :type    => :pong,
        :data    => message.data
      )
      @io.write(frame.to_s) # <-------------- no I/O sequencing
    when :pong
      # ...
    when :close
      # ...
    end
  end
```


!SLIDE title
# An object-oriented approach
## Tell, don’t ask


!SLIDE

```
                      +------------+
                      |            |
                      |            |                   +------------+
                      |            | |-- server() ---> |            |
+----+                |            |                   |            |
|    | <-- read() --| |            | |--- parse() ---> |            |
|    |                |   Socket   |                   |  WebSocket |
| IO |                | Controller | <-- on_message -| |   Driver   |
|    |                |            |                   |            |
|    | <-- write() -| |            | |--- text() ----> |            |
+----+                |            |                   |            |
                      |            | <--- write() ---| |            |
                      |            |                   +------------+
                      |            |
                      +------------+
```


!SLIDE

```rb
class SocketController
  def initialize(io)
    @io     = io
    @driver = WebSocket::Driver.server(self)

    @driver.on :message do |event|
      # ...
    end

    loop { @driver.parse(@io.read) }
  end

  def write(data)
    @io.write(data)
  end

  def send(message)
    @driver.text(message)
  end
end
```


!SLIDE

```rb
class SocketController
  def initialize(io)
    @io     = io
    @driver = WebSocket::Driver.server(self) # <-- pass in reference to self

    @driver.on :message do |event|
      # ...
    end

    loop { @driver.parse(@io.read) }
  end

  def write(data)
    @io.write(data)
  end

  def send(message)
    @driver.text(message)
  end
end
```


!SLIDE

```rb
class SocketController
  def initialize(io)
    @io     = io
    @driver = WebSocket::Driver.server(self)

    @driver.on :message do |event|
      # ...
    end

    loop { @driver.parse(@io.read) } # <---------- tell the driver to parse
  end

  def write(data)
    @io.write(data)
  end

  def send(message)
    @driver.text(message)
  end
end
```


!SLIDE

```rb
class SocketController
  def initialize(io)
    @io     = io
    @driver = WebSocket::Driver.server(self)

    @driver.on :message do |event| # <------------ driver tells you about events
      # ...
    end

    loop { @driver.parse(@io.read) }
  end

  def write(data)
    @io.write(data)
  end

  def send(message)
    @driver.text(message)
  end
end
```


!SLIDE

```rb
class SocketController
  def initialize(io)
    @io     = io
    @driver = WebSocket::Driver.server(self)

    @driver.on :message do |event|
      # ...
    end

    loop { @driver.parse(@io.read) }
  end

  def write(data)
    @io.write(data)
  end

  def send(message)
    @driver.text(message) # <--------------------- tell the driver to send
  end
end
```


!SLIDE

```rb
class SocketController
  def initialize(io)
    @io     = io
    @driver = WebSocket::Driver.server(self)

    @driver.on :message do |event|
      # ...
    end

    loop { @driver.parse(@io.read) }
  end

  def write(data) # <----------------------------- driver tells you
    @io.write(data)                              # what/when to write
  end

  def send(message)
    @driver.text(message)
  end
end
```

