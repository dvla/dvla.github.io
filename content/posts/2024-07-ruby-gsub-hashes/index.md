---
author: "George Bell"
title: "TiL: Doing multiple replacements with gsub"
description: "How to make multiple changes to a string with only one gsub call"
draft: false
date: 2024-07-30
tags: ["Ruby", "Gsub", "Today I Learned"]
categories: ["TIL", "Ruby"]
ShowToc: true
TocOpen: true
---

`gsub` is great. It lets you replace characters in a string. All you have to do is give it a regex that will match whatever it is you want to replace and the string that you'd like to replace each match with. For example, if you wanted to replace the time of 18:00 with something a bit more human-readable, you could do the following:

```ruby
'As always, tea time is at 18:00.'.gsub(/18:00/, '6pm')
# => "As always, tea time is at 6pm."
```

But what if you wanted to do something different for each match? Well, because `gsub` returns a string you can chain the methods together, which looks like the following:

```ruby
'There are 5 cakes and 3 teapots on the table'.gsub(/5/, 'several').gsub(/3/, 'a few')
# => "There are several cakes and a few teapots on the table"
```

This can quickly become unmanagable. However, there is another way we can interact with `gsub`. Instead of simply giving it a string as the second argument, we can pass in a hash of matches, with the keys being potential matches and the values being the replacement strings.

```ruby
tea_party_matches ={
    '3' => 'a few',
    '5' => 'several'
}
'There are 5 cakes and 3 teapots on the table'.gsub(/\d/, tea_party_matches)
# => There are several cakes and a few teapots on the table
```

This tends to be much more readable, particularly when you get into more real-world examples with complex regex statements matching several different things you might want to replace. 