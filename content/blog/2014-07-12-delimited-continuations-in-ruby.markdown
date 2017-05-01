---
title: "Delimited Continuations in Ruby Part 1"
date: 2014-07-12
comments: true
categories: hacker_school
---

For the past few days at Hacker School, I've been exploring continuations. Continuations are hard to describe. Basically, a continuation represents the execution state of a program at a point. Capturing the continuation and invoking it later allows you to come back to that point in the programs execution. Continuations can be used to implement complicated control flow constructs.

If that was complicated, here's a sandwich metaphor from [Luke Palmer](https://groups.google.com/forum/#!msg/perl.perl6.language/-KFNPaLL2yE/_RzO8Fenz7AJ):

>Say you're in the kitchen in front of the refrigerator, thinking about a sandwich. You take a continuation right there and stick it in your pocket. Then you get some turkey and bread out of the refrigerator and make yourself a sandwich, which is now sitting on the counter. You invoke the continuation in your pocket, and you find yourself standing in front of the refrigerator again, thinking about a sandwich. But fortunately, there's a sandwich on the counter, and all the materials used to make it are gone. So you eat it. :-)

Another way to put it: a captured continuation is a label, and to invoke it is to GOTO.

## The Basics

On that note, let's start by building a simple GOTO.

In Ruby, you need to `require 'continuation'` to enable continuations. `callcc` takes a block, and passes it the current continuation as parameter. That continuation can be invoked later.

```ruby
require 'continuation'
$labels = {}

def label(label_name)
  callcc { |continuation|   $labels[label_name] = continuation }
end

def goto(label_name)
  unless $labels.has_key? label_name
    raise "No label #{label_name}"
  end
  $labels[label_name].call
end
```

We can then build a for loop

```ruby
i = 1
puts "entering loop"
label "loop"
if i < 10
  puts i
  i += 1
  goto "loop"
end
puts "loop done"
puts "i is #{i}"
```

We can even build a reverse GOTO (known as the COMEFROM, implementation courtesy of [Wikipedia](https://en.wikipedia.org/wiki/COMEFROM#Practical_uses)).

{% codeblock ruby %}
$come_from_labels = {}
 
def label(l)
    if $come_from_labels[l]
        $come_from_labels[l].call
    end
end
 
def come_from(l)
    callcc do |block|
        $come_from_labels[l] = block
    end
end
{% endcodeblock %}

## Problems Abound

You can implement pretty much any control flow you want with continuations, but it may be [hairy](http://okmij.org/ftp/continuations/against-callcc.html#traps).

To give an example of the kind of problems you face when working with traditional continuations, here's a bug I ran into when writing the GOTO example above. My orginal code for `label` looked like this: 

```ruby
def label(label_name)
   $labels[label_name] = callcc { |continuation| continuation }
end
```

Running it, I got the following output:

```
entering loop
1
2
goto.rb:15:in `goto': undefined method `call' for nil:NilClass (NoMethodError)
	from goto.rb:27:in `<main>'
```

Can you guess the issue? When `goto` was called, it returned to the place where `callcc` was called. That means that whatever was passed to the continuation would now be assigned to `$labels[label_name]`. We didn't pass anything to the continuation, and so `$labels[label_name]` becomes `nil`.

To further illustrate the point:

```ruby
$continuation = callcc { |continuation| continuation }
puts "$continuation is #{$continuation}"
# "$continuation is #<Continuation:0x007fbbaa0193f8>"
$continuation.call("hello world")
puts "$continuation is #{$continuation}"
# "$continuation is hello world"
```

It seems like it would be convent to separate the concept of shifting control from the place we want to return to.

## Enter delmited continuations

[Delimited continuations](https://en.wikipedia.org/wiki/Delimited_continuation) are way of capturing a fixed slice of computation instead of the unlimited computation captured by a regular continuation.

I wrote a little library called [DelimR](https://github.com/mveytsman/delimr) that implemnts delimited continuations for ruby by using the native `callcc` construct. I made a direct port of Oleg Kiselyov's scheme [implementation](http://okmij.org/ftp/continuations/implementations.html#delimcc-scheme). In an inversion of Knuth's famous [line](http://staff.science.uva.nl/~peter/knuthnote.pdf), the code seems to work, but I don't understand it.

Delimited continuations have two operations. `prompt` marks a continuation, when the continuation is invoked, it will return to where prompt was called. `control` allows us to capture the continuation.

It's best to demonstrate this with some examples. `prompt` on it's own is a no-op.
```ruby
DelimR.prompt { 123 }
# => 123
```
Inside of `prompt`, you can call `control`, and pass it a block that receives a continuation as an argument
```ruby
DelimR.prompt { 1 + DelimR.control { |k| k.call(2) }}
# => 3
```
`k` captures the computation that surrounds `control` inside of `prompt`. If we don't call `k`, that computation is lost
```ruby
DelimR.prompt { 1 + DelimR.control { |k| 2 }}
# => 2
```
And finally, the most exciting part,
```ruby
DelimR.prompt { 1 + DelimR.control { |k| k.class }}
# => Proc
```
This is really cool! In delimited continuations, `k` is a `Proc`, which is Ruby-speak for a function. Delimited continuations are also called composable continuations, because you can *compose* `k` like any other function.

Here's an example:

```ruby
DelimR.prompt { 1 + DelimR.control { |k| k.call(k.call(k.call(2))) } + 7 }
# => 26
```

How did 26 get there? The way to think about this is to realize that `k` captures the execution around `control`. In this case it's `k = lambda { |x| 1 + x + 7}`. So, in the above statement 

```ruby
k.call(k.call(k.call(2))) = 1 + (1 + (1 + (2) + 7) + 7) + 7 = 26
```

Delimited/composable continuations are really cool and powerful abstractions. Next time, I'm going to show you how to implement generators and co-routines with them!

