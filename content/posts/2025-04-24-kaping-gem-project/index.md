---
author: "Kevin Upstill"
title: "KA-PING! a gem to integrated with Elastic and OpenSearch"
description: "Idiomatic may to build complex OpenSearch queries in Ruby."
draft: false
date: 2025-04-24
tags: ["Ruby, ElasticSearch, OpenSearch"]
categories: ["Ruby, Elastic"]
ShowToc: true
TocOpen: true
---

***You can only stretch elastic so far before it goes KA-PING!***


The starting point for creating a new DVLA gem to integrate with AWS OpenSearch and ElasticSearch was the amount of documentation 
there is surrounding the technology. There is a lot for a reason, it's a very powerful and useful tool,
but the flip side is the learning curve involved in understanding all the features and query structure. 

Our use case is we wanted squads to have a simple way to retrieve specific driver test data to use in their
tests without having to fully understand Elasticsearch and all its capabilities.

A lot of our tests require very specific data requirements which can result in complex search queries with multiple parameters and 
search terms to consider, so the code to write this query can ballon into a very complex nested JSON.

Out test platform architecture is based on Ruby, so we wanted a Ruby way to write large queries easily, which lead to the 
development of a query builder using the dot notation concept to chain the queries terms together.

We didn't need every aspect of the full OpenSearch capabilities to begin with, so we started with a sub set of search terms
which we commonly use. With this in mind there is further development work to cover more of the OpensSearch functions down the line.


## What a multiple search term query looks like with JSON notation

Probably the best way to demonstrate Kaping is to look at the traditional query construct, then the Kaping way.


## Traditional JSON query
```ruby

def multi
  date = DateTime.now.strftime('%Y-%m-%d')

  query = {
    query: {
      bool: {
        must: [
          { match_phrase: { 'fruit.destination': 'Market' } },
          { match_phrase: { 'fruit.code': 'LA-MOP' } },
          { match_phrase: { fruitStatus: 'Ripe' } },
          { match_phrase: { 'current.address.first_line': 'Plantation Row' } },
          { match: { 'fruit.category': 'Tropical' } },
          { range: { 'fruit.pickedDate' => { gte: '2025-08-21', lte: date } } },
          { range: { 'fruit.inspection.taken' => { gte: date } } },
          { range: { 'fruit.importDate' => { gte: date } } },
        ],
        must_not: [{ exists: { field: 'fruitEndDate' } },
                   { exists: { field: 'fruit.import.USATariffs' } },
                   { exists: { field: 'fruit.status.goneBad' } },
                   { match_phrase: { 'fruit.category': { query: 'B A NN NA AN', operator: 'or' } } },
                   { wildcard: { 'fruit.address.import.code': 'NIT*' } },
                   { wildcard: { 'fruit.address.import.code': 'NAT*' } },
                   { wildcard: { 'fruit.address.import.code': 'NOO*' } }],
        filter: { term: { 'licence.fruit': 'Valid' } },

      },
      match: { 'fruit.type': 'Tropical' },
    },
  }
end
```
In the above example the nested JSON can become quite a handful


## What the query looks likes with Kaping
```ruby
def multi
  date = DateTime.now.strftime('%Y-%m-%d')  
  
  q = DVLA::Kaping::Query.new('bool')  
  q.must.match('fruit.destination', 'Market').
    match_phrase('fruit.code', 'LA-MOP').
    match_phrase('fruitStatus', 'Ripe').
    match_phrase('current.address.first_line', 'Plantation Row').
    between('fruit.pickedDate', '2025-08-21', date.to_s).
    between('fruit.inspection.taken', '2025-08-21', date.to_s).
    between('fruit.importDate', '2025-08-21', date.to_s).
    match('fruit.category', 'Tropical').
  q.must_not.
    exists('field', 'fruitEndDate').
    exists('field', 'fruit.import.USATariffs').
    exists('field', 'fruit.status.goneBad').
    match_phrase('fruit.category',  query: 'B A NN NA AN', operator: 'or').
    wildcard('fruit.address.import.code', 'NIT*').
    wildcard('fruit.address.import.code', 'NAT*').
    wildcard('fruit.address.import.code', 'NOO*').
    filter('licence.fruit', 'Valid')
  q.to_json
end
```

The key point here is you don't need to worry about the nested JSON structure, the naming convention is intuitive and closely resembles
the OpenSearch syntax.

Let's break that down further

```Ruby
my_query = DVLA::Kaping::Query.new('bool')
```
This is the starting point for creating a query definition by calling an instance of the Kaping:: Query 
class and assigning it to 'my_query'. 

We set then set the type of query we want. The common ones are 'bool' or 'match' 
depending on your search context.

If don't require a complex query you could do a very basic match query:

```Ruby
my_query = DVLA::Kaping::Query.new('match_phrase', foo: 'Bar')
my_query.to_json
```
this is equivalent to writing this JSON query

```Ruby
my_query = { "query":
               { "match_phrase":
                   { "foo": "Bar" }
               }
           }
```

With Kaping the JSON formation is taken care off with the common ruby call .to_json

In the large example at the top, each line is a new search term, there are various different terms you can use depending on what 
functionality you require. 

> DVLA::Kaping::Query.new( **'match_phrase'**, foo: 'Bar')

#### Current sub list of terms are:

- **match_phrase** - match documents that contain an exact phrase
- **match** - full-text search on a specific document field
- **exist** -  search for documents that contain a specific field.
- **wildcard** - match a wildcard pattern, such as He**o
- **term** - search for exact term in a field.
- **prefix** - search for terms that begin with a specific prefix
- **regex** - search for terms that match a regular expression,  eg "[a-zA-Z]amlet"
- **between** - search for a range of values in a field

These are a mix of full-text queries and term-level queries, but they are most commonly use for our kind of searches, 
other can easily be added as requirements dictate.

The next part is the data field you want to search on

> DVLA::Kaping::Query.new('match_phrase', **foo**: 'Bar')

Then the last part is the data you are looking for, the data type can be a range, exact match string, regex or a filter for example

> DVLA::Kaping::Query.new('match_phrase', foo: **' Bar'**)

## Configuration
The Gem can be configured through a 'kaping.yml' file. Such configs as the logging level, result size and the AWS configs
can be set in the yaml, The config file can also be used to pick up any environment settings which is useful if running 
in a CI pipeline.


## The Client
Currently we have a AWS OpenSearch client which takes care of the Sig4C signing. The AWS credentials can either be supplied
as ENV variables or through a profile.

## Search
As long as your client is configure you can also use the optional built-in search facility

```ruby
my_query = DVLA::Kaping::Query.new('match_phrase', foo: 'Bar')
my_query.to_json
response = DVLA::Kaping.search(my_query)
response.dig('hits', 'hits')
```

## Opens Source

[External link to Kaping in rubygems](https://rubygems.org/gems/dvla-kaping)

We wanted to open source this Ruby Gem to a wider audiance so that others can also benefit from simplifying their OpenSearch queries. We also
embrace the feedback to further enhance the Gem and increase it's scope.


## Further Development

As mentioned before, not all functionality of OpenSearch has been implemented, but any requests to expand will be taken into consideration.
The code base for Kaping is small, the query builder is 40 lines of code. The separation of terms into their own module makes it easy to
add additional query terms as required



