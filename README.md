# after_do [![Gem Version](https://badge.fury.io/rb/after_do.png)](http://badge.fury.io/rb/after_do)[![Build Status](https://travis-ci.org/PragTob/after_do.png?branch=master)](https://travis-ci.org/PragTob/after_do)[![Code Climate](https://codeclimate.com/github/PragTob/after_do.png)](https://codeclimate.com/github/PragTob/after_do)[![Coverage Status](https://coveralls.io/repos/PragTob/after_do/badge.png)](https://coveralls.io/r/PragTob/after_do)

after_do is simple gem, that allows you to execute some blocks (callbacks) after specific method of a class are called. If the class extends `AfterDo` you can simply do this by

```
MyClass.after :some_method do whatever_you_want end
```

Why would you want to do this? Well to fight cross-cutting concerns such as logging. E.g. there are concerns in an applications that apply to multiple objects (e.g. they cross-cut). A popular example is logging - you might want to log multiple actions but logging is not the primary concern of the class in question. With logging you litter all your code with logging statements - that concern is spread over many files. With after_do you could put all the logging in one file. Other use cases include gathering business statistics or redrawing timing of elements. Personally I extracted this gem from a project where I wanted to decouple my domain objects from the way they are saved (for fun and profit!).
This should generally not be done to alter behavior of the class and its instances - this makes programs more confusing rather than easier to understand.

The idea for this is inspired by Aspect Oriented Programming - e.g. do something when specific methods are executed. However I doubt that this formally fulfills the lingo (join points, aspects, advice...)

after_do has no external runtime dependencies and the code is around 160 lines (blank lines and documentation included) with lots of small methods. So simplecov reports there are a little above 70 relevant lines code (it ignores blank lines, docs etc.).

## Installation

Add this line to your application's Gemfile:

    gem 'after_do'

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install after_do

## Usage

This section is dedicated to show of the general usage and effects of after_do. You can also check out the samples directory for some samples or the specs in the spec folder.

### General usage

In order to use after_do the class/module you want to use it with first has to extend the `AfterDo` module. You can do this right in the class definition or afterwards like this: `MyClass.extend AfterDo`.
With this setup you can add a callback to `method` like this:

```ruby
MyClass.after :method do magic end
```

With after_do you can do simple things like printing something out every time a method is called as in this example:

```ruby
class Dog
  def bark
    puts 'Woooof'
  end
end

Dog.extend AfterDo
Dog.after :bark do puts 'I just heard a dog bark!' end

dog = Dog.new
dog2 = Dog.new

dog.bark
dog2.bark

# Output is:
# Woooof
# I just heard a dog bark!
# Woooof
# I just heard a dog bark!

```

### How does it work?

When you attach a callback to a method with after_do what it basically does is it creates a copy of that method and then redefines the method to basically look like this (pseudo code):

```ruby
execute_before_callbacks
return_value = original_method
execute_after_callbacks
return_value
```

To do this some helper methods are defined in the AfterDo module. As classes have to extend the AfterDo module all the methods that you are not supposed to call yourself are prefixed with `_after_do_` to minimize the risk of method name clashes. The only not prefixed method are `after`, `before` and `remove_all_callbacks`.

### Getting a hold of the method arguments and the object

With after_do both the arguments to the method you are attaching the callback to and the object for which the callback is executed are passed into the callback block.

So if you have a method that takes two arguments you can get those like this:

```ruby
MyClass.after :two_arg_method do |argument_one, argument_2| something end
```

The object itself is passed in as the last block argument, so if you just care about the object you can do:

```ruby
MyClass.after :two_arg_method do |*, obj| fancy_stuff(obj) end
```

Of course you can get a hold of the method arguments and the object:

```ruby
MyClass.after :two_arg_method do |arg1, arg2, obj| something(arg1, arg2, obj) end
```

If you do not want to get a hold of the method arguments or the object, then you can just don't care about the block parameters :-)

Here is an example showcasing all of these:

```ruby
class Example
  def zero
    # ...
  end

  def two(a, b)
    # ...
  end

  def value
    'some value'
  end
end

Example.extend AfterDo

Example.after :zero do puts 'Hello!' end
Example.after :zero do |obj| puts obj.value end
Example.after :two do |first, second| puts first + ' ' + second end
Example.after :two do |a, b, obj| puts a + ' ' + b + ' ' + obj.value end
Example.after :two do |*, obj| puts 'just ' +  obj.value end

e = Example.new
e.zero
e.two 'one', 'two'
# prints:
# Hello!
# some value
# one two
# one two some value
# just some value
```

### Attaching a callback to multiple methods

In after_do you can attach a callback to multiple methods by just listing them:

```ruby
SomeClass.after :one_method, :another_method do something end
```

Or you could pass in an Array of method names:

```ruby
SomeClass.after [:one_method, :another_method] do something end
```

So for example if you have an activity and want the activity to be saved every time you change it, but you don't want to mix that persistence concern with what the activity actually does you could do something like this:

```ruby
persistor  = FilePersistor.new
Activity.extend AfterDo
Activity.after :start, :pause, :finish, :resurrect,
             :do_today, :do_another_day do |activity|
  persistor.save activity
end
```

Doesn't that seem a lot drier then calling some save method manually after each of those in addition to separating the concerns?

### Attaching multiple callbacks to the same method

A method can have as many callbacks as a Ruby Array can handle (although I do not recommend you to have many callbacks around). So this works perfectly fine:

```ruby
MyClass.after :method do something end
MyClass.after :method do another_thing end
```

The callbacks are executed in the order in which they were added.

### Working with inheritance

after_do also works with inheritance. E.g. if you attach a callback to a method in a super class and that method is called in a sub class the callback is still executed.

See this sample:

```ruby
class A
  def a
    # ...
  end
end

class B < A
end

A.extend AfterDo
A.after :a do puts 'a was called' end

b = B.new
b.a #prints out: a was called
```

### Usage from within a class

If you got some repetitive tasks, that needs to be done after/before a lot of methods in a class then you can also use after_do for this. This works a bit like `before_action`/`after_action` which you might know from Ruby on Rails.

E.g. like this:

```
class MyClass
  extend AfterDo

  # ...

  after :my_method, :method2, :m3 do |args*, instance| instance.a_method end
end
```

See this example:

```ruby
class Team
  extend AfterDo
  
  def add_member(member)
    # ...
  end
  
  def remove_member(member)
    # ..
  end
  
  def change_name(new_name)
    # ..
  end
  
  def save
   # ..
   puts 'saving...'
  end
  
  after :add_member, :remove_member, :change_name do |*, team| team.save end
end

team = Team.new
team.add_member 'Maren'
team.change_name 'Ruby Cherries'
team.remove_member 'Guilia'

# Output is:
# saving...
# saving...
# saving...
```

### Removing callbacks

You can remove all callbacks you added to a class by doing:

```ruby
MyClass.remove_all_callbacks
```

Note that this not remove callbacks defined in super classes.

### Errors

There are some custom errors that after_do throws. When you try to add a callback to a method which that class does not understand it will throw `AfterDo::NonExistingMethodError`.

When an error occurs during one of the callbacks that are attached to a method it will throw `AfterDo::CallbackError` with information about the original error and where the block/callback causing this error was defined to help pinpoint the error.


## Is there a before method?

Yes. It works just like the `after` method, but the callbacks are executed before the original method is called. You can also mix and match before and after calls.

Before for me is a far less common use case, that's why it was only added later (in the 0.2 release).

Here is a small sample:

```ruby
require 'after_do'

class MyClass
  attr_accessor :value
end

MyClass.extend AfterDo
MyClass.after :value= do |*, obj| puts 'after: ' + obj.value.to_s end
MyClass.before :value= do |*, obj| puts 'before: ' + obj.value.to_s end

m = MyClass.new
m.value = 'Hello'
m.value = 'new value'

# Output is:
# before:
# after: Hello
# before: Hello
# after: new value
```

### Method granularity

after_do works on the granularity of methods. That means that you can only attach callbacks to methods. This is no problem however, since if it's your code you can always define new methods. E.g. you want to attach callbacks to the end of some operation that happens in the middle of a method just define a new method for that piece of code.

I sometimes do this for evaluating the block, as I want to do something when that block finished evaluating so I define a method `eval_block` wherein I just evaluate the block.

## Is this a good idea?

Always depends on what you are doing with it. As many things out there it has its use cases but can easily be misused.

### Advantages

- Get cross cutting concerns packed together in one file - don't have them scattered all over your code base obfuscating what the real responsibility of that class is
- Don't repeat yourself, define what is happening when in one file
- I feel like it helps the Single Responsibility principle, as it enables classes to focus on what their main responsibility is and not deal with other stuff. I initially wrote this gem when I wanted to decouple an object of my domain from the way it is saved.

### Drawbacks

- You lose clarity. With callbacks after a method it is not immediately visible what happens when a method is called as some behavior might be defined elsewhere.
- You could use this to modify the behaviour of classes everywhere. Don't. Use it for what it is meant to be used for - a concern that is not the primary concern of the class you are adding the callback to but that class is still involved with.

A use case I feel this is particularly made for is redrawing. That's what we use it for over at [shoes4](https://github.com/shoes/shoes4). E.g. we have multiple objects and different actions on these objects may trigger a redraw (such changing the position of a circle). This concern could be littered all over the code base. Or nicely packed into one file where you don't repeat yourself for similar redrawing scenarios and you see all the redraws at one glance. Furthermore it makes it easier to do things like "Just do one redraw every 1/30s" (not yet implemented).

## Does it work with Ruby interpreter X?

Thanks to the awesome [travis CI](https://travis-ci.org/) the specs are run with MRI 1.9.3, 2.0, 2.1, the latest jruby and rubinius releases in 1.9 mode. So in short, this should work with all of them and is aimed at doing so :-)

## Contributing

Contributions are very welcome. Whether it's an issue or even a pull request. For pull requests the you can use the following flow:

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request

I'd also really appreciate spec only pull requests or bug reports with a failing spec/minimal example as this makes fixing it a lot easier =)

Thanks in advance for all contributions of any kind!
