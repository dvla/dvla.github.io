---
author: "George Bell"
title: "Why we wrote Herodotus"
description: "A look at the gem Herodotus and the problems we have been using it to solve."
draft: false
date: 2023-05-17
tags: ["ruby", "logging"]
categories: ["ruby"]
ShowToc: true
TocOpen: true
---

We've written Herodotus[^1] as a lightweight logger to solve some very specific problems we had in our tests, but we don't think they are unique to us so we're going to share with you what we have done and why.

[^1]: Herodotus is named after [a greek scholar and historian](https://en.wikipedia.org/wiki/Herodotus). One of the most famous works that Herodotus produced was the "Histories" which are some very important lodsfgs

## Logging to the standard output and the main problem we faced

The problem we are trying to solve is one of readability. All good logs are human readable, as when something has gone wrong to the point of a human needing to look in the logs and piece together what has happened we don't want to make that more difficult than is needed. Now, there are things out there that will help you manage your logs (for example, OpenSearch provides a fantastic toolkit for exploring a large volume of logs), these are often looking at things on a grander scale than we are. There are always going to be times where you want your system to just leave a little message wherever it is currently running, which is why we are looking at things that are logging to the standard output on whatever device they are running. Specifically in our case, we are looking at the output of an automated test pack.

Take the following code snippet, where we set up a default ruby logger and seed a database (that has access to the same global logger) for three different test scenarios, along with the output when we run it:

```ruby
module DatabaseAccess
  def self.seed_database(test_data)
    Database.add_record(test_data[:name], test_data[:dob])
  end
end

LOGGER = Logger.new(STDOUT)

test_data = [
  {
    name: 'Alice',
    dob: '14/07/1982'
  },
  {
    name: 'Bob',
    dob: '23/01/1992'
  },
  {
    name: 'Carol',
    dob: '09/12/1975'
  }
]

Parallel.map(test_data, in_processes: 2)  do |person|
  LOGGER.info("Seeding details about #{person[:name]}")
  DatabaseAccess.seed_database(person)
  # Complex test built on top of the identity that we've just seeded
end
```

```
I, [2023-05-15T13:34:15.480703 #75155]  INFO -- : Seeding details about Bob
I, [2023-05-15T13:34:15.480834 #75156]  INFO -- : Seeding details about Alice
I, [2023-05-15T13:34:15.481945 #75156]  INFO -- : Insert successful
E, [2023-05-15T13:34:15.481961 #75155] ERROR -- : Database rejected insertion, constraint violation
I, [2023-05-15T13:34:15.482423 #75156]  INFO -- : Seeding details about Carol
I, [2023-05-15T13:34:15.482646 #75156]  INFO -- : Insert successful
```

Now, in this example we could probably have a guess based on the process ids that are visible that it was Bob that had trouble in the database, but as these things scale up the ability to confidently guess about this can become both harder and riskier. This lead to our first major requirement for Herodotus, that there should be an easy way for us to identify which scenario has logged out a specific message.

## Corralling our test cases with Correlation Ids

Correlation ids are a common way to keep track of the flow of a given flow through a system, which makes them exactly what we are looking for to solve our current problem. Lets get Herodotus set up in the above example and then we will take a dive into how it works under the bonnet.

```ruby
module DatabaseAccess
  def self.seed_database(test_data)
    Database.add_record(test_data[:name], test_data[:dob])
  end
end

LOGGER = DVLA::Herodotus.logger

test_data = [
  {
    name: 'Alice',
    dob: '14/07/1982'
  },
  {
    name: 'Bob',
    dob: '23/01/1992'
  },
  {
    name: 'Carol',
    dob: '09/12/1975'
  }
]

Parallel.map(test_data, in_processes: 2)  do |person|
  LOGGER.new_scenario(person[:name])
  LOGGER.info("Seeding details about #{person[:name]}")
  DatabaseAccess.seed_database(person)
  # Complex test built on top of the identity that we've just seeded
end
```

```
[2023-05-15 13:58:43 20e66f0f] INFO -- : Seeding details about Alice
[2023-05-15 13:58:43 ab0506e8] INFO -- : Seeding details about Bob
[2023-05-15 13:58:43 20e66f0f] INFO -- : Insert successful
[2023-05-15 13:58:43 ab0506e8] ERROR -- : Database rejected insertion, constraint violation
[2023-05-15 13:58:43 45d88073] INFO -- : Seeding details about Carol
[2023-05-15 13:58:43 45d88073] INFO -- : Insert successful
```

There are two key changes to look at here. First up, we are now creating our logger with a call to `DVLA::Herodotus.logger`. This will give us a Herodotus logger that is pointed at the standard output. The other thing is that, at the start of each new scenario, we are explicitly informing Herodotus that this is a new scenario for it to keep track of[^2]. With this in place, we are able to see clearly which scenario it is that was having issues, confirming our earlier suspicious that it was Bob. Now, let's take a look at how [Herodotus](https://github.com/dvla/herodotus/blob/main/lib/dvla/herodotus/herodotus_logger.rb) manages all of these scenarios.

[^2]: While in this example we are giving it the name of the test data being seeded, this can be anything so long as it is unique. For example, in [Cucumber](https://cucumber.io/docs/tools/ruby/) we use the scenario name as an identifier.

```ruby
module DVLA
  module Herodotus
    class HerodotusLogger < Logger
      attr_accessor :system_name, :requires_pid, :merge, :correlation_ids

      def register_default_correlation_id
        @correlation_ids = { default: SecureRandom.uuid[0, 8] }
        reset_format
      end

      def new_scenario(scenario_id)
        update_format(scenario_id)
        merge_correlation_ids(new_scenario: scenario_id) if @merge
      end
      # Snip
    private

      def update_format(scenario_id)
        @correlation_ids[scenario_id] = SecureRandom.uuid[0, 8] unless @correlation_ids.key?(scenario_id)

        self.formatter = proc do |severity, _datetime, _progname, msg|
          "[#{@system_name}" \
            "#{Time.now.strftime('%Y-%m-%d %H:%M:%S')} " \
            "#{@correlation_ids[scenario_id]}" \
            "#{' '.concat(Process.pid.to_s) if requires_pid}] " \
            "#{severity} -- : #{msg}\n"
        end
      end
      # Snip
    end
  end
end
```

Here we can see that Herodotus (which is built on top of the existing Logger class, meaning you get all the built-in functionality for free) has a Hash it uses to keep track of the correlation ids for each scenario, keyed off of the identifier that it is given with each new scenario. When `new_scenario` is called, the provided key is added to the hash with a new id (provided that it isn't one that is already registered) and the format of the logger is then updated to pull through the correlation id that is associated with the provided identifier. Thus, when the logger is next called in that context, the correct correlation id will be logged out and we've got ourselves a nice simple way of keeping track of all our scenarios. Now, if you aren't familiar with `formatter`, this is [a method that exists on the default Ruby logger](https://ruby-doc.org/stdlib-2.6.4/libdoc/logger/rdoc/Logger.html#class-Logger-label-Format) that lets you specify the format you want messages to be logged out in. As mentioned, Herodotus is built on top of that Logger, so we can leverage this functionality.

The only methods that are built into the default logger that Herodotus has any really interaction are the five logging methods (`debug`, `info`, `warn`, `error`, `fatal`) and even then only as a small wrapper around it that leverages the sort of metaprogramming that Ruby really excels at. This also allows you to apply a specific correlation id on a log by log basis if this is required, with the majority of the heavy lifting being handed over to the built-in Ruby class.

```ruby
%i[debug info warn error fatal].each do |log_level|
  define_method log_level do |progname = nil, scenario_id = nil, &block|
    if scenario_id == nil
      super(progname, &block)
    else
      update_format(scenario_id)
      super(progname, &block)
      reset_format
    end
  end
end
```

## Configuring Herodotus to suit each consumer

Now, while the above logs are a great starting point, it's also important to allow for a degree of flexibility. As such, Herodotus allows for the following configuration changes to be made:

```ruby 
DVLA::Herodotus.configure do |config|
  config.system_name = 'person-database-testpack'
  config.pid = true
  config.merge = true
end
```

We'll have a quick overview of the first two before taking a dive into what is going on behind the scenes with the third.

First up, `system_name` is nice and simple. This allows you to add an overall system name to the logs that are being output. `pid` allows you to mimic the behaviour of the default logger, adding the process id to the output at the start of a log message. If we turn both of these on and re-run our test from before, we get an output that looks like this:

```
[person-database-testpack 2023-05-15 14:59:27 683dd5bb 94910] INFO -- : Seeding details about Alice
[person-database-testpack 2023-05-15 14:59:27 3425d11a 94911] INFO -- : Seeding details about Bob
[person-database-testpack 2023-05-15 14:59:27 683dd5bb 94910] INFO -- : Insert successful
[person-database-testpack 2023-05-15 14:59:27 3425d11a 94911] ERROR -- : Database rejected insertion, constraint violation
[person-database-testpack 2023-05-15 14:59:27 709eaf7d 94910] INFO -- : Seeding details about Carol
[person-database-testpack 2023-05-15 14:59:27 709eaf7d 94910] INFO -- : Insert successful
```

### Merging various instances of Herodotus together

`merge`, the final thing that can be configured within Herodotus is the most complex. Think back to our original example, but this time imagine that rather than the database being configured in a different module, it is something we are pulling in from an external package that also implements Herodotus. Let's have a look at what that would output:

```
[person-database-testpack 2023-05-15 15:13:01 9e5ca313 98217] INFO -- : Seeding details about Alice
[person-database-testpack 2023-05-15 15:13:01 d6812ffc 98218] INFO -- : Seeding details about Bob
[database-gem 2023-05-15 15:13:01 fefcf70f 98217] INFO -- : Insert successful
[database-gem 2023-05-15 15:13:01 99e3eb47 98218] ERROR -- : Database rejected insertion, constraint violation
[person-database-testpack 2023-05-15 15:13:01 a56c4b52 98217] INFO -- : Seeding details about Carol
[database-gem 2023-05-15 15:13:01 7539e3ee 98217] INFO -- : Insert successful
```

Suddenly, we are back to the problem we originally had, were no error case can be linked back to an individual testcase. This is where `merge` comes in. This allows us to merge the correlation id hashes across multiple instances of Herodotus, allowing us to get consistent correlation ids. We've already seen what is called when `merge` is set to true, back when we looked at the method `update_format` inside Herodotus, where the method `merge_correlation_ids` was called if it is active. Let's look at what that does:

```ruby
def merge_correlation_ids(new_scenario: nil)
  ObjectSpace.each_object(DVLA::Herodotus::HerodotusLogger) do |logger|
    unless logger == self
      logger.merge = false if logger.merge
      logger.correlation_ids = self.correlation_ids
      logger.new_scenario(new_scenario) unless new_scenario.nil?
    end
  end
end
```

Now, the above looks complex but if we break it down it's actually quite simple. First up, we are grabbing every instance of `HerodotusLogger` that currently exists by using the [ObjectSpace.each_object](https://ruby-doc.org/core-2.6.1/ObjectSpace.html#method-c-each_object) method, which will return an enumerator that we can use to iterate through all of them. While we are iterating through them, we make sure we don't try and merge this logger with itself, as that can lead to some nasty unexpected behaviour. Once the code is happy it has hold of a different logger, it first stops that logger from trying to merge with the others if has is set up to do so. Again, this is for safety as if that wasn't toggled off we'd end up in an endless loop. What we actually want is a single source of truth, which will be the logger highest up the stack, in our case the one that is created in the tests. Finally, we override that loggers collection of correlation ids and then get it to set itself to the current scenario, safe in the knowledge it will have a correlation id for this scenario as we've just given it that. 

Taking that into account and enabling merge on our logger, we get the following output:

```
[person-database-testpack 2023-05-15 15:29:47 594053ed 4415] INFO -- : Seeding details about Bob
[person-database-testpack 2023-05-15 15:29:47 2472e136 4414] INFO -- : Seeding details about Alice
[database-gem 2023-05-15 15:29:47 594053ed 4415] ERROR -- : Database rejected insertion, constraint violation
[database-gem 2023-05-15 15:29:47 2472e136 4414] INFO -- : Insert successful
[person-database-testpack 2023-05-15 15:29:47 cb266a96 4415] INFO -- : Seeding details about Carol
[database-gem 2023-05-15 15:29:47 cb266a96 4415] INFO -- : Insert successful
```

And there we have it, we are merging our ids across different instances of Herodotus allowing us trace problems as and when they arise, while also retaining the clarity that allowing each system that implements a logger to have that name reflected in all log messages. All in all, this makes for a much nicer debugging experience when something does go a bit strange in the codebase.

## Making tests even more human-readable

Right back at the start, I said that one of the most important things about these sorts of logs is that they should be easy for a human to read. As such, there is one last thing included in Herodotus that aids in that goal. There are [a collection of extensions to the default String class](https://github.com/dvla/herodotus/blob/main/lib/dvla/herodotus/string.rb) that allow you to simple apply colour to string, helping you easily highlight specific points of interest in your log messages. With a quick pass of colour to our loggers, we can quite quickly get something that looks like this:

{{< figure src="images/logger_output_in_colour.png" title="Logger output, now in colour" >}}

## Conclusion

So. that's the story of how we came up with Herodotus. It's by no means the most complex gem out there, but that's no bad thing. We were looking for something pretty lightweight to extend some already quite powerful functionality within the default logger, just in a way that makes it easier for us to follow the thread through when we are perusing our logs. We've certainly hit that goal and have even come up with some extra functionality along the way to help with running multiple instances of Herodotus in a way that abstracts that complexity away from the consumer.