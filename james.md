# On tell-don't-ask

Let's look at an example where tell-don't-ask improves the design of a system.
I work on a WebSocket server, and a while ago some people asked me if they could
reuse the parsers from it to add WebSocket support to other systems like
Celluloid and Puma. I said they could, and set about documenting my parsers so
other people could use them. But as I was doing so I realized the design was no
good; protocol details were leaking out of my library and would need to be
reimplemented by the user.

If you're not familiar, WebSocket looks like this:

    GET /eurucamp HTTP/1.1
    Upgrade: websocket
    Connection: Upgrade
    Host: localhost:6001
    Origin: https://github.com
    Sec-WebSocket-Key: 6V3e74K7aT/sxaUL/d8k0A==
    Sec-WebSocket-Version: 13

    81 8b 22 4f b2 dd 6a 2a de b1
    4d 6f c5 b2 50 23 d6 ...

That is, you get an HTTP GET request with some special headers, followed by a
stream of bytes that represent a sequence of frames. Those frames can be text
or binary messages, they can continue a previous message, and there are control
frames like ping, pong and close. There are multiple versions of the WebSocket
protocol, so you need to look at the headers to decide which version is being
used, then interpret the byte stream according to that version.

My original design for this was similar in style to what the 'websocket' gem
does: you have a handshake parser, and once that's complete you replace it with
a message parser of the right version. So, you start your socket object up with
its handshake parser:

    class WebSocket
      def initialize(io)
        @io        = io
        @handshake = WebSocket::Handshake::Server.new
        @state     = :connecting

        loop { parse(io.read) }
      end

and when data comes in, you give to the handshake parser until it says it's
complete:

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

There's a lot of complexity here: you need to maintain parser state yourself,
and remember to hand any data not processed by the handshake parser off to the
message parser, you need to take the version identifer from the handshake and
give it to the message parser, you need to poll the parser for messages.

And the server using *this* class will probably need to do the same thing by
polling it for responses. With this code, we can't just return the handshake
response after we're done parsing the request, because then we'll forget to
parse the leftover message bytes. And what if the data contains the second half
of one message frame and two thirds of the next one? What can't return a message
from our parse() method so we'll have to let the client poll a queue.

We see similar stateful problems when trying to send messages to the client. The
'websocket' library makes us give the version back again, even though the
version is determined by the handshake and all future messages must use that
version. We also have to maintain a queue for messages we attempt to send before
the handshake is complete.

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
        
And then you have control frames: frames the server *has* to respond to in a
certain way according to the protocol. For example if it receives a ping frame
it must return a pong with the same data:

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

And there's a bunch more stuff the 'websocket' gem forces you to implement by
hand, because it's based on this model where you ask it to parse some data, you
query the result, and then you decide what to do next, often by asking it to
format something for you.

But this doesn't solve the problem at hand, namely: let's have a WebSocket
protocol library you can use with any I/O system. Yes, you can use any I/O
system, but you also need to implement half the protocol yourself. The code
we've seen is constantly asking the websocket library what to do; if you forget
to ask the right question at the right time or with the wrong parameters, it
changes the meaning of the rest of the input stream, leading to broken behaviour
an attacker could exploit.

A protocol is not just some parsing and formatting functions, it's a sequence of
actions that must be done in a certain order for two parties to communicate. The
protocol library therefore needs to take an *active* role, driving your code and
telling it when to do certain things, rather than you asking it all the time.
This makes it much easier for the user to deploy the protocol correctly and
securely.

The protocol library should be telling you what to do, but how? Well, there's a
clue in the code we've seen so far which is that the only real command in there
is @io.write(), which tells a TCP socket what to send back to the client. We
know there's a bunch of protocol details we don't want in our app, so let's push
those down and leave only the bit we want to vary: the I/O integration. Let's
tell the protocol library what we received, and let it tell us what to send
back.

We can achieve this by routing all TCP data to the protocol driver, and giving
it a method to tell us what to write back to the socket.

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

When we receive data, we tell the driver to parse it. When we want to send a
message, we tell the driver. And when the driver has data it needs to send over
the socket, it tells us by calling out write() method. It notifies us about
incoming messages the application must handle using an event, rather than us
having to loop over every incoming message frame ourselves. It can handle pings
itself rather than expecting us to wire the ping/pong mechanism up ourselves.

This radically simplifies the design: all the protocol details are in the driver
library, and all we're left with is glueing this up to some I/O and a high-level
API. Because the driver encapsulates protocol state and can talk back to us, it
can control when the handshake response is sent, when messages are sent, it can
automatically reply to ping frames, it can remember which protocol it should be
using, and all sorts of other bits of control that must be implemented
correctly. You never run into problems where the protocol didn't do the right
thing just because you forgot to ask.

Letting a component tell others what to do rather than having it sit dormant
until asked to do something, gives the component more agency and power. In this
case it lets it ensure the WebSocket protocol functions correctly and securely,
without the user needing to read the RFC.


# On return values

We've discussed how giving an object the power to tell its colloborators what to
do gives it more power than having it return answers when other objects ask
questions of it. But we still need to consider return values and their role in
our programs. In Ruby, every method implicitly returns the value of its last
expression, and the caller can use this value and introduce accidental coupling,
so it pays to be deliberate about what you return.

In our WebSocket example, it makes sense for the parse() method to return
nothing. You feed data into it, and once you've fed in enough data to form a
message, the parser will call you back. There is no other useful feedback the
parse() method can give to a single call, since its results will usually be the
result of aggregating several calls worth of input. So it can happily return
nil.

However, it does make sense for the send() method to return something. Even
though its response is to call us back with the write() method, it might not do
that immediately. If we're still in the handshake phase, the message might be
queued and write() will be called later. The socket may have closed, meaning the
parser should not emit any more messages. So, some feedback about what will be
done with our message might help. You might return a boolean to indicate whether
the message will ever be sent or not. You might raise an exception if sending
messages at this stage is illegal. Either way, the application may need this
feedback and it's a viable use for an immediate return value.

Going beyond return values we've seen two more ways for the protocol driver to
send information to our code. It can, since it holds a reference to our object,
call our methods. And, it can emit events that we listen to to find out what's
going on.

    class WebSocket
      def initialize(io)
        @io     = io
        @driver = WebSocket::Driver.server(self)

        @driver.on(:message) { |e| do_something }

        loop { @driver.parse(@io.read) }
      end

      def write(data)
        @io.write(data)
      end
    end

Why doesn't it use method calls for both types of feedback, or use events for
both. Well, one type of feedback is mandatory: if the application doesn't write
the correct byte stream out to the socket, the protocol is broken. An
application that cannot do this should raise errors, and requiring it to
implement a certain method is a good way of doing this.

But listening to incoming messages is optional: it's totally fine to write a
WebSocket server that ignores all incoming messages. Implementing message
notification as a method call would mean apps would have to implement it to
avoid errors, even if they didn't care about the messages. So, using an event
lets the app opt in to optional behaviour.

If it's not clear whether a method should return anything, consider having it
explicitly return nil. Leaking the last internal computed value creates
coupling, and having it return self doesn't give the caller any information it
didn't already have, but encourages it to rely on this behaviour, painting you
into a corner if you want to add a real return value later.
