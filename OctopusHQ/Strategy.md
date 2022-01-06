## Summary ##

This document describes the integration strategy for systems that are part of the Octopus HQ (formerly Octofront) ecosystem. 

The purpose of this document is to provide enough guidance to system integrators to increase their chances of falling into the pit of success.

## Goals ##

* Systems are coupled only via published contracts
* Systems can perform their primary function without needing to communicate with/rely on other systems 
* Systems can be evolved and deployed without affecting any of the other systems 
    *  No need to make changes to a group of systems in lockstep


## Implementation ##

The current approach is a mix of [Event Notifications](https://martinfowler.com/articles/201701-event-driven.html#EventNotification) that is used for events representing state changes and [Event-Carried State Transfers](https://martinfowler.com/articles/201701-event-driven.html#Event-carriedStateTransfer) that is used for well-scoped data updates. As long as we avoid `TheWholeThingUpdated` events, this approach, which is not perfect, should be a good starting point.

We might revisit this approach in the future and for example, use Event Notifications as the only type of events and rely on other types of APIs for data transfer (e.g. Data Lake) 

### Good Event Test ###

A good event has the following properties:

* It’s treated as a strict contract that needs to be honoured for a long period of time. 
* It's frequencey can be reasonably well estimated. E.g. 10s/100s/1000s an minute/an hour.
* It’s specific (e.g. `SubscriptionTerminatedV1Event`, `OutageWindowChangedV1Event`) not generic (e.g. `SubscriptionUpdatedV1Event`)
* It’s unlikely to change in the future. E.g. The team is willing to keep publishing it for at least 6 months.
* Its entire payload is mandatory. E.g. All attributes have values assinged to them.
* It only includes data specific to what it represents (e.g. `OutageWindowChangedV1Event` shouldn’t carry subscription limits just in case one of the consumers needs them)
* It is published only when a real change has occurred (e.g. `OutageWindowChangedV1Event` should not be published if Start and End properties have not changed)
* It’s smaller than 64KB
* Its name ends with a version number and `Event` suffix. E.g. `OutageWindowChangedV1Event`

## Infrastructure ##

Octopus HQ uses [Azure Service Bus](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messaging-overview)  (ASB) as the messaging platform. In most cases this fact should not matter but at the same time, it can’t be entirely ignored because ASB is a product that offers a very specific set of features.

There is one ASB instance per environment (e.g. Production, PreProd, Dev-A). In Production and Pre-Prod environments,, ASB is configured to use Premium Tier. 

Systems acting as publishers are required to create an [ASB topic](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-queues-topics-subscriptions) per [bounded context](https://martinfowler.com/bliki/BoundedContext.html) and subscribers are expected to use this rule to find the events they are interested in. Subscriptions need to have unique names and it is recommended that their names start with the name of the system that created them (e.g. `cloudplatform-purchasing`).

## Security ##

As specified, there will be a dedicated topic per bounded context. This should give us enough flexibility to implement effective security controls (e.g. who can send messages to a topic,  who can subscribe to a topic)  in the future. This work hasn't been done yet.

## FAQ ##

* I can’t use transactions. How can I avoid publishing duplicate events?
    * [Azure Service Bus duplicate detection](https://docs.microsoft.com/en-us/azure/service-bus-messaging/duplicate-detection)
* I can get events out of order. How do I deal with this problem?
    * In general, it is up to the subscriber to make sure events are processed in the correct order. That being said, the message bus is structured in such a way that messages in a topic are delivered to subscribers in the order they were sent to the topic. If a subscriber limits the number of processors to 1, which should be more than enough for most subscribers, then events will be processed in the correct order. 
    * There should be no dependency between messages in differnt topics. If a subscriber needs to process events from multiple topics in a specific order then it can either [defer](https://docs.microsoft.com/en-us/azure/service-bus-messaging/message-deferral) out of order messages or it can send them to [a dead-letter queue](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-dead-letter-queues) for future manual or automated processing.
    * Can I subscribe to an event as soon as it is announced? 
        * Yes, as long as the topic where the event will be published already exists. 
    * How do I get historical events/data?
        * This problem doesn't have a good solution yet.
        * Ideas: 
            * Use APIs provided by the system that publishs the events
            * Request a data dump from the system that publishes the event
    * How do I notify the wider ecosystem about changes to the events I publish?
        * Now: Update System Events section of this document and  announce your changes in #topic-octopushq.
        * Future: Some form of contract repository

 
    	
## Octopus Event Specification TO REVIEW ## 

Every event should have the following structure

```json 
{
    "id" : "SubscriptionCreated-licensekey-8a52c8af-2408-486f-ba4a-d277079de2b0",
    "source": "Octofront",
    "type": "SubscriptionCreatedV1",
    "occurredTimeUtc": "2021-11-08T12:33:00Z",
    "activityId": "00-15c3886877ecbd4a95b8b031cd7fbb46-c5fe780fb33bb443-00",
    "specVersion": "1",
    "payload": { }
}
```

| Name        | Type        | Description | Implementation |
| ----------- | ----------- | ----------- | ----------- |
| id          | string      | The unique indentifier | This property can be used to take advantage of [the message deduplication feature offered by Azure Service Bus](https://docs.microsoft.com/en-us/azure/service-bus-messaging/duplicate-detection). At the minimum the identifier needs to be unique within the topic it is used.
 |
| source   | string        | The name of the source system | |
| type   | string        | The type of the event | |
| occurredTimeUtc   | string        | An ISO 8601 formatted string representing the data and time of when the event occured  | |
| activityId   | string        | An identifier that identifies the overall activity in traces | |
| specVersion   | string        | The version of the Octopus Events spec this message adheres to | |
| payload   | json        | The payload of the event itself, with all it’s relevant data | |

