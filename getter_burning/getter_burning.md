!SLIDE title
# Removing query methods saved my code
## James Coglan & Tom Stuart


!SLIDE

![Eurucamp](logo-1024.png)


!SLIDE

```rb
  def get_trolls
    @twitter_client.get_recent_tweets.select do |t|
      t.author == "mortice"
    end
  end
```


!SLIDE

```rb
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
```


!SLIDE

![Burn your getters](byg.jpg)
