---
author: "George Bell"
title: "TiL: Double Splat Nil in Ruby"
description: "What is a Double Splat Nil and how does it help clamp down on unwanted named arguments"
draft: false
date: 2025-06-09
tags: ["Ruby", "nil", 'double splat', "Today I Learned"]
categories: ["TIL", "Ruby"]
ShowToc: false
TocOpen: false
---

# How Ruby can do what you don't expect

Ruby is a very clever language. It will take the code you've written and do it's best to make sure it runs. Take for example the following method call:

```ruby
method_one('Hello', param_two: ' world')
```

That looks simple enough as it takes two arguments, one positional and one keyword. Now, lets take a look at what the method is actually doing:

```ruby
def method_one(param_one, param_two)
    p param_one, param_two
end
```

Interesting. At first glance it might appear that this method would print `Hello world`, but there is a subtle difference between how the method has been defined and how it is called. Let's see what it actually outputs:

```ruby
"Hello"
{:param_two=>" world"}
```

So what is going on here? Well, it's because when we call `method_one` we are incorrectly assuming that `param_two` is a keyword argument and are providing the name alongside the value. This is where Ruby tries to be clever as it transforms that into the hash `{:param_two=>" world"}`, which is not what we were expecting. There are a few ways you can deal with this, but the one this post is about is adding a double splat nil to the end of the list of arguments.

# Double Splat Nil

`**nil` might look a bit strange if you haven't seen it before, so lets break it down. A double splat (`**`) is used to indicate that you are expecting a quantity of keyword arguments to be provided when a method is invoked. For example:

```ruby
def double_splat(**args)
    p args
end

double_splat(arg_one: 123, arg_two: 'Four Five Six', arg_three: 789)
```

That prints out the following hash:
```ruby
{:arg_one=>123, :arg_two=>"Four Five Six", :arg_three=>789}
```

This has loads of uses, but as we've already seen the conversion into a hash can have unintended consequences. That is where `**nil` comes in. This tells Ruby that we are expected no keyword arguments when this method is called. Let's use this to write a version of `method_one` that doesn't accept keywords:

```ruby
def method_two(param_one, param_two, **nil)
    p param_one, param_two
end
```

Now, what happens if we call it with the unexpected positional argument?

```ruby
method_two('Hello', param_two: ' world')
    => no keywords accepted (ArgumentError)
```

Success! Now, we get an error raised saying something has been called with a keyword when it shouldn't have been which means we don't have to deal with unexpected hashes.