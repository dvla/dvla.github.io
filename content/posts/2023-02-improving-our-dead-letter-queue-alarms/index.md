---
author: "Tom Collins"
title: "Improving our DLQ alarms"
description: "How we learnt the hard way about dead letter queue alarms and retention limits."
draft: true
date: 2023-02-27
tags: ["dead letter queue", "DLQ", "alarms", "AWS"]
categories: ["AWS"]
ShowToc: true
TocOpen: true
---

Dead letter queues (DLQ) are often used to catch items that have not been successfully processed or delivered when using queues based integration patterns (\*).

Typically engineering teams will want to be alerted to the presence of items on a dead letter queue so they can do something about it e.g. redrive messages back to a source queue, inspect the messages for further understanding of an issue.

- _To be more accurate DLQs are used with a wide within a range of service integration patterns (queues, buses, topics etc)._
