---
title: "Delimited Continuations in Ruby Part 2: Generators and Coroutines"
date: 2014-07-21
comments: true
categories: hacker_school
published: true
---

Last [time](http://blog.ontoillogical.com/blog/2014/07/12/delimited-continuations-in-ruby/), I showed some basic things you can do with delimited continuations. If you're still confused about them (as I am!) another good tutorial is [here](http://community.schemewiki.org/?composable-continuations-tutorial).

Let's dive right in and build some more complicated control structures!

## Generators

Let's start by building what Python calls "Generators." Ruby has Enumerators, which are pretty similar, but I'll call it a generator in order to differentiate my implementation from the Ruby core.

Here's the code I want to write
```ruby
counter = Generator.new do |g| 
  ctr = 0
  while true
    g.yield(ctr)
    ctr += 1
  end
end
```
Then I use it like so

```ruby
counter.next
# => 0
counter.next
# => 1
counter.next
# => 2
# ...
```

Using delimited continuations to build a generator is pretty straight-forward. Our constructor will get a block and surround it with a `prompt` to delimit a continuation. Yielding adds the yielded value into an internal queue, and uses `control` to capture a continuation. When we call that continuation, we will execute the code around the control, which is precisely like entering at where we called yield. Getting the next value of our generator pops a result off the queue and invokes the saved continuation.

The annotated code is below.

```
class Generator
  def initialize(&block)
    # @results will be used to save values yielded by our generator
    @results = []
    # Calling the block is wrapped in a prompt in order to enclose it in a continuation
    DelimR.prompt do
      # The block is passed self, so that it can call the yield method defined below
      block.call(self)
    end
  end
  
  def yield(result)
    # The yield method saves the result it receives, and saves the continuation in @next
    @results << result
    DelimR.control do |k| 
      @next = k
    end
  end
  def next
    # @result is used as a FIFO queue of yielded values
    r = @results.shift

    # an empty @results means that no more values have been yielded
    if r.nil?
      raise "Iterator finished"
    end

    # This invokes the saved continuation, not passing it any values
    @next.call(nil)

    # return the saved @result that was shifted off the bottom of the queue
    r
  end
end
```

## Coroutines

Coroutines, also known as Fibers in ruby-land, are subroutines that can suspend their execution, to be resumed at a later point. They are very useful as a lightweight thread-like construct. They are very similar to generators, and can actually be implemented in terms of [them](http://legacy.python.org/dev/peps/pep-0342/). Besides being useful for asynchronous tasks, ["Co-routines are to state machines what recursion is to stacks"](http://eli.thegreenplace.net/2009/08/29/co-routines-as-an-alternative-to-state-machines/). 

We can can modify our `Generator` class to be a coroutine by renaming `next` to `send` and allowing data to be passed into it, so `foo = c.yield` will alssign whatever is sent to the coroutine to `foo` when execution resumes.

```ruby
class Coroutine
  # initialize and yield are the same as for Generator
  def send(value=nil)
    # @result is used as a FIFO queue of yielded values
    r = @results.shift

    # an empty @results means that no more values have been yielded
    if r.nil?
      raise "Iterator finished"
    end

    # This invokes the saved continuation, not passing it any values
    @next.call(value)

    # return the saved @result that was shifted off the bottom of the queue
    r
  end
end
```

Here's an implementation of the parsing example from Eli Bendersky's blog [post](http://eli.thegreenplace.net/2009/08/29/co-routines-as-an-alternative-to-state-machines/) using the Coroutine class defined above.

```ruby
def hex_encode(str)
  str.split(//).map{ |x| x.ord.to_s(16) }.join
end

frame_receiver = Coroutine.new do |c| 
  while true
    frame = (c.yield(true))
    puts 'Got frame:', hex_encode(frame)
  end
end

unwrap_protocol = Coroutine.new do |c| 
  header = "\x61"
  footer = "\x62"
  dle = "\xAB"
  after_dle_func = lambda { |x| x }
  target = frame_receiver
  # Outer loop looking for a frame header
  #
  while true
    byte = (c.yield(true))
    frame = ''

    if byte == header
      # Capture the full frame
      #
      while true
       
        byte = (c.yield(true))
        
        if byte == footer

          target.send(frame)
          break
        elsif byte == dle
          byte = (c.yield(true))
          frame += after_dle_func.call(byte)
        else
          frame += byte
        end
      end
    end
  end
end



bytes = [0x70, 0x24,
         0x61, 0x99, 0xAF, 0xD1, 0x62,
         0x56, 0x62,
         0x61, 0xAB, 0xAB, 0x14, 0x62,
         0x7
        ].map(&:chr)


bytes.each do |byte|
  unwrap_protocol.send(byte)
end

# Got frame:
# 99afd1
# Got frame:
# ab14
```

## Final thoughts 

Once I had prompt/control, I found implementing generators and coroutines pretty straightforward. I think that prompt/control are easier to reason about then non-delimited continuations. One piece of evidence that I have for this is that Ruby's [implementation](https://github.com/ruby/ruby/blob/ruby_1_8_7/lib/generator.rb) of generators using non-delimited continuations is harder (for me) to understand then the one above using delimited continuations.

Either way, having an abstraction that allows you to implement new control structures is really cool and empowering. For instance, this [paper](http://www.ccs.neu.edu/racket/pubs/pldi93-sitaram.pdf) uses delimited continuations to implement Prolog-like backtracking!

There are still many things I don't understand about continuations, a partial list is below

1. There is a difference between prompt/control and reset/shift. I don't know what it is.
2. I don't entirely understand what `@@keep_delimiter_upon_effect` does in https://github.com/mveytsman/DelimR/blob/master/lib/delimr.rb
3. In the generator example, why doesn't the line `ctr=0` get executed on every iteration?
4. I actually implemented semi-coroutines. For full coroutines I need to be able to transfer exeution between them. My instricnt is that I can do this by calling resume on the target coroutine, but seeing this in the Ruby Fiber documentation makes me think there is something non-trivial I don't understand here:
> You cannot resume a fiber that transferred control to another one. This will cause a double resume error. You need to transfer control back to this fiber before it can yield and resume.
