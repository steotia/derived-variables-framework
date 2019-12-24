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
## Approach
1. Since derivation seems to be a reaction to an event (passage of time, change of property, change of logic), it seems that an event driven architecture seems natural
2. Publishers of event could be crontabs, time tickers, data store change logs, CI pipeline deploy agents
3. Topics to publish into could be Entity specific - Warehouse, Inventory, Products, Workforce, etc
4. Consumers of event could be the derivation functions and they could subscribe to topics as per their dependencies on other entities
5. The extended set of entity properties can be present in a noSQL store with a schema which captures metadata - processor version, processed date
6. Consumer lag and Sink growth are some metrics which can be tracked for anamoly detection of stream processing
## Solution
1. Event sourcing model
