---
author: "Thomas Feathers"
title: "The at_exit function"
description: "Using the at_exit function to run functions when a program exits"
draft: false
date: 2023-08-23
tags: ["Ruby", "Kernel", "Testing", "Today I Learned"]
categories: ["TIL", "Ruby", "Kernel", "Testing"]
ShowToc: true
TocOpen: true
---

Traditionally, in functional testing, a clean up/tear down script would be run in something called an `After` hook. These hooks are part of the Cucumber DSL and are designed to execute when a scenario has finished.

Cucumber example:
```ruby
After do |scenario|
  if scenario.failed?
    MultiLogger.clean_logs
  end
end
```

There is a drawback to these hooks where they will fail to run when certain exit codes are returned from the scenario or the program is interrupted.  This isn't great when you need these scripts to execute on every run, regardless of the exit reason.

This is where `at_exit` comes in.  

## The at_exit function

The `at_exit` function is defined in the `Kernel` module within the Ruby class Object. This makes their methods available in every Ruby Object. This is ideal as we have no complex dependencies
to consider when using it.

`at_exit` works when a program exits. It guarantees to run at exit regardless of the reason; be it a successful program execution, an uncaught exception or someone manually killing the program (ctrl + c).

It works by converting a given block to a [Proc](https://ruby-doc.org/core-3.0.0/Proc.html) and registers it for execution when the program exits.

## Example:

Here's how this function is used in the TYR project to clear our Mailsac account of all temporary inboxes created within the tests.

```ruby
at_exit do
  list_inboxes.map { |i| i['_id'] }
              .grep(/tyr-e2e-*/)
              .then { |i| i.each { |inbox| delete_inbox(inbox: inbox) }}
end
```

## Conclusion

For a reliable solution to execute a function/script at program exit in Ruby, the `at_exit` function is the most reliable to use.
