# derived-variables-framework
In a catalog of objects, certain object properties are static and some are derived. What could be a good software design to:
1. incorporate derived properties into a system which was till now handling only static properties
2. keep business and engineering in close loop?
2. ensure freshness and validity of derived properties
## Assumptions
1. an object's derived values' may depend on (a) it's own properties (b) other properties (c) time and any combination.
2. trigger of derivation could be (a) time (b) addition of entity record (b) update of entity properties (c) change of derivation logic (d) on demand/manual
3. derivations could have SLAs - time to compute, cost to compute, cost to develop, etc
4. derivations may be based on small or large volume of data, and can involve simple or complex calculations
5. some derivations may already be happening in offline systems, like xls sheets and it maybe important to accept derivations happening offline
6. derivations could be nested in nature, where changing one property/logic should cascade down to all properties which depend on upstream edits
7. incorrect derivation topologies and runaway derivations should be detectable and cancellable without impacting other derivations
8. the system should be flexible enough to allow writing derivation logic as suited to the problem
9. even though the system sounds complicated, it should be easy to work with - should allow for easy visualisations
10. the system should not be expensive to run and preferably not cost when not deriving
11. the system is built on AWS
12. A business maybe operating in many regions, derivation frequency, logic or scope maybe bounded within that region
13. There are many derived properties and there are many entities which can have derived properties
## Approach
0. Even though the question mentions `Product` variables growth, this is a general solution, where other entities can be onboarded as first class citizens with properties. It is true that data points collaborating together create better business insights, so I have taken the liberty to consider expanding scope at the risk of appearing to over engineer.
1. Since derivation seems to be a reaction to an event (passage of time, change of property, change of logic), it seems that an event driven architecture seems natural
2. Publishers of event could be crontabs, time tickers, data store change logs, CI pipeline deploy agents
3. Topics to publish into could be Entity specific - Warehouse, Inventory, Products, Workforce, etc
4. Consumers of event could be the derivation functions and they could subscribe to topics as per their dependencies on other entities
5. The extended set of entity properties can be present in a noSQL store with a schema which captures metadata - processor version, processed date
6. Consumer lag and Sink growth are some metrics which can be tracked for anamoly detection of stream processing
7. Topics could be isolated by region either as separate cluster or by suitable naming convention (recommended in the beginning to keep operating costs low) to keep processing isolated
8. Idea is to have the engineering structure in place, so that it is easier to build on derivations. 
## Solution
1. Event sourcing pattern for all the derived data
2. Reactive streams - sources of events being entity stores (entity CRUD - store could be a SQL DB or even an S3 folder, in which case CloudTrail/CloudWatch are event sources), a command topic (for intention to trigger), time-ticker (to announce passage of time), deployment telemetry (to know new logic is deployed), weather, traffic data, etc
3. Events as changelog - use Apache Kafka or equivalent for the streaming backbone for checkpointing & replay (Logic changed and you want to reprocess the events to re-derive)
4. Event processors - use Apache Flink or equivalent for writing the functions which encapsulate the business logic. The jobs can subscribe to various topics in which lie the events which are of interest to the job. If there is a need for batch processing with events beyond retention period, events can trigger batch processing logic. If the processing is happening outside the system (xls), derived data can directly be injested via S3
5. Use a schema registry to ensure that events schema are valid, possibly use something like Avro which also helps in efficient serialization. Attach schema to event with versioning.
6. Since we are using a CQRS pattern for deriving data, and the derived data ledger are the various topics, the derived data can be fed into whichever BI tool business prefers, Kibana/Athena/RedShift/Aurora, etc
7. Topic structure:
   - For every Entity Category / Resource (Catalog, Inventory, Logistics, ), have the following topics:
     - Telemetry (CRUD operations)
     - One topic for all derivations OR One topic for each derivation
     - Command (Signal plane)
   - Topic examples:
   `catalog.furniture.telemetry`
   `catalog.command`
   `catalog.furniture.derived.timetoassemble`
   `catalog.furniture.derived.interest`
   `crm.user.derived.loyalty`
   `store.inventory.telemetry`
   `store.inventory.derived.performance`
   `store.inventory.derived.forecast`
   `delivery.telemetry`
   `delivery.derived.skillgap`
   `website.telemetry`
   `website.derived.throughput`
 ## Alternatives considered
 - vs simplicity: SQS/SNS/AWS Step functions and Lambda functions for defining workflows/dataflows which create derived data. This is a good option if the number of derivations are complex and limited. However, if the properties are limited to only Products, I will go with this approach
 - vs operating cost: complete serverless arch via Kinesis/Lambda/SQS and rejected due to steeper learning curve involved. If cost to operate is a concern and time to climb the learning curve is available, then this is a good option
## Closing thought
It is hard to solve this problem without knowing more about the domain. I have worked with reactive systems earlier and love the power of data democratization it allows. Also, since it is reactive and can be real time, the real benefit is available only when there is enough scale, where humans can no longer use intuition to solve the problems of matching demand and supply.
However, running these systems can also be costly, since these run in clusters. Generally, a discussion helps me know if my assumptions are getting ahead of me. However, for the assumptions I have taken, the design I have recommended is good and takes a bit of internal adoption to shine.

