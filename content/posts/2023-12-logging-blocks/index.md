---
author: "George Bell"
title: "TiL: Save CPU cycles by logging blocks"
description: "A look at passing blocks into."
draft: false
date: 2023-12-15
tags: ["Ruby", "Logging"]
categories: ["Ruby"]
ShowToc: true
TocOpen: true
---

We log a lot in our functional tests. In fact, we log so much we wrote our own logger called [Herodotus](https://github.com/dvla/herodotus) that extends the default Ruby logger so we can have a correlation id added to the log to help keep track of which scenario a given log is from[^1]. But with that much logging, we want to keep things lightweight. That's where we want to make sure our logs aren't doing more work than we expect.

[^1]: All the examples in this post are written using Herodotus, but as this is functionality we've inherited from the built-in logger, the same principles apply there

## Log levels

Log levels are great. They let you keep production logs clear of anything that doesn't require looking at while allowing lower level environments to output verbose debugging information as required, all without having to add or remove log statements to your code as it moves between environments. In Ruby, you can control what the lowest level of logs that'll be output in the following way:

```ruby
logger = DVLA::Herodotus.logger
logger.level = Logger::WARN
```

In the above example, we've set the minimum level to `warn`. Let's log some things and see what the output is:

```ruby
logger.info('Application starting')
logger.warn('DB connection timed out. Reconnecting')
```

```sh
[2023-12-15 09:09:49 45f91875] WARN -- : DB connection timed out. Reconnecting
```

As you can see, we're only getting one log statement, with the `info` not being printed. But what does that mean under the bonnet? Let's take a look at what happens if we pass in a more complex parameter. 

## Passing parameters

Consider the following code snippet: 

```ruby
logger.info("Current active transactions: #{TransactionCounter.count_transactions}")
```

Not much fancier than what we were doing previous. We're now just interpolating the response from the method `count_transactions` into the message we are logging out. We don't know what is in that method but it could be quite a costly call with lots of calculations going on behind it, so if we are in an environment where we aren't logging anything at `info` level we probably don't want to make that call. Now, suppose we add the following line somewhere in `count_transactions`:

```ruby
puts "Starting transaction count"
```

This doesn't change the logic in the method, but makes it nice and obvious if we're executing the code in there. Let's have a look at the output when we hit the log statement:

```sh
Starting transaction count
```

This happens because the string interpolation is happening before we get into the logger and find out that we aren't logging at this level. So, how to prevent this?

## Passing blocks

Let's make a small change to our code:

```ruby
logger.info { "Current active transactions: #{TransactionCounter.count_transactions}" }
```

Now, we're passing a block into the logger and the logger will, assuming it is logging out the current log level, log out the value returned by the block. The block above will return the same string as before, so let's take a look and see if anything is logged this time:

```sh
 
```

That's more like it. This time we aren't calling `count_transactions`. Now, let's take a look at why that is.

Whereas before we were seeing the string fully interpolated before the `info` method was called, we're now offloading that into the block we're passing into the logger. As the logger is now in control of calling that block to get the value to log out, it defers doing so until it knows if it should be logging at the current level. Thus, for logs that are too low level, we don't make any expensive calls and save a few CPU cycles. 