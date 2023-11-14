---
author: "Paul Lewis"
title: "Linux timeout command"
description: "Ensuring a stale drone step is timed out"
draft: false
date: 2023-10-14
tags: ["Drone", "Testing", "AWS", "Today I Learned"]
categories: ["Drone", "Testing", "AWS", "TIL"]
ShowToc: true
TocOpen: true
---

When a drone build is configured to both deploy and then delete AWS services, it is vital the AWS clean up step runs to maintain
the AWS stacks within their limit.

To ensure these steps run before the build times out, you can wrap your cucumber command in the linux timeout command within the drone pipeline

example:
````
commands:
    - cd functional-tests || exit 1
    - timeout -k 10 10m bundle exec rake test
````

## Linux Timeout
The timeout command is a command-line utility that will run the specified command for a given period of time, once that period of time is reached
if the given command is still running it will be terminated.

timeout command:
````
timeout [OPTIONS] DURATION COMMAND [ARG]…
````

### Duration
The duration is an integer followed by one of the following units, if no unit is used then it defaults to seconds:

* s - seconds (default)
* m - minutes
* h - hours
* d - days

### Signal flag
If no signal is given, timeout sends the SIGTERM signal to the managed command when the time limit is reached. You can specify which signal to send 
using the -s (--signal) option.

### Kill-after flag
To ensure the command is killed, you can use the -k (--kill-after) flag with a specified time period. When the time limit is reached, the timeout command 
will then send the SIGKILL signal.

### Preserve-status flag
There is a preserved status flag that can be passed in, when this is used timeout will always returns 124 if it times out.


## Conclusion
Prior to introducing this command into our pipelines, we had many drone build failures where, for a variety or reasons, the functional test step would hang and the 
build would time out, our 'AWS clean up step' would not execute and this resulted requiring manual intervention to clean them up (before we exceed our capacity).

Since the introduction of this command we have seen a marked reduction in our drone builds timing out
