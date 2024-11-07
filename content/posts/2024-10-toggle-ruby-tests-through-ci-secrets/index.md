---
author: "Leighton Taylor"
title: "Toggling Ruby scenarios in Drone using secrets"
description: "How to toggle Ruby scenarios to be run in a CI pipeline, from secrets to Cucumber tags"
draft: false
date: 2024-11-07
tags: ["Ruby", "Drone", "CI Pipeline", "Cucumber", "Testing", "Today I Learned"]
categories: ["TIL", "Ruby", "Drone", "Testing", "Cucumber"]
ShowToc: true
TocOpen: true
---


# *TiL* **how to toggle Ruby scenarios to be run using CI pipeline secrets**

---
## Scenario

In a relatively niche scenario, we have functionality that is being temporarily removed while new code is being developed.

While we still want to keep our current scenarios for function X to use them again when it's reintroduced, if they are left to run in our pipeline, every build will fail until it is reintroduced again.

---
### How do we get around this?

Well to start off with, because we're good testers, we need to write new scenarios that would check to see that the expected errors will be returned once the code is removed, providing us with confidence that it'll be working correctly.

These unhappy path tests should only run when the code is removed otherwise, they'll fail too.

### The Config

We can use CI pipeline secrets to our advantage, in our pipeline config file we'll add an environment variable in our functional test step that we can overwrite when the functionality is removed.

```yaml
RUN_FUNCTION_X:
    from_secret: RUN_FUNCTION_X # Either ':yes' or ':no' -- set as secret 
```

Best practice would be to set this variable with a default value.
That can be done within our config settings.

```yaml
RUN_FUNCTION_X: <%= ENV['RUN_FUNCTION_X'] || :yes >
```

And then in our env.rb, we will set a boolean value to `RUN_FUNCTION_X` by checking if the drone secret has been added.


```ruby
# Switch the drone secret from :yes to :no to toggle test functionality
RUN_FUNCTION_X = SimpleSymbolize.symbolize(Settings.RUN_FUNCTION_X) == :yes
Logger.info{ "Run function X scenarios set to: #{RUN_FUNCTION_X}" }
```

### The Hooks

Now, this is where our `before` hooks come into play.

Using conditionals, `if` and `unless`, we can use our hooks to skip one set of tests and run the others depending on whether our `RUN_FUNCTION_X` variable is true or false. [^1]

[^1]: We took the approach of using 2 hooks for better clarity and readability, however, the same result can be achieved using 1 before hook by tagging both sets of tests with `@FUNCTION_X` and using a conditional statement like `unless RUN_FUNCTION_X || scenario.source_tag_names.include?('@UNHAPPY_PATH_FUNCTION_X')`.

```ruby
Before('@UNHAPPY_PATH_FUNCTION_X') do
  if RUN_FUNCTION_X
    Logger.info { 'Skip function X scenarios are set to be ignored. Skipping scenario.' }
    raise Core::Test::Result::Skipped, 'Scenario Skipped' 
  end
end

Before('@FUNCTION_X') do
  unless RUN_FUNCTION_X
    Logger.info { 'Function X scenarios are set to be ignored. Skipping scenario.' }
    raise Core::Test::Result::Skipped, 'Scenario Skipped' 
  end
end
```


### The Tests

We need to make it clear which test scenarios are going to be skipped, and which tests will be our unhappy path tests, so we add our custom Cucumber tags to our feature files.

```gherkin
@FUNCTION_X
Scenario: AC3.1 Customer successfully applies for function x
Given a customer logs into the system
When the customer applies for function x
Then the system returns a 200 code

@UNHAPPY_PATH_FUNCTION_X # ToDO Remove scenario when function X is reintroduced
Scenario: AC1 Function X is removed, returning an error
Given a customer logs into the system
When the customer applies for function x
Then the system produces the following error:
| status | code | detail           |
| 400    | 123  | Function X error | 
````

### And Finally

{{<figure src="images/Drone secrets banner screenshot.png" caption="Drone Repository Settings Banner">}}

With all this in place, as soon as we require function X scenarios to stop running and our unhappy path tests to start, all we need to do is add a secret to our CI pipeline called `RUN_FUNCTION_X` and set it to `:no`.

{{<figure src="images/Drone secrets pop up box.png" caption="Drone Secret Pop Up Box" >}}

Then all you have to do is delete the secret once the function is reintroduced, the removal of the config, unhappy path tests and tags afterwards is up to your discretion depending on whether that functionality is to be temporarily removed again in the future or not.


## TL;DR

If a function is temporarily removed, we can skip our tests for that functionality and run tests to ensure that the unavailable function returns the correct error.

Setting CI pipeline secret that's linked to an environment variable in our CI pipeline config file and in a config file, best set to a default of truthy or true value, which is set to a boolean value by the env.rb before each run. 

Using that boolean, we decide whether to skip the function X scenarios and run our unhappy path tests using before hooks linked to Cucumber tags.


---