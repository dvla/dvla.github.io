---
author: "George Bell"
title: "TiL: The difference between exec and system in Ruby"
description: "What exactly are exec and system doing and why do they differ?"
draft: false
date: 2024-09-23
tags: ["Ruby", "exec", 'system', "Today I Learned"]
categories: ["TIL", "Ruby"]
ShowToc: true
TocOpen: true
---

Ruby provides a few different ways to execute commands against the underlying kernel programmatically, but they all work slightly differently. The simplest way to execute a command againts the shell is to surround it in backticks. For example, the following will run the command `date` and return whatever was output to `$stdout`.

```ruby
current_date = `date`
```

However, there are also more explict methods you can call. In particular, there is `exec` and `system` which look like they do the same thing at a quick glance. However, they actually work very differently once you get down into what they are doing.

# Exec

`exec` works by replacing the process it is called within with the one that is passed into it. This can either be a command to be executed or the path to an executable, but either way the current process will be replaced with what is passed in. This means that when the newly created process exits, it won't have a ruby application to return to. Take the following code:

```ruby
value = 123
puts 'Getting the date'
exec 'date'
puts value * 2
```

Because `exec` is called, the final line will not be called and 246 will not be printed out. The last thing to be output will be the current date, as seen below:

```shell
Getting the date
Mon Sep 23 10:22:14 BST 2024
```

# System

The basic interface with `system` is very similar. When provided with either a command as a string or the path to an executable, it will be started in a process. The difference is that, rather than replacing the current process, it will instead create a child process. When this process exits it will return back to the ruby process and continue executing, returning `true` if the command exits with status 0 and `false` if it exits with a non-zero integer. Let us return to the previous example:

```ruby
value = 123
puts 'Getting the date'
system 'date'
puts value * 2
```
Now that we using system, the final line will execute and our output will look like the following:

```shell
Getting the date
Mon Sep 23 10:22:19 BST 2024
246
```

Much more useful if you need to quickly run a command as part of your code and don't wish to fully cede control to a different program.

# A word of warning

As with anything like this, it is always worth keeping in mind the risk of executing a command aganst the shell directly. No matter which way you are doing it, never execute anything you aren't confident you understand everything it can do.