---
author: "Nigel Brookes-Thomas & George Bell"
title: "Artefacts in tests and managing them with Atlas"
description: "A look at our concept of artefacts in functional test packs and how we manage them using Atlas."
draft: false
date: 2023-11-14
tags: ["Ruby", "Testing", "Cucumber", "Gem"]
categories: ["Ruby", "Gem"]
ShowToc: true
TocOpen: true
---

We've recently publicly released a Gem called [dvla-atlas](https://github.com/dvla/atlas)[^1] and today we are going to take you through a bit of history surrounding testing at the DVLA that led to use developing Atlas, along with a dive into some of the code that makes it tick. Atlas is designed to make the managing of properties in functional tests easier while also ensuring that each test is run in isolation and without any cross-pollination of test data. But first, lets take a look at what we mean by artefacts and we used them in tests we'd written in [Cucumber](https://cucumber.io/).

[^1]: As you'll see, we've named it that because it supports the `World`

## Artefacts



## Atlas

With Atlas, we hope to formalise what we were doing with artefacts without distrupting the way we like to write tests. Our goal was to leverage some functionality built into Cucumber called `World`[^2] while also having something nice and easy to pick up and use.

[^2]: If you haven't come across `World` in Cucumber before, it's a way of influencing the context within with a test scenario is run. You don't need to worry about the specifics for this post, as we'll explain them as they become relevant, but if you are interested in the inner workings you can find some tests that document the common uses for it in [the Ruby implementation of Cucumber](https://github.com/cucumber/cucumber-ruby/blob/main/features/docs/writing_support_code/world.feature)

### Defining properties

Within a test pack, it is not uncommon for multiple scenarios to require property with a common name, such as `email_address`. While with tools like [faker-maker](https://github.com/BillyRuffian/faker_maker) it's nice and simple to make sure each scenario has a different, but realistic-looking, value. We want to make sure that each step that references that property is consistent in what it is called. Having half your tests refer to `email_address` and the other half to `emailAddress` is a fantastic way to introduce confusion where it isn't needed. That's where `World` comes in handy.

`World` allows you to define the context within which test steps are executed. You can provide it with a block of code, traditionally in your `env.rb` when setting up the tests, that will be executed prior to each test, with the value that block returns being the context in which the tests are run. For example, suppose we write the following in our environment file:

```ruby
class Example
    attr_reader :value

    def initialize(base_value)
        @value = base_value
    end

    def add_to_value(value_to_add)
        @value += value_to_add
    end
end

World do
    Example.new(2)
end
```

With the above set up, you'd be able to access `value` and call `add_to_value` in any test step, with that block being called again at the beginning of each new scenario to generate the context that it will be run in. Therefore, each test will start with a `value` of 2. It's on top of this Cucumber functionality that we built Atlas:

```ruby
World do
    world = DVLA::Atlas.base_world
    world.artefacts.define_fields('email_address')
    world
end
```

`DVLA::Atlas.base_world` returns an object that contains an empty set of artefacts. Upon that we can call `define_fields`, which will allow us to create as many properties as we need. These are then made accessible to all test steps, as they are now part of the context they are being run in, ensuring that we have consistent naming throughout our test pack with a singular source of truth for what those names are.

### Default values

Once you've got property creation on the artefacts up and running, the next obvious step is default values. There are times where might want a property to start out a specific value. In that case, you can pass in that value as the keyword against the property name, as seen below:

```ruby
World do
    world = DVLA::Atlas.base_world
    world.artefacts.define_fields(url: 'www.example.com')
    world
end
```

That gives all test steps access to the property `url`, which they can alter as much as they like to do whatever it is they need to do, with each new scenario getting a newly initalized world that is back to the default value

### Scoping

A number of our tests rely on code being able to access artefacts from not only within the scope of the test steps but within various other classes and modules. There are times where it makes sense to pass these values around through the various method calls, but likewise there are times were that proved to not be practical. Thus, by adding `DVLA::Atlas.make_artefacts_global(world.artefacts)` into the `World` block once all the fields on artefacts have been set up, we can make them globally accessible. This works within Atlas by using a bit of metaprogramming to define a getter on `Object` that points to the artefacts, as seen here:

```ruby
def self.make_artefacts_global(artefacts)
    DVLA::Atlas::Holder.instance.artefacts = artefacts
    Object.send(:define_method, :artefacts) { DVLA::Atlas::Holder.instance.artefacts }
end
```

### Trackable history

Finally, this is something we've had a few people had implemented independently and Atlas felt like the perfect place to formalise this functionality. There are times where a test might want to validate a sequence, such as the order pages have been visited in a journey through a website. Historically, people were creating various data structures to store these details, however in Atlas all properties that are initialised also come with a history field that stores an array of all previous values that that property has held. Take the following example: 

```ruby
World do
  world = DVLA::Atlas.base_world
  world.artefacts.define_fields(journey_status: :started)
  world
end

...

Given 'the customer hits the submit button' do
    artefacts.journey_status = fetch_journey_status # Should now be submitted 
end

...

When 'the query has been dealt with' do
    artefacts.journey_status = fetch_journey_status # Should now be completed
end

Then 'the journey has hit all the correct statuses' do
    expect(artefacts.journey_status).to eq(:completed)
    expect(artefacts.journey_status_history).to eq([:started, :submitted])
end
```

In the above example, we can not only assert that the `journey_status` ends up at `completed` but that it took the right path to get there. If a code change went in that accidentally pushed the journey into `cancelled` instead of `submitted`, we'd be able to catch that even if the system still ended up in the correct place. Again, through the use of metaprogramming, we were able to implement this in quite a lightweight fashion, as seen in the below code extract from Atlas:

```ruby
instance_variable_set("@#{name}_history", [])

define_singleton_method :"#{name}_history" do
    instance_variable_get("@#{name}_history")
end

define_singleton_method :"#{name}=" do |arg|
    current_value = send(:"#{name}")
    send(:"#{name}_history").push(current_value) unless send(:"#{name}_history").empty? && current_value.nil?
    instance_variable_set("@#{name}", arg)
end
```

As you can see, we create the history as an empty array and define a getter for it. Then, as part of the definition of the setter for the actual property, we have line that pushes the value that is being overwritten into the history. This means that the setter can't be called without also updating the relevant history and ensures we are can access whatever previous values we require with no additional setup required when actually writing the tests.