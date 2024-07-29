---
author: "Nigel Brookes-Thomas"
title: "Non-Blocking Ruby Methods"
description: "Make light work of asynchronous Ruby"
draft: false
date: 2024-07-26
tags: ["Ruby", "Async", "Fibers", "Today I Learned"]
categories: ["TIL", "Ruby"]
ShowToc: true
TocOpen: true
---

There are lot of options for concurrency in Ruby, from spawning child processes to using threads, ractors and Async. Each have their merits and drawbacks, but one of the simplest and most effective ways to run code concurrently is to use Fibers.

In my use-case, I want to do some work and, in the background, perform some network requests. Network I/O is blocking, so I want to make sure I'm doing productive work on the main thread while I wait for the network requests to complete.

## Fibers
Fibers are lightweight but fairly low-level primitives. They allow you to pause and resume execution at will. Whenever they pause, they yield control back to the caller, which can then resume the Fiber at a later time. This is great for enumerators which can process a loop and return the computed value to the caller.

In my case I want to yield control whenever an operation blocks. Fortunately, Ruby has introduced a Fiber scheduler interface and the `async` gem provides a high-level API which handles the gnarly details of just this use-case.

## Example
With this in my Gemfile:
```ruby
gem "async", "~> 2.14"
```

Here's a simple example of how to use the `async` fiber scheduler to run a non-blocking method:

```ruby
module API
  scheduler = Async::Scheduler.new
  Fiber.set_scheduler(scheduler)

  def self.perform(iteration)
    Fiber.schedule do
      puts "Making request...#{iteration}"
      req = HTTParty.get('http://localhost:9292')
      puts " \tDone with request #{iteration}: #{req.body[0..30]}"
    end
  end
end
```

When this module loads, `Fiber.set_scheduler(Async::Scheduler.new)` installs a `Async::Scheduler` as the fiber scheduler on the current thread. In my case, this is the main thread.

A call to `API.perfom` will create the fiber, register it with the scheduler and start running it. The networking request via `HTTParty.get` blocks -- this causes the fiber to yield control back to the scheduler. The scheduler can then run other fibers, including the main thread, until the network request completes.

Let's put it to use:

  ```ruby
  require 'async/scheduler'
  require 'httparty'
  require_relative 'api'

  puts 'Starting up...'
  5.times do |i|
    API.perform(i)
  end
  puts 'Finished!'
```

The code will schedule 5 fibers to run concurrently, each making a network request. The main thread will continue to run and print 'Finished!' before the network requests complete.

To try this out, I have a super simple Roda web server running on `http://localhost:9292`:

```ruby
require 'roda'

class App < Roda
  route do |r|
    # GET / request
    r.root do
      sleep(rand(0.3..1)) # Simulate some work
      'Hello world!'
    end
  end
end

run App.freeze.app
```

When I run the the client code, I see the following output:

```shell
Starting up...
Making request...0
Making request...1
Making request...2
Making request...3
Making request...4
Finished!
 	Done with request 3: Hello world!
 	Done with request 4: Hello world!
 	Done with request 1: Hello world!
 	Done with request 2: Hello world!
 	Done with request 0: Hello world!
```

The main thread continues to run while the network requests are in progress. When the requests complete, the fibers are resumed and the results are printed to the console. All the fibers run to completion before the program exits. Nice.

## Bonus section
Roda is pretty speedy framework but I'm running here with the basic Rackup server. I can queue about 500 Fibers before saturating the server and it starts rejecting connections. The Falcon web server is written by Samuel Williams, the same author as the Async gem, and is designed to work well with Async. It's a great choice for running high-performance Ruby web servers, so let's try that.

To use Falcon, add it to your server's Gemfile:

```ruby
gem "falcon", "~> 0.47.7"
```

Because I want to start it without TLS, I run it like this:

```shell
bundle exec falcon serve -b http://localhost:9292
```

Now I can run the client code and queue up as many fibers as I like. Falcon will handle them all without breaking a sweat. I can hit 2,000 connections and exhaust my client machine's file handles before Falcon starts to struggle. That's a lot of concurrent requests!

Timing the execution tells me this:

```shell
$ time bundle exec ruby main.rb
bundle exec ruby main.rb  0.64s user 0.25s system 45% cpu 1.958 total
```

Two thousand requests in under 2 seconds. Not bad for a single-threaded Ruby program!
