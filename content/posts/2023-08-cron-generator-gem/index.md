---
author: "Thomas Feathers"
title: "Dronicron - The cron generator gem"
description: "A personal project to understand cron jobs that led to a Ruby gem"
draft: false
date: 2023-08-23
tags: ["Ruby", "cron", "Testing", "Today I Learned", "Ruby Gem", "Drone"]
categories: ["TIL", "Ruby", "Kernel", "Testing", "Gem", "Cron", "Drone"]
ShowToc: true
TocOpen: true
---
## Disclaimer

This gem is currently only available to DVLA employees with a view to go open source in the future.

## How it started

I was setting up cron jobs in Drone for the CI/CD pipeline on the Application TYR project. We needed to stagger build times to ease pressure on
the CI/CD infrastructure.

I found the UI in Drone to be rather restrictive regarding the schedules you could set for a cron job.

The Drone UI:

![UI](images/cron%20ui.png)

This is restrictive if you want to run every 15 minutes or something obscure like every other day etc.

## The Drone cli

Drone does have a cli that allows you to adjust the cron schedule down to these granular details but the syntax isn't the most readable.
It's easy to forget (_at least for me_) what to enter where if you're not using it regularly.

Example of a schedule to pass in to the cli for running 25 minutes passed the hour, between the hours of 7am and 5pm.

```ruby
0 25 6-16 * * *
```

Personally, not my favourite! I set out on a mission to write my own cli that would enable me to set these ridiculous schedules without having to remember or constantly look up
which * represents which part of the schedule.

## Dronicron - The Cron Generator

I know, epic name, right!?

I decided to see if I could create a cron generator of my own that can use the Drone cli to do my bidding. Dronicron is the result of that crusade. 

It has several uses which are:

- _generate_
- _execute_
- _schedule_

Each option has its own help by appending the `-h` flag. 

**Example**:

```
dronicoron generate -h
```

Output:

```shell
Usage: generate
    -r, --repo, REPO_NAME            Mandatory. Name of the repository that will host the cron job
    -n, --cron, CRON_NAME            Mandatory. Name to be assigned to the cron job:
                                       Format: alphanumeric separated by hyphen (-). e.g. 'my-cron-name'
    -s, --schedule, SCHEDULE         Optional. Standard schedule for setting a time/date to run. Defaults to hourly if omitted.
                                       Can be one of:
                                        - hourly
                                        - daily
                                        - weekly
                                        - out_of_hours
                                        - weekend
                                        - office_hours
                                       'out_of_hours' does have a shortened version of 'oo' - Added because I'm lazy!
    -b, --branch, BRANCH             Optional. The branch to run the cron against. Defaults to master if omitted
    -a, --addargs, ADDITIONAL_ARGS   String containing additional args to customise the cron job. Option and value can be separated by a comma (:) or equals sign (=):
                                       days_of_the_month : Define the days of the month to run. -
                                                               value can be 1-31
                                                               Example singular: 'days_of_the_month:15'
                                                               Example multiple: 'days_of_the_month:1-6' or 'days_of_the_month:1-6,7-12' or 'days_of_the_month:1-9,10-17,31'
                                       hours             : Define the hours to run -
                                                               value can be 0-23 where 0 is midnight
                                                               Example singular: 'hours:3'
                                                               Example multiple: 'hours:8-12' or 'hours:0-3,16-23' or 'hours:0-12,18'
                                       minutes           : Define the minutes to run -
                                                               value can be 0-59 where 0 is minute that lands on the new hour
                                                               Example singular: 'minutes:30' or 'minutes:0-30'
                                                               Example multiple: 'minutes:0-30' or 'minutes:0-29,50-59' or 'minutes:0-29,59'
                                       months            : Define the months to run -
                                                               value can be 1-12 where 0 is Sunday or JAN-DEC in a logical sequence i.e not DEC-JAN
                                                               Example singular: 'months:1' or 'months:JAN'
                                                               Example multiple digits: 'months:1-3' or 'months:1-3,5-7' or 'months:1-3,5-7,10'
                                                               Example multiple words: 'months:JAN-MAR' or 'months:JAN-MAR,JUN-OCT' or 'months:JAN-MAR,JUN-SEP,NOV'
                                                               Example multiple mix: 'months:JAN-MAR,5-6'
                                       seconds           : Define the seconds to run -
                                                               value can be 0-59 where 0 is the second that lands on the new minute
                                                               Example singular: 'seconds:30'
                                                               Example multiple: 'seconds:0-29' or 'seconds:0-29,50-59' or 'seconds:0-29,59'
                                       week_days         : Define the the week days to run -
                                                               value can be 0-6 where 0 is Sunday or MON-SUN
                                                               Example singular: 'week_days:0' or 'week_days:SUN'
                                                               Example multiple digits: 'week_days:0-3' or 'week_days:0-2,4-6' or 'week_days:0-1,3-4,6'
                                                               Example multiple words: 'week_days:SUN-TUE' or 'week_days:SUN-TUE,THU-SAT' or 'week_days:SUN-MON,WED-THU,SAT'
                                                               Example multiple mix: 'week_days:SUN-TUE,4-6'
                                     addargs example usage:  --addargs="minutes=10 hours:10-15"
    -e, --example                    Prints an example of using generate
```
# Usage

## Generate

Caveat: this gem does A LOT so you can see the full readme for it [HERE](https://bitbucket.tooling.dvla.gov.uk/projects/QE/repos/cron-generator/browse/README.md)

`generate` (or `g` for short) will allow you generate a Drone cli command that you can run yourself after exporting your Drone credentials (drone token and drone server).  

This command has several options that can be passed in, such as:

- `-r` or `--repo` (the drone repo for the cron)
- `-n` or `--cron` (the name for your cron job)
- `-s` or `--schedule` (a selection standard schedules)
- `-b` or `--branch` (the branch on which to run your cron - defaults to master)
- `-a` or `--addargs` (additional arguments - the custom scheduler)

Generate is run like so:

```shell
  dronicron generate -r "my_repo" -s "weekly" -n "hourly-cron" -a "hours:8-18 minutes:25" -b "main"
``` 

  or shorthand

```shell  
  dronicron g -r "my_repo" -s "weekly" -n "hourly-cron" -a "hours:8-18 minutes:25" -b "main"
```

Output:

```shell
Your cli to run is:
drone cron add my_repo hourly-cron "0 25 7-17 * * 0" --branch main

NOTE:
Remember to export your DRONE_TOKEN and DRONE_SERVER prior to running.
or
use the `execute` command and pass in the drone credentials. See `dronicron execute --help` for more info.

Example:
dronicron execute -r 'my-repo' -c 'my-cron' -s 'weekend' -d 'your server' -t 'your token'
```

Let's focus on the `--addargs` (or `-a` for short) flag here as this is where the scheduling takes place. This flag has the following options:

- seconds
- minutes
- hours
- week_days
- months
- days_of_the_month

You set the values with = or : as delimiters:

# Here are some examples:

Run the job between 7am and 5pm.

```shell
  dronicron generate -r "my_repo" -n "hourly-cron" --addargs "hours:7-17"
```


Set the schedule the run Sun-Fri every week.

```shell
 dronicron g -r "my_repo" -n "hourly-cron" -a "week_days:0-5" 
```


Let's mix them to go really insane on it. This would run January-March, Monday-Friday, 8am-6pm, half passed every hour.

```shell
  dronicron g -r "my_repo" -n "hourly-cron" --addargs "months:1-3 week_days:1-5 hours:8-18 minutes:30"
```

The last example is silly but shows how flexible your schedule can be depending on your project's needs. There is validation written into these ranges to match what Drone allows.

This is covered in the gem's Readme.

## Execute

Execute does everything that Generate does but it will also create your cron job in Drone. This requires the addition of two mandatory flags to the cli.

- `-d` or `--drone_server`
- `-t` or `--drone_token`

These two flags can be found in the settings of your Drone repo.

Example usage:

```shell
  dronicron exec-r "my_repo" -n "hourly-cron" --addargs "hours:8-18 minutes:30" -b "main" -t "my_token" -d "my_server"
```

Output:

```shell
  Cron deployed
  Job: dronicron exec-r "my_repo" -n "hourly-cron" --addargs "hours:8-18 minutes:30" -b "main" -t "my_token" -d "my_server" 
  Drone credentials: drone_server = my_server drone_token = my_token
```

You will see your cron job in Drone.

## Schedule

This is more of an educational option that allows you just pass in the additional arguments and return the Drone schedule in the Drone cli format.

This could be used to help you understand the drone scheduler or it can just be used for you to use in the Drone CLI.

Basic example:

```shell
  dronicron s -a "hours:8-18 minutes:12"
```

Output:

```shell
Your schedule is:

  0 12 7-17 * * * 
```

That's literally all there is to the schedule option!

## Installation

This was just a personal project but if you choose to use or check out the gem, it's currently stored in Nexus. 

Add the gem `cron-generator` to your Gemfile under the nexus gem group or you can install it manually using `gem install cron-generator -s path_to_gem_group+private`

## Summary

I enjoyed writing this gem. It has definitely broadened my understanding of cron scheduling with Drone.
It has (for me) simplified cron scheduling. Maybe it can for you, too.
