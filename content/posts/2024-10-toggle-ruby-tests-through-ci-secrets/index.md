---
author: "Leighton Taylor"
title: "Toggling Ruby scenarios in Drone using secrets"
description: "How to toggle Ruby scenarios to be run in a CI pipeline, from secrets to Cucumber tags"
draft: false
date: 2024-10-31
tags: ["Ruby", "Drone", "CI Pipeline", "Cucumber", "Testing", "Today I Learned"]
categories: ["TIL", "Ruby", "Drone", "Testing", "Cucumber"]
ShowToc: true
TocOpen: true
---


# *TiL* **how to toggle Ruby scenarios to be run using CI pipeline secrets**

---
## Scenario

In a relatively niché scenario, we have a service that is being temporarily being blocked while new code is being put into place.

While we still want to keep our current scenarios for service X to use them again when the serive is unblocked, if they are left to run in our pipeline, every build will fail until it is unblocked again.

---
### How do we get around this?

Well to start off with, because we're good testers, we need to write new scenarios that would check to see that the expected errors will be returned once the block put in, providing us with confidence that it'll be working correctly.

These sad path tests should only run when the block is in place otherwise they'll only fail too.

### The Config

We can use CI pipeline secrets to our advantage, in our pipeline config file we'll add a environment variable in our functional test step that we can overwrite when our block goes into place.

```yaml
RUN_SERVICE_X:
    from_secret: RUN_SERVICE_X # Either ':yes' or ':no' -- set as secret 
```

Best practice would be to set this variable with a default value.
That can be done within our config settings.

```yml
RUN_SERVICE_X: <%= ENV['RUN_SERVICE_X'] || :yes >
```

And then in our env.rb, we will set a boolean value to `RUN_SERVICE_X` by checking if the drone secret has been added.


```ruby
# switch the drone secret from :yes to :no to toggle test functionality
RUN_SERVICE_X = SimpleSymbolize.symbolize(Settings.RUN_SERVICE_X) == :yes
Logger.info{ "Run service X scenarios set to: #{RUN_SERVICE_X}" }
```

### The Hooks

Now, this is where our `before` hooks come into play.

Using conditionals, `if` and `unless`, we can use our hooks to skip one set of tests or the other depending on whether our `RUN_SERVICE_X` variable is true or false.

```ruby
Before('@BLOCK_SERVICE_X') do
  if RUN_SERVICE_X
    Logger.info { 'Block service X scenarios are set to be ignored. Skipping scenario.' }
    raise Core::Test::Result::Skipped, 'Scenario Skipped' 
  end
end

Before('@SERVICE_X') do
  unless RUN_SERVICE_X
    Logger.info { 'Service X scenarios are set to be ignored. Skipping scenario.' }
    raise Core::Test::Result::Skipped, 'Scenario Skipped' 
  end
end
```

As you can see, we've used 2 different tags for the blocked service scenario and our blocked service sad path test.

### The Tests

We need to make it clear which test scenarios are going to be blocked and which tests will be our blocked sad path tests so we add our custom Cucumber tags to our feature files.

```gherkin
@SERVICE_X
Scenario: AC3.1 Customer successfully applies for service x
Given a customer logs into the system
When the customer applies for service x
Then the system returns a 200 code

@BLOCK_SERVICE_X # ToDO remove scenario when service X is unblocked
Scenario: AC1 Service X is blocked returning error
Given a customer logs into the system
When the customer applies for service x
Then the system produces the following error:
| status | code | detail           |
| 400    | 123  | Service X errror | 
````

### And Finally

{{<figure src="images/Drone secrets banner screenshot.png" caption="Drone Repository Settings Banner">}}

With all this in place, as soon as we require the blocked scenarios to stop running and our sad path tests to start, all we need to do is add a secret to our CI pipeline called `RUN_SERVICE_X` and set it to `:no`.

{{<figure src="images/Drone secrets pop up box.png" caption="Drone Secret Pop Up Box" >}}

Then all you have to do is delete the secret once the service is unblocked, the removal of the config and tags afterwards is up to your discretion depending on whether that service is to be temporarily blocked again in the future or not.


## TL;DR

If a service is temporarily blocked, we can skip our tests for that service and run tests to ensure the block is returning the correct error/s.

Setting CI pipeline secret that's linked to a environment variable in our CI pipeline config file and in a config file, best set to a default of truthy or true value, which is set to a boolean value by the env.rb before each run. 

Using that boolean, we decide whether to skip the blocked service scenarios and run our sad path tests using before hooks linked to Cucumber tags for blocked scenarios and sad path scenarios.


---