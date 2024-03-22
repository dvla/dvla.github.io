---
author: "Gareth Griffiths"
title: "Streaming Application State Changes using DynamoDB streams and EventBridge Pipes"
# description: "DVLA's use of AWS Step Functions to streamline their application processing"
draft: false
date: 2024-03-21
tags: ["AWS", "AWS EventBridge Pipes", "DynamoDB", "DynamoDB Streams"]
categories: ["AWS", "AWS EventBridge Pipes", "DynamoDB", "DynamoDB Streams"]
ShowToc: true
TocOpen: true
---

## Introduction
At DVLA, we offer various types of applications for individuals to apply for, and managing or monitoring the progress of these applications can present challenges. This is where the [Driver and Vehicle account](https://www.gov.uk/driver-vehicles-account) comes into play - a centralised platform for viewing or managing driver and vehicle-related information, updating contact details, or tracking the progress of ongoing applications.

## Application processing and persistence
DVLA has recently developed a comprehensive application process using AWS Step Functions. Each application is stored in DynamoDB, and we record actions taken on the application, representing state changes. As applications progress, they transition through various phases, with corresponding status updates recorded. This data can then be utilised to provide tracking information for both internal and external users.

## Application status updates
DynamoDB offers a highly beneficial feature called DynamoDB streams, which aligns perfectly with our requirements. Whenever actions such as status updates are written to the database, they trigger events to be added to the stream. An example of such an event is:

```json
{
   "Records":[
      {
         "eventID":"1",
         "eventName":"INSERT",
         "eventVersion":"1.0",
         "eventSource":"aws:dynamodb",
         "awsRegion":"eu-west-2",
         "dynamodb":{
            "Keys":{
               "sk":{
                  "S":"#ACTION"
               }
            },
            "NewImage":{
               "sk":{
                  "S":"#ACTION"
               },
               "type":{
                  "S":"ApplicationStatusUpdated"
               },{
                "status": {
                  "S": "Preparing"
                }
               }
            },
            "SequenceNumber":"111",
            "SizeBytes":26,
            "StreamViewType":"NEW_AND_OLD_IMAGES"
         },
         "eventSourceARN":"stream-ARN"
      },
    ]
}
```

## EventBridge Pipes
Now that we have status updates streaming out of our DynamoDB database, the next step is to transform, enrich, and relay these updates to the driver and vehicle accounts domain. The team responsible for developing the driver and vehicle account has deployed an SQS in their account and established cross-account access for the application account. This step provided the final piece needed to set up AWS EventBridge Pipes.

{{<figure src="images/eventbridge-pipes.png" caption="AWS EventBridge Pipes">}}

### Event Source
AWS offers a variety of data services that can be configured for the source of the pipeline, one of which is DynamoDB streams, perfect for our solution. In our deployment of AWS EventBridge Pipes, we selected the DynamoDB stream as the primary source.

### Filtering
This is a crucial step that can potentially save invocation costs, especially if using Lambda in the subsequent enrichment process. Here, stream events can be filtered using various [patterns](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-event-patterns-content-based-filtering.html). For instance, in our example of a stream event provided above, we would require a pattern like the following to match events starting with `#ACTION` and having an action type of `ApplicationStatusUpdated` or `ApplicationCreated`.

```json
{
  "dynamodb": {
    "$or": [
      {
        "NewImage": {
          "sk": {
            "S": [
              {
                "prefix": "#ACTION#"
              }
            ]
          },
          "type": {
            "S": [
              {
                "prefix": "ApplicationStatusUpdated"
              }
            ]
          }
        }
      },
      {
        "NewImage": {
          "sk": {
            "S": [
              {
                "prefix": "#ACTION#"
              }
            ]
          },
          "type": {
            "S": [
              {
                "prefix": "ApplicationCreated"
              }
            ]
          }
        }
      }
    ]

  }
}
```
The example provided employs the `$or` statement in the filter pattern, signifying that two conditions must be met. This accommodates recording the initial creation of the application as an action. Consequently, if our action fulfils either of the filtering conditions, it progresses to the subsequent step in the pipeline - Enrichment. This approach ensures that we don't unnecessarily invoke a lambda for an event that doesn’t contribute to the workflow.

### Enrichment
Enrichment serves as an optional step in the pipeline, offering the flexibility to either skip it or leverage it for enrichment and/or transformation purposes. In our scenario, we utilise both enrichment and transformation. The driver and vehicle service assigns a unique ID to each user, which is a mandatory field required in the payload sent to the service.

Although the actions we stream lack the unique ID of the driver and vehicle services, the persisted application retains this information, owing to the close relationship between both services during the application process. Therefore, our enrichment step entails querying our application domain to obtain the driver and vehicle services' unique ID. Each action contains the application ID.

Subsequently, we construct a payload adhering to the agreed schema, incorporating the current application status, the unique ID of the driver and vehicle services, and additional metadata. This constructed payload is then forwarded to the final stage of the pipeline.

### Target
The concluding segment of the pipeline involves determining the target for the event. AWS offers a diverse array of services for this purpose, such as SQS, Lambda, API Gateway and many more. Unfortunately, at the time of writing, configuring cross-account services was not feasible, although we anticipate this to be addressed in the AWS roadmap - a highly sought after solution for us.

Consequently, we opted for AWS Lambda as the target, leveraging the AWS SDK to dispatch messages to the target SQS in the driver and vehicle service account, once permissions were configured for cross-account access. Through this setup, our Lambda function seamlessly transmits application status updates to the driver and vehicle service, facilitating users in tracking their applications within the central driver and vehicle account.

## Error handling
In the realm of event-driven architectures facilitated by EventBridge Pipes, robust error handling mechanisms are indispensable. This segment addresses the strategies employed to manage errors effectively throughout the pipeline, ensuring the seamless flow of data and processes.

EventBridge Pipes streamlines error handling through its intuitive integration capabilities. One notable feature is the provision to configure dead letter queues, offering granular control over error management at different stages of the pipeline. This empowers developers to isolate and address errors specific to each component, thereby enhancing the fault tolerance and resilience of the system.

Moreover, EventBridge Pipes facilitates the implementation of retry logic, a pivotal aspect of error mitigation strategies. Developers can incorporate dead letter queues at each stage of the pipeline or opt for a unified queue to manage all errors.

### Handling DyanmoDB stream Failures
Encountering failures in the DynamoDB stream posed a significant hurdle. The problem originated from the type of messages sent to the dead letter queue, all of which were DynamoDB stream events. These iterator events contain shard metadata (as detailed below), this enabled querying of the shard to obtain the record via the AWS CLI.

```json
{
  "context": {
    "partnerResourceArn": "arn:aws:pipes:eu-west-2:123456789123:pipe/dev-application-pipe",
    "condition": "RetryAttemptsExhausted"
  },
  "version": "1.0",
  "timestamp": "2024-03-08T00:00:00Z",
  "DDBStreamBatchInfo": {
    "shardId": "shardId-00000001234567891234-aaa1abcd",
    "startSequenceNumber": "1234567800000000001234567899",
    "endSequenceNumber": "1234567800000000001234567891",
    "approximateArrivalOfFirstRecord": "2024-03-08T00:00:00Z",
    "approximateArrivalOfLastRecord": "2024-03-08T00:00:00Z",
    "batchSize": 1,
    "streamArn": "arn:aws:dynamodb:eu-west-2:123456789123:table/dev-application-domain/stream/2023-01-01T00:00:00Z"
  }
}
```

### Steps to Obtain Event
Obtain the ARN for the stream, which can be from the AWS DyanmoDB streams and exports dashboard or querying using the AWS CLI or obtaining it from the DyanmoDB streams event on the dead letter queue. 

Getting the shard iterator, which returns the shard ID for the next command:

```bash
aws dynamodbstreams get-shard-iterator \
--stream-arn arn:aws:dynamodb:eu-west-2:123456789123:table/dev-application-domain/stream/2023-01-01T00:00:00Z \
--shard-id shardId-00000001710846791106-82d20ebc \
--shard-iterator-type TRIM_HORIZON
```

This should return the required shard iterator

```bash
{
    "ShardIterator": "arn:aws:dynamodb:eu-west-2:123456789123:table/dev-application-domain/stream/2023-01-01T00:00:00Z|1|AAAAAAAAAAPmKq1E+po5C3vW2v8IXtK8lOLDiu3"
}
```

Grab the value of the shard iterator and use it in the next command which will then provide the DynamoDB stream event:

```bash
aws dynamodbstreams get-records --shard-iterator arn:aws:dynamodb:eu-west-2:123456789123:table/dev-application-domain/stream/2023-01-01T00:00:00Z|1|AAAAAAAAAAPmKq1E+po5C3vW2v8IXtK8lOLDiu3
```

This returns something like below:

```bash
{
  "Records": [
    ... Some Records
  ]
}
```

The solution above is straightforward to script. However, it presents an issue as the engineer handling the support task needs to possess the appropriate permissions. Subsequently, they must either manually craft the command or develop a script to retrieve the information, all within the constraint of a 24-hour retention period on the DynamoDB stream event. Should the defect result in numerous event failures, this process would swiftly become time-consuming and arduous.

### Automating the Solution
To tackle this challenge, we formulated an automated solution. Essentially, it entailed setting up a Lambda function triggered by events added to the dead letter queue. The Lambda then retrieves and handles the events from the dead letter queue. Utilising the AWS SDK, the Lambda queries the DynamoDB stream,  disassembles the event, and reinstates it in the dead letter queue in the parsed format.

With our queue retention period configured to 14 days, we've allotted ample time for resolution. Consequently, we can promptly visualise problematic records within the dead letter queue by polling for messages. This streamlined method enables our support team to swiftly pinpoint issues in the data and promptly tackle them, thereby reducing the steps needed to resolve them.

## Summary
In this comprehensive exploration of DVLA's application processing and error handling strategies, several key insights have emerged. The implementation of EventBridge Pipes has made it much simpler to manage application updates to the centralised platform for tracking and managing driver and vehicle-related information.

The integration of dead letter queues and retry logic has bolstered the system's resilience against failures, ensuring the seamless flow of data and processes. Despite challenges posed by DynamoDB stream failures, DVLA's proactive approach led to the development of an automated solution, extending the retention period of dead letter queues and facilitating rapid issue resolution.