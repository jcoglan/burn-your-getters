!SLIDE

#But there's a problem

!SLIDE

```ruby
    case some_string
    when "HELLO"
      io.write("stop shouting")
    when "hello"
      io.write("speak up")
    when "12345"
      io.write("numbers!")
    end
```

!SLIDE

```ruby
  io.write({"HELLO" => "stop shouting",
           "hello" => "speak up",
           "12345" => "numbers!"}[some_string])
```

!SLIDE
```ruby
  def handler_for(string)
    {"HELLO" => Shouting.new,
     "hello" => Quiet.new,
     "12345" => Numbers.new }[string]
  end

  handler_for(string).write_response(io)
```

!SLIDE

polymorphize

Pronunciation: /ˌpɒlɪˈmɔːfɪz/

*verb, transitive*: To make polymorphic. Definitely a real word.
