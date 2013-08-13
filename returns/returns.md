!SLIDE title
# Return values


!SLIDE

```rb
  def parse(data)
    @reader.put(data.bytes)
    buffer = true
    while buffer
      case @stage
      when 0
        buffer = @reader.read(1)
        parse_opcode(buffer[0]) if buffer
      when 1
        # ...
    end # <----------------------------------- implicit return
  end
```


!SLIDE

```rb
  def parse(data)
    @reader.put(data.bytes)
    buffer = true
    while buffer
      case @stage
      when 0
        buffer = @reader.read(1)
        parse_opcode(buffer[0]) if buffer
      when 1
        # ...
    end
    
    self # <--------------------------------- explicit return
  end
```


!SLIDE

```rb
  def parse(data)
    @reader.put(data.bytes)
    buffer = true
    while buffer
      case @stage
      when 0
        buffer = @reader.read(1)
        parse_opcode(buffer[0]) if buffer
      when 1
        # ...
    end
    
    true # <--------------------------------- explicit return
  end
```


!SLIDE

```rb
  def parse(data)
    @reader.put(data.bytes)
    buffer = true
    while buffer
      case @stage
      when 0
        buffer = @reader.read(1)
        parse_opcode(buffer[0]) if buffer
      when 1
        # ...
    end
    
    !!?! # <--------------------------------- explicit return
  end
```


!SLIDE

```rb
  def parse(data)
    @reader.put(data.bytes)
    buffer = true
    while buffer
      case @stage
      when 0
        buffer = @reader.read(1)
        parse_opcode(buffer[0]) if buffer
      when 1
        # ...
    end
    
    nil # <---------------------------------- explicit return
  end
```


!SLIDE

```rb
  def text(message)
    if @ready_state < 0
      queue(:text, message) 
      return true # <------------------------ explicit return
    end

    return false unless @ready_state == 1 # < explicit return

    frame = format_frame(:text, message)
    @client.write(frame)

    true # <--------------------------------- explicit return
  end
```


!SLIDE title
# Other feedback mechanisms
## Events or method calls?


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
end
```


!SLIDE

```rb
class SocketController
  def initialize(io)
    @io     = io
    @driver = WebSocket::Driver.server(self)

    @driver.on :message do |event| #     An event listener
      # ...                        # <-- Used to opt-in to being told about
    end                            #     interesting changes

    loop { @driver.parse(@io.read) }
  end

  def write(data)
    @io.write(data)
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

  def write(data)   #                    A method call
    @io.write(data) # <----------------- Used when handling a certain
  end               #                    message is mandatory
end
```

