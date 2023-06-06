---
author: "Choon Meng Yap"
title: "Efficient Cucumber Step Regex Matching"
description: "How to utilise powerful regex to write better cucumber step definitions"
draft: false
date: 2023-06-05
tags: ["Ruby", "Cucumber", "Regex", "Testing", "Today I Learned"]
categories: ["TIL", "Ruby", "Cucumber", "Regex", "Testing"]
ShowToc: true
TocOpen: true
---

**TL:DR Avoid using Cucumber Expression if possible. Using the conventional regular expression Cucumber is key to more flexible, reusable and customisable test code.**

When writing cucumber step definitions, developers and testers often face some difficulty of capturing parameters in the test steps while making sure the steps are flowing smoothly in natural English sentences.

Without good knowledge of general regex, step definitions can look clunky despite using [Cucumber Expressions](https://github.com/cucumber/cucumber-expressions#readme) to capture step definitions parameters.

## Cucumber Expressions

Cucumber Expressions are more intuitive than [regular expressions](https://en.wikipedia.org/wiki/Regular_expression) in that it uses a more generic expression to match a given parameter in a text. 

Consider the following Cucumber step:

```cucumber
Given a message of 'I like Cucumber' was added to the queue
```

The step can be captured using the following Cucumber Expression in the step definition block:
```cucumber
Given('a message of {string} was added to the queue') do |message_text|
    ...
end
```

The `{string}` output parameter will extract the string `message` and pass it as an argument to the step definition.

This might be fine and dandy, as the Cucumber Expression looks intuitive and we can understand what is being captured as parameter in the text.

If we want to reuse the step definition to do the same thing again, for example adding another message to the queue in this case, we can easily change the step definition to capture alternative text.

```cucumber
Given('a/another message of {string} was added to the queue') do |message_text|
    ...
end
```

The backlash provides an option for the first word to be `a` or `another`. Thus, the updated Cucumber Expression would match the following two steps in a test scenario:

```cucumber
Given a message 'I like Cucumber' was added to the queue
And another message 'I like Ruby' was added to the queue
```

Note that Cucumber does not care if the step begins with `Given`, `When` or `Then`, as it will capture them in the step definitions as the keywords are interchangeable.

Despite the intuitiveness of Cucumber Expression, we are going to propose using ol' elementary regex as the more customisable approach.

## Regular Expressions with Cucumber

Cucumber supports using regular expression in its step definition, and Cucumber Expression is built on top of it to help us write our step definition in more intuitive style. Let's attempt to capture the above examples using the following regex in the step definition block:

```cucumber
Given(/^(a|another) message of '(.*)' was added to the queue$/) do |_unused, message_text|
    ...
end
```

The backslashes in the beginning and end of the text are necessary to indicate a regex. The caret `^` in the beginning and dollar sign `$` at the end helps to avoid ambiguous matches (e.g. the regular expression `/there is a message/` matches both texts of `there is a message` and `there is a message in the queue`).

Note that we need the single quotes around the `(.*)` in order to capture only the string and not the single quotes itself. We also note that there is an unused argument, `_unused` captured from the first capturing group. A capturing group is denoted by the surrounding round brackets in the regex.

A quick summary of some common patterns in regex are shown below:

|  Patterns   |                             Definition                              |
|:-----------:|:-------------------------------------------------------------------:|
|    `.*`     | matches any character (except for newline) between 0 or more times” |
|    `.+`     | matches at least one of any character (except for line terminators) |
|    `.?`     | matches one or zero of any character (except for line terminators)  |
|     `^`     |                 asserts position at start of a line                 |
|     `$`     |                  asserts position at end of a line                  |
|    `\d`     |                matches a digit (equivalent to [0-9])                |
| `[A-Za-z]*` |   matches letter 'A' to 'Z' or 'a' to 'z' between 0 or more times   |

To make this regex better, we can simplify it to the following:

```cucumber
Given(/message of '(.*)' was added to the queue$/) do |message_text|
    ...
end
```

We could have used a non-capturing group by adding a `?:` in the round brackets at the start of the group (e.g. `(?:a|another)`), but since we don't care about preceding words and texts we can drop the group and the caret in the beginning.

The result as seen in the block above is it is much simpler and customisable compared to the Cucumber Expression we've seen before. Our regex can match now any of the following:

```cucumber
Given the message of 'Regex is awesome' was added to the queue
And another message of 'Regex is super awesome' was added to the queue
And a special message of 'This is a special message' was added to the queue
```

## Why use regular expression over Cucumber Expression

### Ambiguous matching

The example seen above shows that we are able to provide ambiguous matching by dropping the caret and unnecessary words at the start of the regex text.

### Stricter parameter capture

Another benefit which can be clearly seen is when we want to only allow certain text of a specific format to be captured. For example we can replace `(.*)` with `([\d-]*)` to captures dates in the format `2023-01-01`. We can even go further by replacing `*` with a `{10}` to indicate that we specifically want 10 characters in our captured parameter.

We can be more specific about the format of the date, giving an regex in this example:

```cucumber
Given(/date of '([\d]{4}-{\d}{2}-{\d}{2})' was added to the schedule$/) do |date|
    ...
end
```

which will capture the following Cucumber step:

```cucumber
Given a date of '2023-01-01' was added to the schedule
```

Doing this sometimes help reduce errors in our test and potentially reduce validation to be performed on our captured parameters.

### No need for passing parameters in single quotes or double quotes

Another benefit of using regex over Cucumber expression is we can avoid passing parameters in single quotes or double quotes within our test steps when it is not needed.

To demonstrate, take a look in an example step such as this:

```cucumber
Given Alice, Bruce and Catherine are in a queue, in no particular order
```

We can capture the text using regex in the step definition:

```cucumber
Given(/^Alice, Bruce and Catherine are in a queue(, in no particular order)?$/) do |option|
  queue = ['Alice', 'Bruce', 'Catherine']
  queue.shuffle! if option.present?
end
```

If instead we have opted for Cucumber expression, we would have the following step definition:

```cucumber
Given('Alice, Bruce and Catherine are in a queue{string})'/) do |option|
  queue = ['Alice', 'Bruce', 'Catherine']
  queue.shuffle! if option.present?
end
```

And our step would need to have quotes around the parameter for the Cucumber expression to match:

```cucumber
Given Alice, Bruce and Catherine are in a queue", in no particular order"
```

Sometimes we still need the quotes to distinguish a data parameter in the step, which is considered good practice. A few examples can be demonstrated in the following steps: 

```cucumber
Given I fill in 'Alice' in the first name field

Given the expiry date of the milk is on '2023-01-05'
```

## Don't overuse regex, keep your step definitions simple and distinctive

Provided we have the following `Given` steps we want to match:

```cucumber
Scenario: Checkout apples in basket
    Given there are 5 discounted apples in the basket
    ...
    
Scenario: Checkout medicine in basket
    Given there are 2 cough medicines in the basket
    ...
```

Both steps add some items into a basket, but the number of items and item types are different and the apples are discounted.

We can match both steps with a single step definition, perhaps using the following:

```cucumber
Given(/^there are (\d*)( discounted)? (\w*) in the basket$/) do |num_item, option, item_name|
  basket = Basket.new
  basket.inventory = basket.insert_into_basket(item_name:, num_item:, is_discounted: option.present?)
end
```

Let's say the requirement states that medicines should never have discounts, so we need to make sure there are in two distinct steps definitions in order to separate the contexts:

```cucumber
Given(/^there are (\d*) (discounted)? (\w*) in the basket$/) do |num_item, option, item_name|
  basket = Basket.new
  basket.inventory = basket.insert_into_basket(item_name:, num_item:, is_discounted: option.present?)
end

Given(/^there are (\d*) (\w*) medicine in the basket$/) do |num_medicine, medicine_name|
  basket = Basket.new
  basket.inventory = basket.insert_into_basket(item_name: medicine_name, num_item: num_medicine) # we assume is_discounted has default boolean of false
end
```

Doing this also helps avoid unnecessary complexity in the code. If we have a lot of test scenarios that involves medicine in the basket we may end up with a lot of repeated logic in code that adds to the compilation time (although in this case `option.present?` is not a very expensive operation to perform every time)

## Conclusion

We can see that regular expression provides a lot of benefits compared to Cucumber Expression, including the flexibility to include options in the steps and provide ambiguous matching while making parameters capture to be stricter.

Care should be taken to make sure we don't overuse regex as we may cause the step definition to do too much and make it harder to understand.