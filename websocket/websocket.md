!SLIDE title

# Burn your getters
## How avoiding query methods improves design


!SLIDE

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


!SLIDE

```rb
class WebSocket
  def initialize(io)
    @io        = io
    @handshake = WebSocket::Handshake::Server.new
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
        @state    = :connected

        @io.write(@handshake.to_s)
        parse(leftovers)
      end
    when :connected
      @parser << data
      while message = @parser.next
        handle_message(message)
      end
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
    when :close
      # ...
    end
  end
```


!SLIDE

```rb
class WebSocket
  def initialize(io)
    @io     = io
    @driver = WebSocket::Driver.server(self)

    @driver.on(:message) { |e| emit_message(e.data) }

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
