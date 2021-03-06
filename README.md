# Introduction

The Pi project was an effort to create a massively scalable application substrate for the development of Cloud and other applications and services.

A fully functional, API-compatible clones of Amazon's EC2 and S3 services were created and released in a 'private cloud' scenario.

# Status

Strategic realignment and organisational seismics eventually put a stop to the development effort in 2011. Although Pi remains in use, it is effectively on life support.

The codebase was open-sourced in 2012 in an effort to extend its lifespan, as well as to preserve and share the experiences and learnings from developing a platform with the particular set of design goals.

# Can I use Pi?

The software has been in production use since 2011, however given its status you are advised to look at currently active projects such as OpenStack.

At this stage, the open-sourced software stack is most likely to be useful as a reference for anyone with an interest in or looking to build highly scalable infrastructure software services.

## The What and the Why
Pi is a software framework for the creation of highly scalable, highly robust software applications and services.

Many products claim to offer these capabilities, usually ‘magically’ - 'just write your app, and we'll make it scale'. Pi is different in that it explicitly defines its own choice of tradeoffs needed to achieve scale and resilience, recommends patterns for implementing these tradeoffs, and provides APIs to facilitate and guide that implementation.

Pi Cloud is the first set of services developed on Pi. It provides de-facto standard, Amazon-compatible infrastructure-as-a-service (IaaS) APIs to compute and storage services.

The creation of Pi has been driven by the need to provide an IaaS platform to underpin BT’s cloud offering. This meant building a distributed system with a large number of nodes from the outset, as each node would only support a finite number of virtual machine instances, finite storage, and finite bandwidth. As an infrastructure platform, it also placed demanding requirements on the resilience.

## Anatomy of Pi

Pi has a distributed, decentralized architecture, as a way of achieving the necessary scale. Pi nodes – each with its own unique identifier - form a ring of peers (a structured p2p overlay), with each node tracking a set of peers (its ‘leaf set’). This organisation is very powerful in that it allows one node to route a message to another via a bounded number of intermediate nodes; thus by mapping resources (such as pieces of data or services) to identifiers, nodes can locate and access those resources without having to have specific knowledge (addresses, routing etc) of how to access them.

The structured p2p overlay is currently provided by the Pastry library, which organizes nodes into a ring, provides message routing and delivery, and exposes message-related and node-related events to p2p applications.

Above Pastry sits Pi Core, the p2p application framework that provides developer-friendly APIs for application creation, messaging, data access and management, and much more. It also abstracts Pastry away.

Pi Cloud services are implemented as a series of Pi applications, starting with a common app that defines the shared data entities, followed by applications for network, volume, image, storage and management, and the API application which exposes the APIs. These applications are then packaged together to form a deployable artifact – in the case of Pi Cloud, a RPM package. This artifact is generally deployed on each node, and is run as a single process instance (JVM).

Pi is written in Java, utilising the Spring Framework. There are certain platform-specific aspects, largely around networking for machine instances. These currently work with CentOS / RedHat Linux.

For Cloud, Pi is deployed on Rocks Clusters, and uses Parascale shared storage.

# Developing Pi applications

Developers familiar with prevalent server-side transaction processing programming models – particularly from the Java community – will face several core concepts that may be unfamiliar, or familiar in a different context. Here are some of the more prominent ones:

* Processing paradigm - It may be helpful to think of Pi more as a workflow-type system rather than a request-response type system. Pi uses message passing, however this only offers best-effort delivery guarantees. Hence operations are typically handled through a series of steps, each of which can be monitored to ensure successful completion and handoff to the next step. Request-response type interactions are of course possible in this context, and Pi provides correlation identifiers and relevant helper APIs.
* Asynchrony - With a few notable exceptions, all operations within Pi are asynchronous. Methods that return a value do so by calling back an anonymous method (typically implemented in Java as an anonymous instance of an interface defining that callback function). Most IO events are currently delivered to Pi from a single worker thread from Pastry – for these events, it is essential that they are processed rapidly, without blocking, This programming model isn’t unlike languages such as Javascript and Actionscript. In Pi, non-trivial processing tasks need to be done in threads other than the thread of the received event.
* Data - Pi provides a distributed key-value data store, in the shape of a distributed hash table (DHT), and APIs to access it. There are no locks and no transactions. Concurrenncy is managed using an optimistic model – we read a record, update it, write it; if another write happened between us reading the record and writing it, we repeat the read-update-write process.
This provides massive scalability. Tradeoffs are around data consistency and expressiveness of modeling data relationships.
A set of Pi applications (forming one or more services) will typically share a data model. It is important to ensure that Pi data entities are robust in the presence of disparate versions of Pi applications across nodes. For this reason, and for ease of binding to Java objects, Pi uses JSON as its data interchange format of choice. Pi applications then define entities as Java objects that implement the PiEntity interface (or extend the PiEntityBase class), and use Jackson framework annotations where non-trivial binding is required.
* Concurrency - Pi has no locks and no transactions. For managing concurrency around single-record updates, optimistic concurrencsy is used as described above. For updates across multiple records, each record is updated separately. Failure is handled through a combination of compensation, watching (see the watcher pattern) and inconsistency resolving logic, as appropriate.
* Idempotence - Operations in Pi should in general be idempotent. For instance, allocation of a resource to a consumer should succeed if that resource has already been allocated to that consumer. This makes applications more robust and enables recovery mechanisms based on re-running failing tasks.
* Polling and eventing - Pi applications often use both polling and eventing around a particular task as a means of achieving improved resilience and failure recovery. Usually eventing is used to ensure tasks are processed eagerly, while polling is used as a fallback, in the event of failure.
* Application coupling - P2P application design differs somewhat from traditional SOA or client server design. P2p apps can be tightly coupled in the sense of sharing knowledge of and responsibility for a resource or a piece of data. They should be thought of as components of a single system. However, good practices around the design of local versus remote interfaces and interactions continue to apply, as does the need to ensure robustness against having disparate versions of each application deployed on each node.

# Patterns of Pi

## Ring creation and initialisation

### Ring initiator
(How does a node start a new Pi ring?)

To join a ring, a Pi node needs to contact a node that is already in that ring. This allows it to bootstrap itself, discover its place in the ring (based on its ID), and populate its initial leafset.

Pi currently supports peer discovery based on a list of hostnames or IP addresses. This is a configuration parameter. In time, Pi will likely support broadcast and other mechanisms too.

When started, all nodes will attempt to join an existing Pi ring, by using one or more available peer discovery mechanisms (see below). If this fails or times out, a node will by default not start a new ring.

In order to create a new ring, a configuration flag that normally prevents nodes from starting new rings needs to be disabled. This allows a node to start a new ring. Once a node has started a new ring, other nodes can bootstrap off of it, inheriting its region and availability zone unless configured otherwise.

### Seeder
(How does a DHT get initialised with required data?)

Certain applications will require presence of one or more data items in the DHT in order to function. Although it is possible to have applications create required data lazily if attempts to read them fail, this makes it difficult to distinguish between data that becomes inaccessible due to a fault, and runs the risk of creating multiple copies of the same entity.

Some examples of data items that may need seeding include entities used to determine whether or not an application should be active on a certain node, records used for resource allocation, task queue records, etc.

A seeder is simply a service that allows such records to be created in the DHT when a new ring has been started. It usually performs a series of DHT put operations, using configuration data specified via the service interface.

In addition, a seeder may be used to initialise a new subset of nodes within an existing ring (for instance to create configuration for a specific environment or availability zone, for use by nodes within that environment), or to alter the settings of a previously seeded record.

# Applications

## Pi application
(How do I create a Pi application?)

Simply extend the appropriate Pi application base class. You can start with the default pi application base class, or use one of its extensions if additional functionality is required - for instance:

* pub/sub support (anycast / broadcast sending and receiving)
* external access to one or more services (assigning an address from a pool of addresses to active applications, and opening / closing access via iptables)
etc.

Then define application activation strategy (see below) and implement handlers for all or some of the following:

* application startup / shutdown
* message delivery and forward
* pub/sub events, such as receipt of anycast or broadcast messages
* application activation / passivation (becomeActive / becomePassive)
* handling of leafset updates (in effect node arrival / departure events)

Make your application a Spring bean, and have Spring inject required dependencies.

## Application Activator
(How can I have an application be active on a subset of nodes in the p2p network?)

Some applications will want to run only on a subset of nodes. Examples might include applications that expose a service on one of N IP addresses, applications that perform some heavy duty processing, or those that aggregate information.

An Application Activator provides a strategy that is used to determine whether or not an application should be active on a particular node. The meaning of 'active' and 'passive' are entirely application specific - indeed it is not uncommon in Pi for 'passive' applications to still perform some basic functions like routing of messages that pass through them.

Pi comes with three activators:

* the simplest is the always-on activator, which basically makes all application instances active
* the shared DHT record activator can be used to specify activation of N applications, each optionally acquiring one of N resources upon activation. Activation state is stored in a DHT record which is polled by applications, hence care must be taken to set polling frequency according to the overall number of candidate nodes
* the supernode activator will activate one of N applications. It differs from the shared DHT record activator in that activation is determined by whether or not a node is nearest to one or more of a set of predetermined points in node id space (in other words, whether a node's id is closest to one of a set of IDs determined by a function stored in the DHT).

It is additionally possible to specify basic exclusion rules between applications, such that an application is prevented from becoming active if some other specific application is already active. This may be desirable if, for instance, two applications may conflict in some way if run on the same node, or place high load on compute, network and / or storage resources on that node.

### Exposed service
(JHow can I expose a service to external consumers?)

Pi can automatically manage the exposure of an application via a set of IP addresses, across a range of nodes. This is achieved by extending the base class that provides managed addressing capabilities for a Pi application, selecting the shared DHT record application activator, and seeding the relevant application record with appropriate addresses in the DHT.

# Data

## Distributed hash table (DHT)
(How do I store and retrieve persistent data in Pi?)

Pi provides an implementation of a DHT, a distributed key-value store for data storage. Currently this is optimised for storage and retrieval of relatively 'small' data records. Support fro append operations on records, with a view to gathering large amounts of data for later indexing or processing, is coming soon.

Pi DHT provides the following operations:

* put, which replaces an existing record with a new one, just like a traditional hashtable put operation
* get, which, given a key, reads and returns the record stored against it - again this is analogous to the get operation on a hashtable
* update, which reads a record, delegates its update to the application, then writes the updated record using optimistic concurrency management. This means that the logic that updates an existing record may be invoked more than once, typically if a different update, perhaps from a different application or node, occurred between the original record being read and the updated record being written.

There is no delete operation due to the replicated and dynamic nature of a DHT. Applications can however flag records as deleted and treat them as such.

DHT APIs come in two flavours - asynchronous and synchronous (blocking). Asynchronous should be used wherever possible. Blocking DHT operations should only be used in specific circumstances, where there is 100% certainty that they will never be invoked from the Pi / Pastry event delivery worked thread (aka 'selector thread'). Examples are DHT operations that are invoked from a synchronous servlet, in the thread of a web server, as well as seeding operations invoked through a synchronous API.

DHT records are stored on the node whose ID is the nearest to the ID of the key. Records being written are typically replicated to one or more nodes adjacent to the nearest-ID node. How many replicas are written in total, and how many replaces have to be written in order for a write to be deemed successful, are configuration parameters of Pi that in effect express a tradeoff between write availability and consistency.

Data records in the DHT have a set of HTTP-like headers that represent metadata describing the record's ID, its URI (and thus, via the scheme, the type of the stored resource), content type, version and so forth.

In addition to using remote DHT APIs, each node can also 'scan' DHT data stored locally on that node. This is very useful when data with certain properties - perhaps a record that has expired in some way - needs to be found and acted upon in some way. It can also be used to index local resources of a certain type, and publish the result to some set of supernodes to provide runtime querying, search or reporting facilities. These local DHT access APIs should NOT however be used to alter or update data.

## Entity identifiers and modelling
(How do I model, identify and locate domain-specific entities stored in Pi?)

Pi mandates the use of a common data model for application entities - clearly there is a need for applications on different nodes to be able to communicate, share data, and understand that data. Furthermore, in general a large P2P network may consist of different versions of applications, hence applications have to be robust for this, and in some cases code for multiple versions of a given entity.

To support these requirements , Pi entities are modelled as Java objects that implement a lightweight interface (PiEntity), serialised and stored as JSON. This provides a structured data format, without imposing a rigid schema. Pi uses the Jackson library for binding Java objects to JSON, and Jackson annotations can be used with Pi entities to provide custom bindings if required.

Pi treats all instances of entities as resources, each with its own URI. The scheme of the URI represents the type of the resource.

A Pi / Pastry ID for a resource - typically corresponding to its DHT key - can be determined from the combination of the URI and the scope of that resource (availability zone, region or global). Pi provides an id generator that will return the correct ID given this scope and identifier information,

## Searching and indexing
(How do I aggregate data, or find data matching certain criteria?)

Due to its decetralised nature, and support for basic data access operations only, search and reporting can be a challenge.

For finite-size data sets, where the size allows it, a single DHT record can be used, and can simply be searched or processed to generate reports.

Taking this a step further, a finite, known set of DHT records can be used to store data, and these can be indexed periodically to create / update a set of index records that can be used for searches. There will be a latency associated with indexed data.

For queries that are known at development time, a search through records on the local node can be used, with the discovery of a matching record triggering some action.

Where the query is not known until runtime, local DHT store can be examined, and data on any records of interest published to a set of supernodes for aggregation and querying.

Finally, where latency is not an issue, DHT records can be batch-processed for reporting, indexing and other purposes.

## Caching
(How do I ensure DHT lookup results are reused?)

DHT reads and writes can be expensive, hence caching becomes important early on. Pi provides blocking and non-blocking DHT cache access APIs, with both read-through and write-through capabilities. These should be used with the same constraints already described around the use of blocking and non-blocking DHT access APIs, with non-blocking preferred wherever possible.

Pi's cache implementation is backed by Ehcache.

# Messaging

## Message passing
(How do I send a message from one node to another?)

Pi makes it easy to send messages from one node to another. All that is required is a destination ID, which can be computed from a resource as described in the Data section. The message will then be delivered to the node whose node ID is the nearest to this destination ID.

Pi messages typically consist of a destination ID, method (a CRUD verb) and a Pi entity being sent. Optionally, a message can be sent to an application that is different from the sending application. Messages additionally contain elements that are generally not visible to the application, above all a unique message identifier, a correlation identifier for matching responses with requests (where applicable), and a transaction ID used to group related messages together.

This structure can be altered or replaced if required.

Messages are sent through a message context. A new message context can be created trivially from a message context factory (an interface implemented by all Pi applications), or a message context of a received message can be used to send a new message. The latter approach has the advantage of keeping a consistent transaction ID between messages - thus related log messages across a series of nodes can easily be identified.

Note that Pi messaging is inherently unreliable. Although nodes do have a level of routing logic that improves robustness under node churn, sent messages are not guaranteed to arrive at their destination. Other mechanisms need to be used to ensure reliability.

## Publish-subscribe
(How can a set of nodes advertise a service or interest?)

Pi supports the standard publish-subscribe paradigm. By leveraging properties of the p2p overlay, it is entirely decentralised and able to support them in a massively scalable manner. Topics are mapped to Pi IDs. Subscribers to a topic join that topic by adding themselves to a tree of subscribers, rooted at the node closest to the topic ID. Nodes wishing to publish to a topic do so by sending a message toward the topic ID; this is intercepted and handled en route by the first node to subscribed to that topic. If there are no subscribers, the message terminates at the topic ID node, where the 'no subscribers' condition can be handled.

All of this is abstacted from Pi applications, which only have to worry about declaring that they wish to subscribe to or unsubscribe from a topic, and optionally handling the 'no subscribers' case.

## Anycast
(How do I send a message to one of a series of interested recipients?)

Anycast involves sending a message to a topic, such that the message is processed sequentially, one node at a time. A node that receives an anycast message may process it without forwarding it onto other topic subscribers, alter and / or process it partially before forwarding, or forward it on without processing.

Anycast messages can be sent 'directly' or via a randomly chosen intermediate node. The latter is useful when wanting to ensure that anycast messages sent from a particular node do not traverse the topic tree the same way, providing better distribution of messages across subscribers.

Among other uses, anycast provides an effective, decentralised mechanism for finding nodes that can handle a certain request or supply a certain resource.

## Broadcast
(How do I send a message to a group of interested recipients?)

An application can broadcast a message to a topic. Subscribers to a given topic will distribute the message to all known peers that have also subscribed to the topic. The message is in effect distributed throughout the topic node tree in parallel, making broadcast efficient.

# Resource management

## Consumed resource registry
(How do I track the usage of a resource that may have multiple consumers and / or require co-ordination across nodes?)

Certain applications need to track usage of resources such that an application instance running on a particular node knows when a resource needs to be created or destroyed. This becomes particularly important when that resource is scarce, or when duplicate instances of the same resource are problematic.

Pi provides a simple 'consumed resource registry' component, which allow applications to track how many consumers a particular resource has, and thus determine when a resource needs to be created (first consumer registered) or removed (last consumer deregistered).

There is also an extended resource registry implementation, a 'cached consumed dht resource registry'. This is appropriate when the resource additionally has a DHT representation. Upon registration, the DHT record is read and stored in memory; upon deregistration, it is removed. Furthermore, it is possible to specify a resource refresh strategy. This enables the triggering of a periodic refresh of the resource through reloading the DHT record, and allows specific actions to be performed upon refresh. Additionally, each consumer can be 'watched', periodically triggering specific application logic that can determine whether a particular consumer still exists.

Cached resource records can be retrieved on demand, as can consumers for a specific resource, and a list of resources of a certain type. This is essential when applications need to track state in memory, avoiding the need for frequent DHT operations. Application design needs to be such that a particular resource only ever has one 'owner' node - this is easily achieved by allocating ownership based to node ID proximity to the ID of the resource.

## Subscriber resource allocation
(How do I allocate resources available across a set of nodes?)

A highly scalable, decentralised scheme for resource allocation involves nodes that possess the given resource advertising its availability by subscribing to a topic. A client wishing to utilise that resource sends an anycast message to the topic. This is handled by one or more recipient nodes, which allocate the resource as appropriate.

# Fault tolerance

## Task processing queues
(How do I capture a task or operation so that it can be retried if unsuccessful?)

Pi provides a means of storing details of pending tasks which can be carried out in an idempotent fashion, and hence retired if a previous attempt to execute had an unsuccessful or unknown outcome.

A Pi application can define one or more task processing queues, usually through seeding. When a particular task needs to be performed, it is added to the appropriate queue. Instances of that same application monitor the queue for any tasks that have become stale, i.e. can be retried. A maximum number of retries can be set.

If the initial (or subsequent) execution of a task succeeds, the corresponding queue item is removed. If, however, it fails, the polling mechanism will ensure it gets retried.

## Watchers / supervisors
(How do I detect and resolve anomalies resulting from failures or malfunctions?)

Pi applications can implement watcher components to monitor and rectify issues - anything from misconfiguration of the local node to data inconsistencies to consumer state monitoring. Pi provides a watcher service, which allows application-specific watchers to be registered for periodic execution. The granularity and frequency of each watcher can be whatever is appropriate for each application.

## Eager node failure recovery
(How do I detect and act on a node failure in a timely manner?)

Pi delivers node arrival and departure events to applications. Among other things, this can be used to detect node failure, reported as a departure event, and trigger appropriate action. This can mean eagerly activating an existing recovery mechanism, for instance forcing application activation checks, picking up a task from a task queue, or sending an anycast message to a set of subscribers capable of taking over from the departed node.

