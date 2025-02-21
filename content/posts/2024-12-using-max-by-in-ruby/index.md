---
author: "Tomos Griffiths"
title: "TiL: Using the max_by method in Ruby"
description: "Using max_by to filter out data based on the highest value for a particular field"
draft: false
date: 2025-02-20
tags: ["Ruby", "max_by", "Testing", "Today I Learned"]
categories: ["TIL", "Ruby", "max_by", "Testing"]
ShowToc: true
TocOpen: true
---

When trying to get an upper or lower value for a certain field in an object can be cumbersome. But when using the `max_by` method, this can actually be done quite elegantly. For example:

```ruby
persons = [
  { "name" => "John McJohnson","age" => 34, "topScoreAtBowling" => 205},
  { "name" => "Davey Jones","age" => 304, "topScoreAtBowling" => 300},
  { "name" => "Willy Wonka","age" => 50, "topScoreAtBowling" => 200}
]
```

If we had a bowling competition and wanted to see who had the `topScoreAtBowling` field, rather than looping through the array and looking at each object, we can simply use the `max_by` command as follows:

```ruby
persons.max_by { |person| person['topScoreAtBowling']} #{"name"=>"Davey Jones", "age"=>304, "topScoreAtBowling"=>300}
```

The resultant object would be returned, and could be processed into their "Hall of Fame"!

## Conclusion

The `max_by` method is very convenient when dealing with a large array of objects, and you are required to find the highest value for a certain field. Alternatively the `min_by` method can be used to find the lower value of a certain field
