---
author: "Nigel Brookes-Thomas"
title: "Testing Standard-Out in Ruby"
description: "How to test output written to STDOUT."
draft: false
date: 2023-05-15
tags: ["Ruby", "Testing", "Today I Learned"]
categories: ["TIL", "Ruby", "Testing"]
ShowToc: true
TocOpen: true
---

Testing what a Ruby process writes to STDOUT.

## Ruby's relationship with STDOUT

Ruby has two built-in values that represent the system's standard output stream:

1. The constant `STDOUT`
1. The global variable `$stdout`

The Ruby [documentation](https://docs.ruby-lang.org/en/master/globals_rdoc.html), describes `STDOUT` as _the_ standard output, and `$stdout` and the _current_ standard output. If you want to change the stream that this Ruby process sends its output to, you can change the value of `$stdout`

## Hello StringIO

Given an RSpec test pack, I should be able to use an [output matcher](https://rspec.info/features/3-12/rspec-expectations/built-in-matchers/output/), but I couldn't get this to reliably work across tests for my system. However, the standard library includes an IO-like object for strings, `StringIO`, which must be explicitly required.

This means I can write my tests like this:

```ruby
require 'stringio'

RSpec.describe DVLA::Engineering do
  output = StringIO.new
  $stdout = output
  puts 'Hello from the DVLA'
  expect(output.string).to match(/DVLA/)
end
```