---
author: "Mark Isaac"
title: "FakerMaker Initialisation"
description: "Initialising FakerMaker objects and doing so with greater efficiency."
draft: false
date: 2023-05-17
tags: ["Ruby", "FakerMaker", "Faker", "Testing", "Today I Learned"]
categories: ["TIL", "Ruby", "FakerMaker", "Faker", "Testing"]
ShowToc: true
TocOpen: true
---

In the world of automation testing, generating realistic-looking test data is a preferred approach to using static fixtures.

[FakerMaker](https://billyruffian.github.io/faker_maker/) is a Ruby gem that allows you to do just that; using factories to create the test data you need. 
It can be used with [Faker](https://github.com/faker-ruby/faker) to dynamically generate test data.

## Initialisation

We can use Faker and FakerMaker to create a factory for an API request body like this:
```ruby
require 'faker'
require 'faker_maker'

FakerMaker.factory :example_request, naming: :json do
  name { Faker::Name.name }
  building_number { Faker::Address.building_number }
  street_name { Faker::Address.street_name }
  city { Faker::Address.city }
  country { Faker::Address.country }
end
```

Which can be called to create a request body like this:
```ruby
body = FM[:example_request].build.as_json
=> {"name"=>"The Hon. Earleen Wunsch", 
    "buildingNumber"=>"148", 
    "streetName"=>"Mueller Lodge", 
    "city"=>"Port Darin", 
    "country"=>"Canada"}
```

Often when testing we need to use specific values in the test data, however.

An example: we need to send a API request with "city" set to "London" and "country" set to "United Kingdom".

## Inefficient overwriting

This can be done in 4 steps by:

1. Initialising a template request:
```ruby
body = FM[:example_request].build.as_json
=> {"name"=>"Eleni Daniel", 
    "buildingNumber"=>"50962", 
    "streetName"=>"Jacqulyn Lights", 
    "city"=>"New Rubin", 
    "country"=>"Isle of Man"}
```
2. Overwriting the value of "city" in the request:
```ruby
body["city"] = "London"
```
3. Overwriting the value of "country" in the request:
```ruby
body["country"] = "United Kingom"
```
4. Sending a request containing the overwritten body:
```ruby
body
=> {"name"=>"Eleni Daniel", 
  "buildingNumber"=>"50962", 
  "streetName"=>"Jacqulyn Lights", 
  "city"=>"London", 
  "country"=>"United Kingdom"}
```

Though this works, it can be considered "more expensive" than needed, as today I learned from a colleague a simple trick to do this more efficiently in just 2 steps...

## Efficient initialising

1. Initialising the request with the values we need for the relevant parameter:
```ruby
body = 
  FM[:example_request].build(city: "London", country: "United Kingdom").as_json
```
2. Sending a request containing the efficiently initialised body:
```ruby
body
=> {"name"=>"Msgr. Ernesto Reynolds", 
"buildingNumber"=>"942", 
"streetName"=>"Goyette Passage", 
"city"=>"London", 
"country"=>"United Kingdom"}
```

Job done. Initialising our test data was done more efficiently in a single line of code.

In the context of large automation test packs dealing with complex data, this simple trick can be particularly beneficial to improving efficiency.