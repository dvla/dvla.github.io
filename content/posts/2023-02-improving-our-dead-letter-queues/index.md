---
author: "Tom Collins"
title: "Improving our dead-letter queues"
description: "Lessons learnt the hard way about dead letter retention limits and alarm configuration."
draft: true
date: 2023-02-27
tags: ["DLQ", "AWS"]
categories: ["AWS"]
ShowToc: true
TocOpen: true
---

Dead letter queues (DLQ) are often used to catch items that have not been successfully processed or delivered when using queue based integration patterns (\*).

Typically engineering teams will want to be alerted to the presence of items on a dead letter queue so they can do something about it e.g. redrive messages back to a source queue, inspect the messages for further understanding of an issue.

- _To be more accurate DLQs are used with a wide within a range of service integration patterns (queues, buses, topics etc) and can be plugged into many cloud vendor managed services to help with processing or delivery failures._

## Background

IMAGE OF DLQ

## Handling items on a dead-letter queue

We don't want items to appear on a dead-letter queue, it means something as probably gone wrong somewhere, and that some remdiating action needs to be taken. This could include:

### Replaying items back to an originating queue

If you're using AWS, and have the appropriate permissions, re-drive can be achieved [through the AWS console](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-configure-dead-letter-queue-redrive.html). This capability is not exposed through an API so may be something you need to implement yourself if the console is not an option in a production environment.

You could also use a utility like the DVLA [aws-sqs-utility](https://github.com/dvla/aws-sqs-utility) to read message from the DLQ, inspect, and replay onto another queue.

Other scenarios and services may require a different approach but you should think about this, and the permissions required, while designing your solution, especially if the processing of items is time sensitive. This could include read and write permissions for multiple queues as well as being able to decrypt and encrypt messages.

## Using AWS SQS as a dead letter queue

Some info on 14 day limit, managed re-drive through console only.

### AWS alarms and PagerDuty

The alarm fires once and latches open.

## Firing an alarm each time items are added to a dead-letter queue

xxx

## Firing an alarm every day

{{< figure src="images/sqs-daily-alarm.png" title="An example of an alarm that is re-triggered every day." >}}
