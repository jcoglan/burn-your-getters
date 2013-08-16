18th August, 2012. (That's last year). Eurucamp. Fuelled by Club Mate and an inflated sense of self-worth, a philosopher named Tom Stuart embarks on his most ambitious trolling campaign so far. He sits down next to a programmer named James Coglan, and tells him that, to improve his code, he should stop using getter methods. Over the next 6 months, Tom (oh, that's me, by the way) took his (my) campaign to Twitter, and is amazed to realize that James has actually gone ahead and taken his advice. This is our story.

# Introduction to burning your getters

What's the biggest problem we have in programming? We call it coupling, we call it complexity, we call it 'crap'; it's the tendency of the components in our system to have too much knowledge. Look at this code: 

    def get_trolls
      tweets = @twitter_client.get_recent_tweets
      trolls = []
      tweets.each do |tweet|
        trolls << tweet if tweet.author == "@mortice"
      end
      trolls
    end

What do you need to know to verify if that code works? You need to know:

* The types of each value expected and returned by the getter methods called
* What different values the getter methods can return in what situations
* What your contract with your callers is, so that you can return the right thing

Look at this code:

    class IRCClient
      def join_channel_with_message
        @irc.connect(:freenode, :username => "getterburner")
        while !@irc.connected?
          sleep
        end

        @irc.join("#eurucamp")
        @irc.send("#eurucamp", "Hey everyone!")

        until interrupted? do
          @irc.messages("#eurucamp").each {|m| puts "#{m.author}: #{m.text}"}
        end
      end
    end
    
What do you need to know to verify if that code works (and that every line is necessary)? You need to know:

* That the IRC library only supports joining a channel once it's connected to a network
* That the IRC library will let you send messages to a channel before you've joined it
* That the IRC library will respond to PINGs correctly and you'll stay connected
* The format of messages returned by the IRC library

What's the common problem here, leading to our code needing to know so much?

Getter methods. What do I mean by that? I mean any method which exposes the state of one component to another component. So that's things like `tweet.author`; it's things like `irc.connected?`; it's things like `irc.messages`. As soon as we define and start to rely on these methods, we end up with behaviours spread around the system. The IRC class that we use here was supposed to do everything to do with communicating with an IRC Server, but in fact it just provides some *methods* for doing so; it can't talk to the server properly unless everything using it uses those methods in the right way and in the right order. If the server's protocol changes, we may have to change the client, or we may have to change everything using the client.

So how do we deal with this?

BURN YOUR GETTERS.

What if we stopped getting the state of other objects? What if we stopped obeying the letter of the Law of Demeter (don't chain method calls) and started obeying its spirit (Tell, Don't Ask)? How would that look?

# Readability

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

But is readability really the virtue we claim it is? I want to suggest: only sometimes. Clearly, all other things being equal, we want our code to be as readable as possible, so that the next person to change it doesn't have to twist their brain in knots to understand it. But what's easier to understand than readable code? Code you don't have to read. That is, if our systems are designed so that I can add new functionality to them at runtime - that is, without reading the code - then they've achieved our goal of being easy on the next person to work with them without pursuing readability.

Now, clearly, there are limits to this. If you're only looking at one if-statement, dealing with one piece of non-local state, you may well decide that the flexibility benefits from polymorphizing it away don't outweigh the disadvantages. Polymorphizing is totally a word.

#Iterative -> recursive

How does telling change the shape of code? 

So when you take state from one place and make a decision about that state somewhere else, you're making use of the heap. You pull things into memory, do something with them, and then do your next thing. Whereas when you tell objects to do things and let them behave differently based on that state, you make use of the stack. Even if your caller doesn't use the return value of the command method it has just called, it's still there on the stack until the receiving method completes. And if the receiver calls another command method, that sits on the stack. And so on. Which means, obviously, that you're at greater risk of "stack level too deep" errors, but also that there aren't really any intermediate steps in your system's execution: you go from your first call to completion, and if you never return anything up the stack, the execution is kind of opaque until you reach the end. Let me show you what I mean. Has anyone read my namesake's book, _Understanding Computation_?

Cool book. So in chapter 2, other Tom talks about writing a parser and interpreter for a simple programming language called Simple. His parser main body looks like this:

    def run
      while expression.reducible?
        puts expression
        step
      end

      puts expression
    end
  
    def step
      self.expression = expression.reduce
    end

So here, we're taking an arithmetical expression like (2 + 2) * (12 / 3), and we're reducing it step by step into its parts until we get to an irreducible expression like "16", where we stop. This means that we can inspect and specify clearly the order of rules to be applied to the expression.

Now, as a zealous conditional hater, I read this, and thought: "hmm, so we call a query method, and do something different to the object based on that? That seems fishy to me. Let's try and write this without ifs." And, of course, as I've shown, you can do that, but I found that I unintentionally went from what's called a 'small-step' semantics of the language to a big-step semantics. My main body looks like this:

    def run
      expression.reduce
    end

And the expressions themselves handle whether or not reduce something. But note that, because of the recursive nature of this solution, if I want to show the expression currently being reduced, I have to put my `puts` statement in each expression. And even then, the output won't make sense unless the logging takes into account the calling context. So I've lost the ability to control in detail or easily to inspect the order of expression evaluation. And this is behavioural change which has come about purely by making refactorings which add up to the same end result.
