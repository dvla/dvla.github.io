---
author: "Peter Squire"
title: "Checking for existing user in Atlas"
description: "Check if a user exists in Atlas and create one if it doesn't"
draft: false
date: 2025-05-29
tags: ["Atlas", "Ruby", "Today I Learned"]
categories: ["Atlas", "Ruby"]
ShowToc: true
TocOpen: true
---

Ever had to create users in MongoDb Atlas in AWS for use when making database enquiries in a functional test written in Ruby?  
Ever wished you could leave it to the test pack to check if one already exists and create one when required, instead of hand cranking a new one every single day of your life?  

Well now you can!!!  

#### But, first things first

What is MongoDB Atlas and what do its users do?

Atlas is a repository of database users. A user assumes a role which has database access and permissions.  

Access could be to one or more databases.  
Permissions could be read, read/write, etc.

All the info you could ever want is here: https://www.mongodb.com/docs/atlas/

#### Why do this? Why not create a new user every time?  

Because Atlas users sit outside of a test pack and can be used for the whole time they exist.
Plus, creating & deleting users for every test is pointless and expensive, data-wise.

#### Is there anybody in there...?  

Assuming you have atlas installed and configured on your machine...

Let's check the existing users:

In Terminal: 

```ruby
atlas dbusers list
```

This will return a list of the current users set up in the MongoDB Atlas instance you're connected to.  

The list will look something like this:  

USERNAME                                                                                      DATABASE
test-repo-admin-user                                                                          admin
mongodb-realm-triggers_realmapp-service-name-here-cluster                                     admin
mongodb-real-service-name-here-mongodb-atlas                                                  admin
arn:aws:iam::<ACCOUNT_NUMBER>:role/<ASSUMED_ROLE_TITLE>                                       $external
arn:aws:iam::<OTHER_ACCOUNT_NUMBER>:role/<OTHER_ROLE_TITLE>                                   $external


Atlas also has a method which can be used to search for a specific user  

```atlas dbusers describe #{enquirier_username} --authDB \\<database> -o json```

which returns a boolean value.

#### Code

Add this to the env.rb in your test pack.

note: the 'LOG.info' commands are to output info to console. 

```ruby

require 'date'

date_today = Date.today.strftime('%Y-%m-%d')
account_num = ENV.fetch('AWS_ACCOUNT_NUMBER', nil)
stage = ENV.fetch('SERVERLESS_STAGE', nil)
service_name = ENV.fetch('SERVICE_NAME', nil)

enquirier_username = "arn:aws:iam::#{account_num}:role/#{stage}-#{service_name}-retrieve-role"

LOG.info { 'Checking if retrieve enquiry user exists'.cyan }
enquirer_exist = system "atlas dbusers describe #{enquirier_username} --authDB \\$external -o json";

if enquirer_exist == false
LOG.info { 'Creating enquirer user in Atlas: '.cyan + "'#{enquirier_username}'" }
system "atlas dbusers create readAnyDatabase -u #{enquirier_username} --awsIAMType ROLE --deleteAfter #{date_today}T23:59:59Z"
else
LOG.info { 'Enquiry user already created: '.cyan + "'#{enquirier_username}'" }
end```