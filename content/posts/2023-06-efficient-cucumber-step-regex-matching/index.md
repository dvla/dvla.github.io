---
author: "Choon Meng Yap"
title: "Efficient Cucumber Step Regex Matching"
description: "How to utilise powerful regex to write better cucumber step definitions"
draft: false
date: 2023-06-07
tags: ["Ruby", "Cucumber", "Regex", "Testing", "Today I Learned"]
categories: ["TIL", "Ruby", "Cucumber", "Regex", "Testing"]
ShowToc: true
TocOpen: true
---

**TL:DR Avoid using Cucumber Expression if possible. Using the conventional regular expression Cucumber is key to more flexible, reusable and customisable test code.**

When writing cucumber step definitions, developers and testers often face difficulties capturing parameters in test steps while ensuring that the steps flow smoothly in natural English sentences.

Without a good knowledge of general regex, step definitions can appear clunky, even when using [Cucumber Expressions](https://github.com/cucumber/cucumber-expressions#readme) to capture step definitions parameters.

## Cucumber Expressions

Cucumber Expressions are more intuitive than [regular expressions](https://en.wikipedia.org/wiki/Regular_expression) as they use a more generic expression to match a given parameter in a text.

Consider the following Cucumber step:

```cucumber
Given a message of 'I like Cucumber' was added to the queue
```

The step can be captured using the following Cucumber Expression in the step definition block:
```ruby
Given('a message of {string} was added to the queue') do |message_text|
    ...
end
```

The `{string}` output parameter will extract the string `message` and pass it as an argument to the step definition.

This may seem fine, and the Cucumber Expression appears intuitive as we can understand what is being captured as a parameter in the text.

If we want to reuse the step definition to perform the same action of adding another message to the queue, we can easily change the step definition to capture alternative text.

```ruby
Given('a/another message of {string} was added to the queue') do |message_text|
    ...
end
```

The slash `/` provides an option for the first word to be either `a` or `another`. Therefore, the updated Cucumber Expression would match the following two steps in a test scenario:

```cucumber
Given a message 'I like Cucumber' was added to the queue
And another message 'I like Ruby' was added to the queue
```

It's important to note that Cucumber does not differentiate between `Given`, `When`, or `Then` in step beginnings; it captures them in the step definitions interchangeably.

Despite the intuitiveness of Cucumber Expressions, we will propose using regular expressions as the more intelligent approach.

## Regular Expressions with Cucumber

Cucumber supports using regular expressions in its step definitions since Cucumber Expressions are built on top of them to help us write step definitions in a more intuitive syntax.

Let's attempt to capture the above examples using the following regex in the step definition block:

```ruby
Given(/^(a|another) message of '(.*)' was added to the queue$/) do |_unused, message_text|
    ...
end
```

The backslashes at the beginning and end of the text indicate a regex. The caret `^` at the beginning and the dollar sign `$` at the end help avoid ambiguous matches (e.g. the regular expression `/there is a message/` matches both `there is a message` and `there is a message in the queue`).

Note that we need the single quotes around the `(.*)` to capture only the string and not the single quotes themselves. Also, there is an unused argument, `_unused` captured from the first capturing group. A capturing group is denoted by the surrounding round brackets in the regex.

Here's a quick summary of some common regex patterns:

|  Patterns   |                             Definition                              |
|:-----------:|:-------------------------------------------------------------------:|
|    `.*`     | matches any character (except for newline) between 0 or more times” |
|    `.+`     | matches at least one of any character (except for line terminators) |
|    `.?`     | matches one or zero of any character (except for line terminators)  |
|     `^`     |                 asserts position at start of a line                 |
|     `$`     |                  asserts position at end of a line                  |
|    `\d`     |                matches a digit (equivalent to [0-9])                |
| `[A-Za-z]*` |   matches letter 'A' to 'Z' or 'a' to 'z' between 0 or more times   |

To simplify this regex better, we can update it to the following:

```ruby
Given(/message of '(.*)' was added to the queue$/) do |message_text|
    ...
end
```

We could have used a non-capturing group by adding a `?:` in the round brackets at the start of the group (e.g. `(?:a|another)`), but since we don't care about preceding words and texts we can drop the group and the caret in the beginning.

As seen in the updated block above, using regex is much simpler and customisable compared to the Cucumber Expression we saw earlier. Our regex will now match any of the following:

```cucumber
Given the message of 'Regex is awesome' was added to the queue
And another message of 'Regex is super awesome' was added to the queue
And a special message of 'This is a special message' was added to the queue
```

## Why use regular expression over Cucumber Expression

### Ambiguous matching

By dropping the caret and unnecessary words at the start of the regex text, we can provide ambiguous matching.

### Stricter parameter capture

Another benefit is that we can specify the format of the text to be captured. For example, we can replace `(.*)` with `([\d-]*)` to capture dates in the format `2023-01-01`. We can even specify the exact number of characters using `{10}` instead of `*` to capture exactly 10 characters in our parameter.

We can be more specific about the format of the date by using the following regex:

```ruby
Given(/date of '([\d]{4}-{\d}{2}-{\d}{2})' was added to the schedule$/) do |date|
    ...
end
```

This will capture the following Cucumber step:

```cucumber
Given a date of '2023-01-01' was added to the schedule
```

Doing this helps reduce errors in our tests and has the added benefit of reducing the need for additional validation on our captured parameters.

### No need for passing parameters in single quotes or double quotes

Using regular expressions eliminates the need to pass parameters in quotes when they are not necessary.

For example, consider the following step:

```cucumber
Given Alice, Bruce and Catherine are in a queue, in no particular order
```

We can capture the text using a regex in the step definition:

```ruby
Given(/^Alice, Bruce and Catherine are in a queue(, in no particular order)?$/) do |option|
  queue = ['Alice', 'Bruce', 'Catherine']
  queue.shuffle! if option.present?
end
```

If instead we had opted for Cucumber expression, the step definition would look like this:

```ruby
Given('Alice, Bruce and Catherine are in a queue{string})'/) do |option|
  queue = ['Alice', 'Bruce', 'Catherine']
  queue.shuffle! if option.present?
end
```

Our step would then require quotes around the parameter to match the Cucumber expression:

```cucumber
Given Alice, Bruce and Catherine are in a queue", in no particular order"
```

In this case, using regex in the step definition provides a smoother flow in the step, compared to using Cucumber Expression.

Although there are cases where quotes are necessary to distinguish data parameters in the step, it is generally considered good practice. For example:

```cucumber
Given I fill in 'Alice' in the first name field

Given the expiry date of the milk is on '2023-01-05'
```

## Don't overuse regex - keep your step definitions simple and distinctive

In cases where we have similar Given steps that we would like to match, it's important to avoid overusing regex to prevent step definitions from becoming too complex and harder to understand.

For example, let's consider the following steps:

```cucumber
Scenario: Checkout apples in basket
    Given there are 5 discounted apples in the basket
    ...
    
Scenario: Checkout medicine in basket
    Given there are 2 cough medicines in the basket
    ...
```

Both steps involve adding items to a basket, but the number of items and the item types are different. The apples are discounted.

We can match both steps with a single step definition using the following regex:

```ruby
Given(/^there are (\d*)( discounted)? (\w*) in the basket$/) do |num_item, option, item_name|
  basket = Basket.new
  basket.inventory = basket.insert_into_basket(item_name:, num_item:, is_discounted: option.present?)
end
```

However, if the requirement states that medicines should never have discounts, it is important to separate the contexts and have two distinct step definitions:

```ruby
Given(/^there are (\d*) (discounted)? (\w*) in the basket$/) do |num_item, option, item_name|
  basket = Basket.new
  basket.inventory = basket.insert_into_basket(item_name:, num_item:, is_discounted: option.present?)
end

Given(/^there are (\d*) (\w*) medicine in the basket$/) do |num_medicine, medicine_name|
  basket = Basket.new
  basket.inventory = basket.insert_into_basket(item_name: medicine_name, num_item: num_medicine) # we assume `is_discounted` has default boolean of false
end
```

This approach helps maintain simplicity in the code and avoids unnecessary complexity. If there are many test scenarios involving medicine in the basket, using separate step definitions will prevent repeated `if` logic and other logics in the code.

## Conclusion

Regular expressions provide several benefits compared to Cucumber Expressions, including flexibility to include options in the steps, stricter parameter capture, and the ability to avoid using quotes unnecessarily in step parameters.

However, it is important to use regular expressions judiciously and avoid overcomplicating step definitions. Keeping step definitions simple and distinctive helps maintain code readability and understandability.
