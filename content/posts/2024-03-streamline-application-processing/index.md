---
author: "Gareth Griffiths"
title: "How AWS Step Functions helped streamline the DVLA's application process"
# description: "DVLA's use of AWS Step Functions to streamline their application processing"
draft: false
date: 2024-03-18
tags: ["AWS", "AWS Step Functions"]
categories: ["AWS", "AWS Step Functions"]
ShowToc: true
TocOpen: true
---

## Introduction
Harnessing the power of AWS Step Functions, the Driver and Vehicle Licensing Agency (DVLA) embarks on a transformative journey to streamline its operations. With over 50 million registered drivers and 40 million vehicles, the DVLA navigates various applications, from driving licences to vehicle registrations. This blog unveils the approach adopted by DVLA to transform its application process, ensuring flexibility and customisation for each application type.

## Background
As the custodian of millions of driver and vehicle records, DVLA grapples with the complexity of managing diverse applications. From initial licence acquisition to periodic photo licence renewals and vehicle-related transactions, the spectrum of services demands a cohesive yet adaptable framework.

## Challenge
DVLA’s organisational structure traditionally compartmentalises teams into the driver or vehicle-focused domains. Integrating disparate knowledge bases becomes an arduous task, compounded by the intricate regulations governing both drivers and vehicles.

## Solution approach
DVLA carried out the analysis of its application landscape, identifying common phases traversed by all applications. This critical insight formed the foundation for crafting a unified application workflow, capable of seamlessly orchestrating diverse applications through their respective journeys.


## Technology choice
Building upon existing expertise with AWS infrastructure, DVLA leveraged AWS Step Functions to architect its solution. The decision was reinforced by the robustness and adaptability offered by AWS Step Functions, aligning seamlessly with DVLA’s cloud-native strategy.

## Introducing the application process engine aka APE
This is where the application process engine was born, a dynamic workflow engine designed to navigate applications through various stages. APE embodies versatility, executing generic business functionalities common to most applications.

{{<figure src="images/application-process-engine.png" caption="Application process engine"  class="center-caption">}}

The aforementioned diagram offers a birdseye view of the orchestration of the application across its diverse phases. The workflows dedicated to driver applications are compartmentalised and self-contained, executing only the driving licence application-specific steps essential for completing each phase.

## Workflow within a workflow 😕?
In our approach to handling the workflow for driving licences and vehicle-related logic, we opted to define sub-workflows for each application phase. For instance, in the scenario where licence holders are mandated to renew their photo on their driving licence, they now have the option to upload a photo of their choice, presenting a unique "selfie opportunity." Should they choose this route, a designated clerk within the DVLA is tasked with conducting meticulous checks on the submitted image to ensure compliance with standards.

However, the preparatory steps involved in photo submission and the subsequent review by a clerk hold no relevance in the context of vehicle applications or instances where individuals seek to update their address information on their driving licence or vehicle - wherein no photo submission is required.


{{<figure src="images/ape-and-dape.svg" caption="10,000m view of the integrated driver application workflows" class="center-caption">}}

## Harnessing AWS Step Functions direct integrations
Within the extensive repertoire of direct integrations offered by AWS Step Functions, lies a notable inclusion: AWS Step Functions themselves (yes, start a workflow within a workflow). Leveraging this optimized integration, we seamlessly triggered sub-workflows tailored to the specific phases of driver or vehicle applications directly from our application process engine. This innovative approach draws inspiration from the [ServerlessLand](https://serverlessland.com/workflows/workflow-within-a-workflow-cdk) website, underscoring our commitment to efficiency and best practices in workflow architecture.

AWS simplifies the process of defining steps necessary to trigger sub-workflows. We leverage the [Serverless Framework](https://www.serverless.com/framework/docs) for deploying our applications onto AWS which further enhances this seamless integration. Below is an example of how a sub-workflow is triggered:

```yaml
  CheckReceptionHook:
    Type: Choice
    Choices:
      - Variable: $.hooks.receptionStateMachine
        "IsPresent": true
        Next: ReceptionHook
    Default: SettlePayment

  ReceptionHook:
    Type: Task
    Resource: arn:aws:states:::states:startExecution.sync:2
    Parameters:
      StateMachineArn.$: $.hooks.receptionStateMachine.arn
      Input:
        applicationId.$: $.applicationId
        iteration.$: $.iteration
        AWS_STEP_FUNCTIONS_STARTED_BY_EXECUTION_ID.$: $$.Execution.Id
      Name.$: States.Format('{}-{}-{}', $.hooks.receptionStateMachine.name, $.applicationId, $.iteration)
    Retry:
      # Some retry definitions
    ResultPath: null
    Next: SettlePayment
```

Before delving into the defined steps outlined above, it's crucial to discuss certain preparatory steps. At the outset of processing, the application process engine initiates a pivotal step. Here, logic is deployed to ascertain the application type, delineate the required sub-workflows, and furnish the necessary Amazon Resource Names (ARNs) for their execution.

Within our framework, we employ the term Hooks to denote instances where the application process engine (APE) workflow triggers a sub-workflow. Pertinent hook details are then propagated through the state input and onto the relevant steps.

While each phase of the application possesses the potential to trigger a sub-workflow, not all applications necessitate every phase. Hence, a preliminary step precedes the hook triggering, deciding whether the state contains details corresponding to the specific phase of the application. As evidenced in the above definition, a step named CheckReceptionHook exemplifies this verification process.

This orchestration empowers the sub-workflows to seamlessly adapt to the idiosyncratic demands of each application type, while concurrently maintaining the application process engine's versatility and generic behaviour.

### Advantages of this approach

#### Addressing the challenge
One of the primary hurdles we encounter revolves around the domain expertise of our teams, particularly concerning drivers or vehicles. This approach empowers driver and vehicle teams to craft their specialised sub-workflows tailored to their applications. These teams seamlessly integrate these sub-workflows into the application process engine, thereby effectively fulfilling the application.

#### Streamlined testing
Testing becomes notably simplified with this approach, as clear boundaries are delineated for testing purposes. The application process engine operates as an independent deployment, facilitating comprehensive testing within a controlled environment. Furthermore, the project can undergo testing in isolation. Simple stubs can be readily implemented for the sub-workflows, facilitating smooth progression through various phases. Each individual sub-workflow operates as a self-contained entity, enabling isolated triggering and testing.

#### Efficient utilisation of common functionality
Every application inherits the shared functionalities embedded within the application process engine, obviating the need for duplicated steps. This streamlined approach empowers teams to focus their efforts on refining their unique business logic and workflow, thereby enhancing overall efficiency and productivity.

### Navigating challenges in implementation
#### Designing a robust generic application process engine
Despite our analysis of each application type within the DVLA, designing a comprehensive application process engine presents formidable challenges. Anticipating the impact of our design decisions across diverse application types proves inherently complex. To mitigate this challenge, communication and collaboration are paramount. Ensuring that all relevant stakeholders are engaged and have oversight of the changes helps minimise the risk of unintended consequences.

#### Deploying to an Active Production Environment
Deploying application workflows to a live production environment introduces another layer of complexity. As these workflows actively process applications, any alterations to the underlying sub-workflow state must be executed with utmost caution. Deploying changes while applications are in progress runs the risk of inconsistencies or disruptions in workflow execution. Hence, we attempt to incorporate robust backward compatibility and versioning mechanisms to navigate the evolving landscape effectively.

#### End-to-end testing dilemmas
While the benefits of testing in isolation are undeniable, they also introduce challenges, particularly in the realm of integration testing. Integrating isolated components developed by different teams necessitates rigorous integration testing to ensure seamless interoperability. Effective communication becomes paramount, especially when teams may naturally prioritise their component's delivery over integration testing. At DVLA, we mitigate this challenge by dedicating a specialised team solely focused on integration testing, ensuring comprehensive validation in a fully integrated environment with an independent view and approach.

## Conclusion 
In summary, the DVLA's adoption of AWS Step Functions and the development of the application process engine, represent significant strides towards modernising its application processing. We are thrilled to announce that we have successfully gone live with the ten-year licence renewal process, marking a significant milestone in our digital transformation journey.

Time will tell whether the design decisions result in an overall improvement in migrating and delivering applications onto the new platform.