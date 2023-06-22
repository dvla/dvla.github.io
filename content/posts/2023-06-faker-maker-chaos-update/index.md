---
author: "Alexander O'Mahoney"
title: "FakerMaker Chaos Mode Update"
description: "Introducing the Chaos mode update for FakerMaker"
draft: false
date: 2023-06-16
tags: ["Ruby", "Chaos", "FakerMaker", "Testing", "Today I Learned", "Gem"]
categories: ["TIL", "Ruby", "FakerMaker", "Testing", "Gems"]
ShowToc: true
TocOpen: true
---

**TL:DR Level up your testing by introducing some chaos to your test data.**

FakerMaker is an incredible factory builder used to generate test data. 

When introduced to a test-pack (alongside Faker), it elevates tests by introducing dynamic test data to ensure
we're not constantly testing with the same old boring static data.

```ruby
FakerMaker.factory :some_fancy_object, naming: :json do
  some_required_attribute { Faker::Lorem.sentence }
  some_optional_attribute_1 { Faker::Lorem.word }
  some_optional_attribute_2 { Faker::Lorem.word }
  some_optional_attribute_3 { Faker::Lorem.word }
  some_optional_attribute_4 { Faker::Lorem.word }
end

FakerMaker[:some_fancy_object].build.as_json 
# => {"someRequiredAttribute"=>"Nulla omnis dolore enim.", "someOptionalAttribute1"=>"qui", "someOptionalAttribute2"=>"deserunt", "someOptionalAttribute3"=>"rerum", "someOptionalAttribute4"=>"facilis"}
FakerMaker[:some_fancy_object].build.as_json 
# => {"someRequiredAttribute"=>"Aut repellendus ut quod.", "someOptionalAttribute1"=>"consequatur", "someOptionalAttribute2"=>"quod", "someOptionalAttribute3"=>"pariatur", "someOptionalAttribute4"=>"rerum"}
```

This is a super effective way of ensuring data within an attribute is dynamic, but can we take this further? 

## Chaos Engineering

Chaos engineering is a concept that has been around since the early 2000s. The idea is simple - can we build resilient, 
fault-tolerant services and how do we prove this? 

How does a service cope when it loses connection to a database?
What about a service that has gone down that our service depends on? 

To prove a system is resilient against these types of issues we might experiment and replicate these issues to monitor
the impact. Remove the database connection, terminate a dependent service etc. 

In general, cause chaos. So let's take that concept, and apply it to something like test data! 
 
## Introducing Chaos for FakerMaker

FakerMaker v2 introduces chaos mode.

Let's explore the `some_fancy_object` example from earlier. This time with chaos enabled.

```ruby
FakerMaker.factory :some_fancy_object, naming: :json do
  some_required_attribute_1(required: true) { Faker::Lorem.sentence }
  some_required_attribute_2(optional: false) { Faker::Lorem.word }
  some_optional_attribute_1 { Faker::Lorem.word }
  some_optional_attribute_2(optional: 90) { Faker::Lorem.word }
  some_optional_attribute_3(optional: 10) { Faker::Lorem.word }
end

FakerMaker[:some_fancy_object].build(chaos: true).as_json 
# => {"someRequiredAttribute1"=>"Sed rem magnam placeat.", "someRequiredAttribute2"=>"nemo", "someOptionalAttribute2"=>"dolores"}
FakerMaker[:some_fancy_object].build(chaos: true).as_json 
# => {"someRequiredAttribute1"=>"Hic neque aut sapiente.", "someRequiredAttribute2"=>"saepe", "someOptionalAttribute1"=>"id"}
FakerMaker[:some_fancy_object].build(chaos: true).as_json 
# => {"someRequiredAttribute1"=>"Fuga est rerum quia.", "someRequiredAttribute2"=>"vero", "someOptionalAttribute2"=>"nulla", "someOptionalAttribute1"=>"dolorem"}
```

When a factory is built with chaos enabled, optional fields become truly optional and may not be present in the built factory. 

Not only is the data dynamic per attribute, but the attributes themselves are now dynamic ... chaos!

Lets dig in a bit ...

### Attribute metadata

Attributes can now be tagged as `required` or `optional`.

```ruby
some_required_attribute_1(required: true) { Faker::Lorem.sentence }
some_optional_attribute_1(optional: true) { Faker::Lorem.sentence }
```
> By default, all attributes are `optional`.

You can also pass through an optional weighting. This determines the likelihood of that attribute appearing in your built factory

```ruby
some_optional_attribute_1(optional: 90) { Faker::Lorem.sentence } # 90% chance this attribute will be present
some_optional_attribute_2(optional: 10) { Faker::Lorem.sentence } # 10% chance this attribute will be present
```
> By default, all `optional` attributes are weighted at `50%`.

Metadata on attributes provides some benefits without chaos enabled. 

Factories are often modelling schemas listed in API contracts. These contracts state which fields are required. 

By having this data available on the factory it gives us at a glance information at the data we are modeling without 
the hassle of checking API contracts.

### Enabling Chaos

Chaos is enabled when a factory object is built by passing the `chaos` flag.

```ruby
FakerMaker[:some_fancy_object].build(chaos: true)
```

You can also specify which fields to run chaos over by passing an `Array`.

```ruby
FakerMaker[:some_fancy_object].build(chaos: %i[some_optional_attribute_1])
```
> Chaos will only run on `some_optional_attribute_1` and not against the other optional attributes.


### Why would I want this?

By using this approach to testing, we are more closely mimicking real life scenarios. 

Data comes in all shapes and sizes, especially when optional attributes come into play. 

How do you know an APIs optional field is truly optional? How do you know your beautiful webpage that displays data with optional fields will actually work?

Chaos introduces new combinations of tests without the hassle or mess of doing it yourself.

### TL:DR
- Every attribute is `optional` by default
- Optional attributes maybe removed when your factory is built and chaos is enabled
- Attributes can be marked as `required` - required attributes will always be present
- You can add weighting to individual optional attributes to determine the likelihood of them being present
- You can run chaos over a subset of optional attributes when building your factory

---
