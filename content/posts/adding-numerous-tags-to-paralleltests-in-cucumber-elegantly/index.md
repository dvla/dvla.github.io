---
author: "Tomos Griffiths"
title: "TiL: Adding numerous tags to ParallelTests in cucumber elegantly "
description: "Creating numerous tags to be included/excluded for ParallelTests in cucumber elegantly"
draft: false
date: 2024-07-02
tags:
  ["Ruby", "ParallelTests", "Tags", "Cucumber", "Testing", "Today I Learned"]
categories: ["TIL", "Ruby", "Cucumber", "ParallelTests", "Testing"]
ShowToc: true
TocOpen: true
---

Trying to run tests in parallel in your CI/CD pipeline is tricky at the best of times, but what if you have tests which should only be run in a certain environment and not as part of the CI build stage?

It is fairly simple to just tag a feature file with a unique tag that can then be passed to the `bundle exec rake` command in the functional-tests step, as an environment variable. However, this tag will not stop these tests being run as part of the CI stage unless you make changes to how the tags are being used.

A useful way is by adopting the approach described below:

## Using 'included' and 'excluded' tags

By using a set of `included` and `excluded` tags, we can pass in the tags we want to use within a particular rake task, and also the tags we want to exclude from this task. We can do this easily and elegantly by using an array for each as below:

```ruby
  # To exclude more tags in the running of the tests, add the argument to the 'excluded_tags' array below
  task :run, %i[threads feature_path tags] do |_t, args|
     args.with_defaults(threads: '1', feature_path: 'features', tags: [], excluded_tags: ['@excluded_tag1', '@excluded_tag2'])

    # puts the array of excluded_tags into a string to use in the tags argument below
    excluded_tags = args.excluded_tags.map { |tag| "not #{tag}" }.join(' and ') # output will be "not @excluded_tag1 and not @excluded_tag2" by default
    included_tags = args.tags

    tags = included_tags.concat(excluded_tags).join(' and ') # joins the two arguments together as a string, which will guard against empty arrays

    parallel.run(['-n', args.threads, '--type', 'cucumber', '--group-by', 'scenarios', '--serialize-stdout',
                  '--', '-f', cucumber_formatter, '--out', '/dev/null', '-f', 'progress',
                  '-t', tags, '--', args.feature_path])
  end
```

The `tags` can then be passed in to the `parallel.run` command we are all familiar with. This ensures that we are running the tests that have been tagged appropriately but also excludes any tags that should not be added to this command.

## Use case

An example of this would be if your test pack runs a `cron` job, where you only want certain tests/feature files to be run within this cron......with the caveat that these `cron` tests SHOULD NOT also be run in your CI stage upon a `push` or `merge to main`. The `excluded_tags` would then contain the "cron test" tags as a default as these should be ignored in your CI stage. Then in your `cron` step, use the `bundle exec rake` command as before, but then pass the "cron tests" tag in here instead, ensuring that all other tags will be ignored and only your appropriately tagged tests will be run.

This is more elegant than writing:

```ruby
  tags = args.tag.nil? ? 'not @exclude' : "#{args.tag} and not @excluded_tag1 and not @excluded_tag2 and not @excluded_tag3....."
```

By using the array, you can add an `excluded_tag`to the running of the tests without programmatically making any code changes to existing tests. This elegant method also looks cleaner.

## Conclusion

Using this method allows any test pack to run and/or ignore certain tests within certain environments when the need arises, without making any drastic code changes or amending the default `bundle exec rake` command to run the tests in parallel.
