What's the biggest problem we have in programming? We call it coupling, we call it complexity, we call it 'crap'; it's the tendency of the components in our system to have too much knowledge. Look at this code:

#If-heavy code here

What do you need to know to verify if that code works? You need to know:

* The types of each value returned by the getter methods called
* What different values the getter methods can return in what situations
* What your contract with your callers is, so that you can return the right thing

Look at this code:

#Protocol-implementing code calling a library here

What do you need to know to verify if that code works (and that every line is necessary)? You need to know:

* That the IRC library will let you send messages to a channel before you've joined it
* That the IRC library only supports joining a channel once it's connected to a network
* That the IRC library will respond to PINGs correctly 
* The format of messages returned by the IRC library

What's the common problem here, leading to our code needing to know so much?

Getter methods. What do I mean by that? I mean any method which exposes the state of one component to another component. So that's things like `socrates.is_man?`; it's things like `irc.current_channel`; it's things like `bank_account.current_balance`. As soon as we define and start to rely on these methods, we end up with behaviours spread around the system. The IRCClient class was supposed to do everything to do with communicating with an IRC Server, but in fact it just provides some *methods* for doing so; it can't talk to the server properly unless everything using it uses those methods in the right way and in the right order. If the server's protocol changes, we may have to change the client, or we may have to change everything using the client.

So how do we deal with this?

BURN YOUR GETTERS.

What if we stopped getting the state of other objects? What if we stopped obeying the letter of the Law of Demeter (don't chain method calls) and started obeying its spirit (Tell, Don't Ask)? How would that look?

#James takes over

BUT THERE'S A PROBLEM

We've split our problem - communicating via WebSockets - over multiple components. This does two things; it reduces surface readability, and it changes the shape of our program. Let's talk about readability.

You can take a branching statement:

    case some_string
    when "HELLO"
      io.write("stop shouting")
    when "hello"
      io.write("speak up")
    when "12345"
      io.write("numbers!")
    end

And transform it to a hash lookup:

    io.write({"HELLO" => "stop shouting",
             "hello" => "speak up",
             "12345" => "numbers!"}[some_string])

And then to a polymorphic solution:

    def handler_for(string)
      {"HELLO" => Shouting.new,
       "hello" => Quiet.new,
       "12345" => Numbers.new }[string]
    end

    handler_for(string).write_response(io)

And you can then extend that polymorphic solution to allow new handlers for new strings to be added at runtime. But this comes at the expense of readability; in the original case statement, we can see exactly which strings are supported by the system and what responses they will produce, all in one place. In the hash lookup version, we can still see this, but it's harder to follow. In the polymorphic version with runtime extensibility, we can't know at all until runtime what the system supports and how it will respond to these different inputs. So we've sacrificed readability to get some flexibility. 
