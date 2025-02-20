---
title: "DVLA open-source projects"
date: 2023-02-27T18:03:22Z
draft: false
---

Ok so we don't code entirely [in the open](https://www.gov.uk/service-manual/service-standard/point-12-make-new-source-code-open) at the DVLA we do like to share useful libraries and utilities when we can. You can find a list of recent open-source projects below.

## AWS

### [aws-sqs-utility](https://github.com/dvla/aws-sqs-utility)

A command-line utility that provides the ability to read, write and transform messages on a AWS SQS queue. The following queue actions are available: List, Extract, Load, Delete.

### [sqs-extended-client](https://github.com/dvla/sqs-extended-client)

A library for managing large AWS SQS message payloads using S3. In particular it supports message payloads that exceed the 256KB SQS limit. It is largely a Javascript version of the Amazon SQS [Extended Client Library for Java](https://github.com/awslabs/amazon-sqs-java-extended-client-lib), although not an exact copy.

### [serverless-apigateway-log-retention](https://github.com/dvla/serverless-apigateway-log-retention)

A serverless framework plugin to control the retention of your ApiGateway access logs and execution logs.

## Ruby

### [simple-symbolize](https://github.com/dvla/simple-symbolize)

SimpleSymbolize takes a string and transforms it into a symbol. Why? Because working with symbols in Ruby makes for a good time.

### [hash_miner](https://github.com/dvla/hash_miner)

Ever experienced the pain of working with deeply nested hashes in Ruby? HashMiner expands the base Hash class in Ruby to provide helpful methods to traverse and manipulate your Hash Object regardless of complexity.

### [block-repeater](https://github.com/dvla/block-repeater)

A way to repeat a block of code until either a given condition is met or a timeout has been reached.

### [dvla-reportportal-ruby](https://github.com/dvla/dvla-reportportal-ruby)

This gem is a Ruby-Cucumber formatter which sends the test output to [Report Portal](https://reportportal.io/). This formatter supports plain 'ol Cucumber tests and those wrapped with parallel_tests.

### [dvla-herodotus-ruby](https://github.com/dvla/herodotus)

This gem provides a lightweight logger that adds functionality to the built in one, allowing for correlation ids to be tracked across various processes.

### [dvla-atlas-ruby](https://github.com/dvla/atlas)

This gem encapsulates a way of having a predefined set of properties within a test pack, built to work in tandem with the World functionality in Cucumber

### [postman-paf](https://github.com/dvla/postman-paf)

Open Source library to apply Royal Mail Rules & Exceptions to PAF (Postcode Address File) addresses when converting to a printable format.

### [dvla-browser-drivers](https://github.com/dvla/dvla-browser-drivers)

This gem has pre-configured browser drivers that you can use out-of-the-box for the development of your application.

# Dynamics 365

### [dataverse-helper](https://github.com/dvla/dataverse-helper)

This gem helps you integrate with Microsoft Dynamics using Microsoft Dataverse Web API. You can create, retrieve, delete or update a record without worrying about authentications as it's automatically managed behind the scenes.

# JavaScript

### [postman-paf-js](https://github.com/dvla/postman-paf-js)

Open Source library to apply Royal Mail Rules & Exceptions to PAF (Postcode Address File) addresses when converting to a printable format.
