# Rective Microservices

## Preliminary Remarks
The goal is to create a **high-performance** scalable software architecture while balancing out
**maintainability** and **conceptual simplicity**.

### Background
* This document is based on the experiences and learnings of a small yet great scrum team at [StyleLounge](http://jobs.stylelounge.de).
* We migrated a monolithic PHP/Postgres application into asynchronous node.js/TypeScript microservices.
* The product is a meta search for fashion that has to deal with a lot of data but has no checkout.
* The absence of a checkout greatly simplifies data flow.
* Arguably a traditional e-commerce application would require more resources (people and experience with microservices) to do true microservices from day one


### Assumptions about our context

#### Eventual consistency is enough
ACID skaliert nicht (-> TODO)
"In der Realität gibt es kein ACID. Auf mehrere Gebäude verteilte Mitarbeiter, die schriftlich Anträge bearbeiten sind schon nur noch eventually consistent. Und in mancher Stadtverwaltung nicht mal das"

#### Infrastructure as Code (+ CI)
Mit manueller Provisionierung zu arbeitsintensiv (-> terraform, ansible, kubernetes) 

#### Open Mind
Microservices und FP sind keine Regeln für Monolithisches OOP sondern erfordern anderes Denken

"Wenn man sich das erste mal mit funktionaler Programmierung besschäftigt denkt man leicht Funktionen sind ja toll, aber ohne Objekte fehlt doch was, weil man sich mit dem Wissen, das man hat orientiert. Man muss sich erst tief einarbeiten um zu verstehen, dass mit FP bereits die Modellierung anders aussieht, also Fragen anders formuliert werden"


Typing helps enforce purity and idempotence because a lot of JavaScript's magic is being made visible

### Don't take this too seriously

This tries to go beyond the typical "here's who we did it" by extracting theoretical concepts.
If done right this allows for fruitful conceptual dicussions. Mut much can go wrong and one easily ends up with something that attempts to be a concept when it actually is just fancy words describing a status quo. Furthermore much more dicussion and feedback is needed before getting even close to be actual recommendations for people out there. So think a lot about the why's, consider completely different situations that might require different solutions and please: share your feedback!

## Prior Art
* http://www.reactivemanifesto.org

# Building Blocks
## Building Block 1: Pure Data

SOAP -> REST: Plain objects are easier
Language-agnostic, human-readable, easy to validate
What holds on the network layer makes sense inside systems as well.
Full structural input validation is king. Type definitions for compile-time are queen

* It's a common best practice to move business logic out of domain objects to make side-effects
  more explicit and discoverable:
> `StreetMapService.move(Car car, int x, int y)` instead of `Car.move(x, y)`
  `JungleService.executeJump(ape: Ape, target: Tree)` instead of `App.jumpTo(tree: Tree)`

* The main reason that's left to use OOP for domain objects imho is to keep things "clean", i.e. have type hints and checks as well as run-time validation.

* Runtime validation is better off being done based on schema validation because of its declarative approach.

* There are ways of having strict types without the complex mapping of data to objects (possibly with inheritance) and back.
 * JavaScript: `TypeScript`/`flow`
 * Go: struct tags and `go-validator`

> A good point can be made to value **data > objects**
 
🚀 Building a system around pure data makes immutability much easier and rewarding

## Building Block 2: Pure Functions and immutability!

Pure function
A function does not change its surroundings (including the input) but only returns newly computed information //TODO bad, get good definition

-> If the input can be split into chunks all pure functions can run massively parallel

When data is immutable and no constant mapping onto object occurs its quite easy to use pure functions with which it is in turn quite easy to horizontally scale across splittable input data

Et voila: We found a way to easily scale business operations and protect against side-effects (as long as we can split our input into chunks!)

Let's save the problem of splitting input for later ;)

## Building Block 4: Shared State

In OOP you have variables to store data. Services, controllers, commands, ... have references to variables
???Extract key arguments from that one YouTube video

This storage-centric thinking dates back to mainframes. You start with designing your storage layout and then implement the processes around it. That order is at the core of OOP.
data structure > processes

Problem: You end up having multiple processes working with the same data.
This is at the heart any scalability problem.
You need synchronization. Synchronization doesn't scale.
Don't. share. state. Don't. alter. somebody's. state.
This is a direct result of thinking data structure > processes

Today there are myriads of storage techniques and storage capacity is virtually infinite.

In reactive programming you start with the processes and their in/output.

### Alternative Approach to Modeling Software
#### Processes > Data
* model processes
* For each process define its unique input and output

We already learned that processes scale as long as we can split the input.
TODO Picture: Lego Brick from up close

#### Let's Zoom Out
TODO Picture: Structure resembling the block that's build from these blocks

❗️ Let's view a micro service as a high-level process that consumes input and produces output

##### Suggested Properties
* It's not responsible to where that output goes
* It must not be pure:
 * It may maintain state to
  a) speed up processing
  b) Use input from various sources to compute output
* It should be idempotent
  Otherwise additional synchronization is required
* It must consider outside data to be immutable (no external writes)
* It can decide what storage suits it best (if microservices are sufficiently small and have a sufficiently simple way of retrieving data a key/value-store suffices)


## Recap
### Building Block 1: Pure Data
* data > objects
* strict input validation

### Building Block 2: Pure functions and immutability
* minimize side-effects
* Make all data immutable

### Building Block 3: Shared State
* processes > data structures
* each process may maintain state (and decides what storage technology to use)
* Processes should be idempotent


# Putting things together

>	"Let the data flow for it to become a powerful stream"
>	
>	- Confucius

In reality there won't be a single stream of data but multiple sources and even circles.
Each circle adds complexity that one should try hard to avoid.

# Best practices for asynchronous micro services

### A service should
* subscribe to other service's events
* publish events for other services to subscribe to
* be able to rebuild its state from receiving everything but its original state
* be able to re-emit its full state to all subscribers

### A service may
* Maintain its own state
 1. inherent (???vs intrinsic) original state: data of which it is the system's single source of truth
 2. aggregated state: state derived from its subscriptions
* Trigger other services synchronously
  (Try hard to avoid and use subscriptions, risk of a distributed monolith)
  See trigger

### A service should never
* Fetch data from another service (Use subscription for that)
* Alter external state


# What do we get from this?

Let's now point out what one gets from all this

## Structural Scalability

You have to scale by messages, not requests.
Unless a user isn't actually doing something, no or very little messages are emitted from the front end

### 1. Locally maintained state
* All services maintain their entire state
* Therefore they can immediately act upon incoming messages.

### 2. Bulk Operations
* If service A subscribes to data from source B to persist it locally it can use bulk inserts to significantly speed up the process. That's not possible with requests

### 3. Delta Filters
*  Each microservice can use a super simple delta filter to make sure only changed data is emitted. This significantly reduces the load emitted towards subscribers and keeps 

### 4. Laid Back Autoscaling
  - If the demand outnumbers the available resources for any service (i.e. queues fill up) you have minutes, not milliseconds, to auto-scale before any service degradation occurs

### 5. Debounce & Aggregating Messages
  - Scaling usually means serving more users
  - Users create events (e.g. tracking events like "user X clicked product Y in the product list")
  - Most of the services don't need to act upon each of these events
  - It's super easy to aggregate these events and only occasionally emit relevant changes
    Instead of ClickEvent { productId: number; userId: number; timestamp: Date}
    cou can use a cron to emit these every n minutes:
    ProductClickAggregate { productId: number; aggregateStart: Date, aggregateEnd: Date, clicks: number}

> #### Let's super-scale!

> This way even with 10000x the traffic the number of emitted events is bound by the number of products and the interval length.


### Maintainability

#### Deploy with downtime
* In a typical setup an asynchronous microservice may be done for seconds, minutes or even hours because it seriously impacts the business
* Use that to cut down your deployment complexity

#### Readable code

* Pure functions are super easy to test
* So are microservices without side-effects 


### Triggers / Synchronous operations (Actually needed❓)

Imagine services A and B. Typically if information should flow from A to B then B simply subscribes to events emitted by A.
There are (very few!) situations where A wants to trigger B without B having subscribed to A beforehand.

Example: Metering

//TODO are there actually good examples for triggers?

Triggers should use a synchronous protocol such as HTTP to allow for input validation

If service A publishes data to service B using a synchronous protocol such as AMQP service B has no way of doing programmatic input validation and rejecting the data with a descriptive error message.
If e.g. AMQP is used the only thing service B can do is to dead-letter the data or throw entirely dispose of it.

Avoid synchronous triggers because they by definition make a system tightly coupled
