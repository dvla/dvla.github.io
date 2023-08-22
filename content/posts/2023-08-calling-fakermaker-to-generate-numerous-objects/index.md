---
author: "Tomos Griffiths"
title: "TiL: Generating numerous FakerMaker objects in one call"
description: "Creating numerous FakerMaker objects and storing in an Array, in one call."
draft: false
date: 2023-08-22
tags: ["Ruby", "FakerMaker", "Faker", "Testing", "Today I Learned"]
categories: ["TIL", "Ruby", "FakerMaker", "Faker", "Testing"]
ShowToc: true
TocOpen: true
---

Trying to generate numerous, realistic-looking objects for automation testing can be difficult.

Using [FakerMaker](https://billyruffian.github.io/faker_maker/) to do just that is preferable; using factories with [Faker](https://github.com/faker-ruby/faker) to dynamically create the test data you need in whatever format you need, is far better than using fixtures. 

But what if you needed more than one object of the particular factory that you have created?

## Creating a factory

We can use Faker and FakerMaker to create an individual factory for an API request body like this:
```ruby
require 'faker'
require 'faker_maker'

FakerMaker.factory :grocery_order do
  id(json: 'id', omit: :nil) { Faker::Number.number(digits: 14).to_s }
  timestamp(json: 'timestamp', omit: :nil) { DateTime.now }
  frozen_food(json: 'frozenFood', omit: :nil) { FakerMaker[:frozenFood].build }
  cupboard_items(json: 'cupboardItems', omit: :nil) { FakerMaker[:cupboardItems].build }
  fridge_items(json: 'fridgeItems', omit: :nil) { FakerMaker[:fridgeItems].build }
  snack_items(json: 'snackItems', omit: :nil) { FakerMaker[:snackItems].build }
  drink_items(json: 'drinkItems', omit: :nil) { FakerMaker[:drinkItems].build }
  alcohol_items(json: 'alcoholItems', omit: :nil) { FakerMaker[:alcoholItems].build }
  grocery_total(json: 'groceryTotal', omit: :nil) {Faker::Number.number(digits: 6).to_s }
end
```
But what if the test requires MORE THAN ONE instance of this fairly complex object be generated?


## Creating multiple instances of the FM factory object

This can be done in 2 ways:

1. Using Array.new:
```ruby
multiple_records = Array.new(3) { FM[:grocery_order].build.as_json }
```
This will create a new array of length 3 AND build 3 instances of the :cpc_record object. The multiple_records array can then be returned for your tests.

2. Using .map:
```ruby
multiple_records = 3.times.map { FM[:grocery_order].build.as_json }
```
This will do the exact same thing as the above. This is a more elegant way of writing the above code, and is more readable, in my opinion.


And that is it! 

## Conclusion

For complex objects, such as the example above, it can be beneficial to reproduce numerous objects to be stored in test artefacts to ensure your application or system can handle them within the response. This is especially useful when there can be numerous instances of certain objects held.
