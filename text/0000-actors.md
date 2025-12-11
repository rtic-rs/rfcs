- Feature Name: actors
- Start Date: 2021-09-10
- RFC PR: [rtic-rs/rfcs#0052](https://github.com/rtic-rs/rfcs/pull/0052)
<!-- - RTIC Issue: [rtic-rs/cortex-m-rtic#0000](https://github.com/rtic-rs/cortex-m-rtic/issues/0000) -->

# Summary
[summary]: #summary

This document proposes adding first class support for "actors" to RTIC to allow building RTIC applications in a way that's more modular, easier to reason about and easier to test.

# Motivation
[motivation]: #motivation

The actor programming model encourages separation of concerns by forbidding data / memory sharing between actors.
An actor has its own state which only it can modify.
The only way to interact with actors is via message passing.

The actor model also decouples "business logic" from the actual execution context:
actors can be executed on a time-sliced thread runtime ([actix] and [riker] are examples of this) or
on a priority based scheduler like RTIC.
The actor itself is unaware of the execution context it runs on.

[actix]: https://github.com/actix/actix
[riker]: https://riker.rs/

This proposal has a particular focus on modularity and testability.
It makes it possible to move business logic into separate crates that can be tested on the "host" (build machine).

# Detailed design
[design]: #detailed-design

RTIC already has all the primitives for implementing an actor runtime.
The focus of the design will be making the actors themselves easy to test *outside* the RTIC runtime.
This design chapter is further divided in 3 sections:

- [the actor API](#the-actor-api)
- [the RTIC runtime](#the-rtic-runtime), or RTIC as an actor runtime
- [extensions](#extensions) to the core actor API

## The actor API

There are two core operations associated to actors.

### `Receive`

An actor can *receive* a message using the `Receive` interface (trait).

``` rust
pub trait Receive<M> {
   fn receive(&mut self, message: M);
}
```

Upon receiving a message, the actor will perform some action and be able to modify its own state, hence the `&mut self` parameter in the method.

``` rust
pub struct Actor {
   state: State,
   // ..
}

impl Receive<Message> {
    fn receive(&mut self, message: Message) {
        self.state.modify();
        // ..
    }
}
```

An actor can receive different *types* of message so the trait can be implemented several times for the same actor type.

``` rust
struct ErrorMessage(&'static str);
struct InfoMessage(&'static str);

struct LoggerActor { /* .. */ }

impl Receive<ErrorMessage> for LoggerActor { /* .. */ }
impl Receive<InfoMessage> for LoggerActor { /* .. */ }
```

### `Post`

The other action an actor can perform is *posting* a message to other actors using the `Post` interface (trait).

``` rust
pub trait Post<M> {
    fn post(&mut self, message: M) -> Result<(), M>;
}
```

There are two main things to note here:

1. An actor will *not directly implement* the `Post` interface.
Instead it will be provided a "handle", that implements the trait, from a *runtime*, like RTIC.
It will be the job of the runtime to deliver the message to other actors.
An actor itself is not concerned with *how* the message is delivered; it simply specifies which types of messages it can post using trait bounds (see [next subsection](#unit-testing))

2. The `post` interface does not specify a recipient.
This lets one focus on testing the `receive` "business" logic at the *unit test* level.
The "routing" between actors itself is tested at the *integration* level, with Hardware In the Loop tests for example.
This design choice also gives runtimes the flexibility of being able to provide point-to-point communication; publish-subscribe communication; or both.

The `post` operation is fallible: if the message cannot be delivered, the `post` call will return the `Err`or variant containing the original message.

### Defining an actor

Actors are defined as data structures that implement the `Receive` interface and hold an "outbox" handle that implements the `Post` trait.

``` rust
pub struct Actor<O>
where
    // message *types* this actor can POST
    O: Post<MessageA> + Post<MessageB>
{
   outbox: O
   state: State,
}

// message *types* this actor can RECEIVE
impl<O> Receive<MessageC> Actor<O>
where // ..
{ /* .. */ }

impl<O> Receive<MessageD> Actor<O>
where // ..
{ /* .. */ }
```

The main take-away here is that actors are *not* hard-coded to any particular runtime, like RTIC, thanks to the generic `outbox` field.
Thus actors can be defined in `no_std` *libraries* which can be compiled for the "host" (build machine) ; this allows unit testing on the host (see next section).

The constructor of an actor will take the generic `outbox` as a parameter.
This lets the actor be used with different runtimes.
Reminder: the runtime will provide the `outbox` handle (`Post` implementations).

``` rust
impl<O> Actor<O>
where
    O: Post<MessageA> + Post<MessageB>,
{
    pub fn new(outbox: O) -> Self {
        Self { outbox, state: State::new() }
    }
}
```

## Unit testing

To support unit testing actors on the "host" a `PostSpy` API will be provided.

### `PostSpy`

As unit testing will performed on a development machine, and not a microcontroller, the `PostSpy` API will depend on the standard library (`std`).

``` rust
#[derive(Default)]
pub struct PostSpy { /* .. */ }

// `M: Any` is equivalent to `M: 'static`
impl<M: Any> Post<M> for PostSpy { /* .. */ }

impl PostSpy {
    pub fn posted_messages<M: Any>(&self) -> impl Iterator<Item = M> {
        // ..
    }
}
```

Almost any message type can be `post`-ed using a `PostSpy`;
the exception are messages that contain *non*-static references.

- Primitives, e.g. `i32`, and raw pointers, e.g. `*const u8`, are OK.
- Statically allocated values, like string literals (`&'static str`), are OK.
- Bytes slices that point into the stack, `&'a [u8]`, are *not* OK.

This `M: 'static` bound is also required by RTIC's message passing API so it should not be a limitation in practice.

The `posted_messages` API allows inspecting any message previously `post`-ed through a `PostSpy`.
The message type needs to be specified when calling this method;
this needs to be done using the turbo-fish syntax when there's not enough type information to infer the message type.

### Example unit test

In this example, we have a temperature monitor actor that can post temperature `Alert`s.
The monitor acts on incoming temperature `Reading` messages.

``` rust
// library crate: temperature
#![no_std]

// Actor
pub struct Monitor<O>
where
    O: Post<Alert>
{
   outbox: O,
   threshold: f32,
}

// Messages
pub struct Reading(pub f32);

// Business logic
impl<O> Receive<Reading> for Monitor<O>
where // ..
{ /* .. */ }

// Configuration (hard-coded in this example for simplicity)
const THRESHOLD: f32 = 30.0;

// Constructor
impl<O> Receive<Reading> for Monitor<O>
where // ..
{
    pub fn new(outbox: O) -> Self {
        Self { outbox, threshold: THRESHOLD }
    }
}
```

The desired behavior is that the monitor posts an alert when it observes a temperature above the `threshold`.
Unit tests would verify that behavior using the `PostSpy`.
These unit tests would run on the host and have access to the standard library.

``` rust
#[test]
fn when_temperature_above_threshold_it_posts_alert_once() {
    let mut monitor = Monitor::new(PostSpy::default());
    monitor.receive(Reading(100.)); // manually post a message into the actor
    let outbox = monitor.outbox; // inspect the outbox
    assert_eq!(1, spy.posted_messages::<Alert>().count());
}

#[test]
fn when_temperature_below_threshold_it_does_not_post_an_alert() {
    let mut monitor = Monitor::new(PostSpy::default());
    monitor.receive(Reading(0.));
    let outbox = monitor.outbox;
    assert_eq!(0, outbox.posted_messages::<Alert>().count());
}
```

## The RTIC runtime

An actor runtime needs to:

- Provide `Post` implementations
- Provide an API to instantiate actors
- Provide an API to establish "routes" between actors
- Provide an execution context which defines the order in which messages will be dispatched and whether actors can operate in parallel or concurrently

Furthermore, an actor runtime SHALL only interact with actors using the `Receive` interface and
Actors SHALL only interact with the runtime using the `Post` interface.

As the RTIC framework includes a priority-based scheduler and does not depend on a dynamic memory allocator, in addition to the above it will provide an API to specify priorities and capacities.

### `Poster` handle

The RTIC framework will provide an *application-specific* `Poster` type as part of the context of the `#[init]` function.

``` rust
#[init]
fn init(cx: init::Context) -> /* .. */ {
    let poster: init::Poster = cx.poster;
    // ..
}
```

The `Poster` handler will be `Copy`-able and implement the `Post` trait for all message types involved in the application (see the [routing subsection](#routing)).

### Instantiation

All actors are declared in a struct annotated with the `#[actors]` attribute.

``` rust
// simplified
#[actors]
struct Actors {
    alert_handler: AlertHandler,
    temperature_monitor: temperature::Monitor<init::Poster>,
    //                                        ^^^^^^^^^^^^ concrete `Post` handle
    temperature_sensor: hal::TemperatureSensor<init::Poster>,
}
```

It's possible to have 2, or more, actor instances of the *same* type.

``` rust
#[actors]
struct Actors {
    a: Actor,
    b: Actor,
    c: Actor,
}
```

### Initialization

Runtime initialization of actors happens in the `#[init]` function.
When the `#[actors]` struct is defined, the return type of the `#[init]` function changes to include the `#[actors]` struct.
The values in this `#[actors]` struct will be used to initialize the declared actors.

Following the previous example:

``` rust
#[init]
fn init(cx: init::Context) -> (Shared, Local, init::Monotonics, Actors) {
    //                                                          ^^^^^^ extra tuple element
    let poster: init::Poster = cx.poster;
    let temperature_monitor = temperature::Monitor::new(poster);

    (
        Shared { /* .. */ },
        Local { /* .. */ },
        init::Monotonics(/* .. */),
        Actors { // extra tuple element
            temperature_monitor,
            // .. other actors ..
        }
    )
}
```

### Routing

Within the `#[actors]` struct, the field-level `#[subscribe]` attribute is used to specify which actor will receive which message *type*.

``` rust
#[actors]
struct Actors {
    #[subscribe(temperature::Alert)]
    alert_handler: AlertHandler,

    #[subscribe(temperature::Reading)]
    temperature_monitor: temperature::Monitor<init::Poster>,

    #[subscribe(ReadTemperature)]
    temperature_sensor: hal::TemperatureSensor<init::Poster>,
}
```

An actor can only be subscribed to a message type it implements the `Receive` trait for.
This requirement is checked at compile time.

### Prioritization

Within the `#[actors]` struct, the field-level `#[priority]` attribute is used to specify the priority of an actor.
All the actor logic, both public `receive` API and private API, will be executed at the same priority level.
If the `#[priority]` field is omitted, the lowest priority level, `1`, will be used.
The priority values specified in the `#[actors]` struct are equivalent to the values used in `#[task]`s.

``` rust
#[actors]
struct Actors {
    #[subscribe(temperature::Alert)]
    // #[priority = 1] // implicit
    alert_handler: AlertHandler,

    #[subscribe(temperature::Reading)]
    #[priority = 2]
    temperature_monitor: temperature::Monitor<init::Poster>,

    #[subscribe(ReadTemperature)]
    #[priority = 3]
    temperature_sensor: hal::TemperatureSensor<init::Poster>,
}
```

### Capacity

Each actor subscription is backed by a fixed-capacity message queue.
The capacity of this queue can be specified in the `#[subscribe]` attribute using the `capacity` argument.
When the `capacity` argument is not specified, the capacity defaults to the value of `1`.

``` rust
#[actors]
struct Actors {
    // each subscription has a separate capacity
    #[subscribe(MessageA, capacity = 2)]
    #[subscribe(MessageB, capacity = 3)]
    actor: Actor,
}
```

### Interaction with tasks

The only way software and hardware tasks can interact with actors is by posting messages to them using the `Poster` type.
The `Poster` value can be stored in a resource and made available to tasks.

``` rust
#[local]
struct Local {
    poster: init::Poster,
}

#[init]
fn init(cx: init::Context) -> /* .. */ {
    let poster = cx.poster;
    let local = Local { poster };
    // ..
}

#[task(binds = TIMER, local = [poster])]
fn periodic(mut cx: periodic::Context) {
    // tell temperature sensor actor to read temperature
    cx.poster.post(ReadTemperature).ok();
}
```

In general, hardware tasks triggered by external events serve as "inputs" to the actor network.
Whereas "terminal" actors that post no messages, like `AlertHandler` in the running example, are usually the "actuators" of the system / application.

### Publish-subscribe

When more than one actor is subscribed to the *same* message type,
the RTIC framework will perform a *publish* operation and send a `clone` of the message to *each* subscriber.

``` rust
#[derive(Clone)]
struct BroadcastMessage { /* .. */ }

#[actors]
struct Actors {
    #[subscribe(BroadcastMessage)]
    listener1: Listener,

    #[subscribe(BroadcastMessage)]
    listener2: Listener,
}
```

Only in this case, the message type needs to implement the `Clone` trait.
This requirement is checked at compile time.

When there's only one subscriber per message type, one effectively has point-to-point communication with / between actors.

The publish operation happens *atomically*: posted messages are only dispatched (`receive`-d) after all outstanding messages has been posted.

In a publish operation, if *any* of the receiver's queues does not have enough space for the message then
the `post` calls return an `Err`or and *no* message is posted or cloned.

## Extensions

This section covers extensions to the API. They are *not* required to fulfill the actor model but are "nice to have".

### Field-level `#[init]` attribute

This extension allows initialization of actors in the `#[actors]` struct itself.
The argument of the field-level `#[init]` attribute is used as the initial state of an actor.

``` rust
#[actors]
struct Actors {
    #[init(Actor::new())]
    actor: Actor,
}
```

The argument can be omitted: this results in the actor being initialized using the `Default` trait.

``` rust
#[derive(Default)]
struct Actor { /* .. */ }

#[actors]
struct Actors {
    #[init]
    actor: Actor,
}
```

### Opting out of memory sharing

"No memory sharing" is a core aspect of the actor model.
To enforce that aspect in the rest of the RTIC API,
this extension proposes adding a `forbid` key-value argument to the `#[rtic::app]` attribute.
The value of the argument is a list of features to disable; one of them being `shared_resources`.

Disabling the `shared_resources` feature makes it a compiler error to add any resource to the `#[shared]` struct.

``` rust
#[rtic::app(forbid = [shared_resources])]
mod app {
    #[shared]
    struct Resources {
        resource: Resource,
        //~^ error: `shared_resources` have been disabled
    }
}
```

### Opting out of software tasks

Actors share many similarities with software tasks but are a "higher-level", or more encapsulated, construct so
mixing actors and software tasks in the same application can be seen as working at the wrong abstraction layer.

This extension proposes a mechanism to disable software tasks.
The surface syntax reuses the `forbid` key-value argument proposed in the previous subsection.

Disabling the `software_tasks` feature makes it a compiler error to declare any software task.

``` rust
#[rtic::app(forbid = [software_tasks])]
mod app {
    #[task]
    fn software(cx: software::Context) { /* .. */ } //~ error

    #[task(binds = TIMER)]
    fn hardware(cx: hardware::Context) { /* .. */ } //~ OK
}
```

Hardware tasks are still allowed when `forbid = [software_tasks]` is used.
Hardware tasks act as inputs to the actor network so most "real world" actor-based applications will use them alongside the `#[actors]` feature.

### Memory watermarks

This extension proposes adding an API to retrieve the highest observed queue "occupancy": a *memory watermark* API.
The goal is to be able to measure the actual memory usage of the message queues internally used by actors.

> Side note: although the memory watermark feature is presented in the context of this actor proposal, a similar API could be added to regular software tasks.

#### `Subscription` API

This API lets the application author query information about the subscription of an actor.
It's presented as a struct with public fields and methods.

``` rust
/// An actor subscription
pub struct Subscription {
    /// The maximum number of messages that *can* be buffered in
    /// the internal message queue
    pub capacity: u8,

    /// The type of the subscription message
    pub message_type: &'static str,

    // .. private fields ..
}

impl Subscription {
    /// The maximum number of messages that *have* been buffered in
    /// the internal message queue so far
    pub fn watermark(&self) -> u8 {
         // ..
    }
}

/// Pretty close to `#[derive(Debug)]` implementation but reports
/// `self.watermark()` as the value of the `watermark` field
impl fmt::Debug for Subscription { /* .. */ }
```

The application will have access to `Subscription` instances only via shared (immutable) references (`&Subscription`).
This effectively turns the `capacity` and `message_type` fields into constants.
Only the `watermark` API can return a different value on each invocation.

The value returned by `watermark` indicates what's the fullest the subscription's internal message queue has ever been.
The returned value is guaranteed to never be smaller than the value returned by a previous invocation ("monotonically increasing function").
The returned value will never be greater than the value of the `capacity` field ("bounded value function").

#### `Subscription` API in an RTIC application

Given the following RTIC code

``` rust
#[actors]
struct Actors {
    #[subscribe(temperature::Reading, capacity = 2)]
    #[priority = 2]
    temperature_monitor: TemperatureMonitor,

    #[subscribe(power_supply::Alert, capacity = 1)]
    #[subscribe(temperature::Alert, capacity = 4)]
    alert_handler: AlertHandler,

    // .. other actors ..
}
```

The RTIC framework will produce the following public API *if* the `memory-watermark` Cargo feature is enabled.
The memory watermark API is behind an opt-in Cargo feature because it incurs a small runtime overhead on each `post` operation.

``` rust
// One module per actor
mod temperature_monitor {
    pub static SUBSCRIPTIONS: &[Subscription] = &[
        // one `Subscription` element per `#[subscribe]` attribute
    ];
    // ..
}

mod alert_handler {
    pub static SUBSCRIPTIONS: &[Subscription] = &[
        // has 2 elements in this example
    ];
    // ..
}
```

Each actor field in the `#[actors]` struct gets its own module inside the `#[rtic::app]` module.

The `SUBSCRIPTIONS` slice variable inside each actor module will contain as many elements as `#[subscribe]` attributes the actor field has.

The order in which the subscriptions appear in the `SUBSCRIPTIONS` variable is left unspecified.

#### Example usage

``` rust
#[idle]
fn idle(_: idle::Context) -> ! {
    loop {
        if some_condition {
            log::info!(
                "temperature_monitor: {:#?}",
                temperature_monitor::SUBSCRIPTIONS,
            );

            log::info!("alert_handler: {:#?}", alert_handler::SUBSCRIPTIONS);
        }
        // ..
    }
}
```

Example output:

``` text
INFO temperature_monitor: [
    Subscription { capacity: 2, message_type: "temperature::Reading", watermark: 2 },
]
INFO logger: [
    Subscription { capacity: 1, message_type: "power_supply::Alert", watermark: 1 },
    Subscription { capacity: 4, message_type: "temperature::Alert", watermark: 2 },
]
```

# Implementation details

This section discusses some implementation details with the goal of informing RTIC developers about the required implementation effort of the proposed feature.
In that sense, this section is *informative* rather than *normative* so an actual implementation is free to diverge from this.

## Actor resources

The *data* that makes up an actor can be implemented as a regular RTIC resource that's *not* visible to the end user.
As data sharing is not allowed between actors, these "actor resources" are *not* part of the ceiling analysis.

## Subscribe tasks

Each `#[subscribe]` attribute an actor has can be "lowered" to a software task.
This software task will forward its incoming message, whose type matches `#[subscribe]`, to the corresponding "actor resource".
All the "subscribe tasks" get assigned the priority specified for the actor.

``` rust
// input
#[actors]
struct Actors {
    #[priority = 2]
    #[subscribe(MessageA)]
    #[subscribe(MessageB)]
    actor: Actor,
}

// input "lowered" to software tasks
#[task(priority = 2)]
fn actor_MessageA(_: Context, message: MessageA) {
     ACTOR.receive(message); // no lock API here
}

#[task(priority = 2)] // same priority
fn actor_MessageB(_: Context, message: MessageB) {
     ACTOR.receive(message); // thus no lock API
}
```

## `Post`

The `Post` implementations of the `Poster` consist of `spawn` invocations to the appropriate "subscribe task".

``` rust
impl Post<MessageA> for Poster {
    fn post(&mut self, message: MessageA) -> Result<(), MessageA> {
        actor_MessageA::spawn(message)
    }
}
```

## Publish

In the case of multiple subscribers,
the `Post` implementation needs to do all the `spawn` invocations *atomically*, that is inside a critical section, and
it needs to `clone` the message as many times as need.

``` rust
// input
#[derive(Clone)]
struct BroadcastMessage { /* .. */ }

#[actors]
struct Actors {
    #[subscribe(BroadcastMessage)]
    listener1: Listener,
    #[subscribe(BroadcastMessage)]
    listener2: Listener,
}

// generated code
impl Post<BroadcastMessage> for Poster {
    fn post(&mut self, message: M) -> Result<(), M> {
        interrupt::free(|_| { // <- critical section
            listerner1_BroadcastMessage(message.clone());
            listerner1_BroadcastMessage(message); // <- no clone here
        });
    }
}
```

## Memory watermark

An atomic variable (e.g. `Atomicu8`) is created for each `spawn` implementation.
On each `spawn` invocation, the length of the message queue is checked;
if the value is greater than what's stored in the atomic variable then the variable gets updated.
This update operation does not need to be lock-free because `spawn` calls always run inside critical sections.

When the `watermark` API is called, the value of the atomic variable is read and returned.

## Summary

In summary, actors can be "lowered" to regular software tasks and resources thus
an implementation can reuse most of the existing internal code generation API.

# Drawbacks
[drawbacks]: #drawbacks

As any new feature, there's both implementation and maintenance costs.
Both costs should be relatively small for the code generation and analysis "passes" given that actors can be lowered to software tasks and resources so existing code can be reused for that.
Thus the main cost is on the front-end (`rtic-syntax`), i.e.  the parsing pass.
There, the long-term maintenance cost can be kept low by including sufficient tests with the initial implementation;
that should lower the chance of bugs and future changes introducing regressions.

# Alternatives
[alternatives]: #alternatives

## Making `Poster` globally accessible

In the main proposal, `Poster` is made available only to `#[init]` and one can make `Poster` available to tasks as needed by the application.
The goal of that design decision is give the user control over which part of the application can post messages into the actor network.
However, the `Poster` handle can post any and all the message types so the access control is not as fine grained as with the `spawn` API.

It may be worthwhile to instead turn `Poster` into a globally available Zero-Sized Type (ZST).
This gives away the access control feature but meshes in nicely with the `#[init]` extension as
one would then be able to initialize more actors.

``` rust
#[actors]
struct Actors {
    #[init(Actor::new(Poster))] // <- now it can be initialized here
    actor: Actor,
    actuator: Actuator,
    // ..
}

#[init]
fn init(cx: init::Context) -> /* .. */ {
    let servomotor = servo::init(); // needs to be done at runtime
    let poster = Poster;
    let actuator = Actuator::new(servomotor, poster);
    // ..
}

// no resource needed to post messages
#[task(binds = TIMER)]
fn periodic() {
    Poster.post(TriggerMessage).ok();
    // ..
}
```

## Turning `#[init]` return type into a struct

This is somewhat orthogonal to the proposal but the return type of `#[init]` is currently a tuple.
This means any new data that needs to be returned by `#[init]` requires changing the signature of the `#[init]` function.
To avoid the issue in this proposal and future feature additions,
the tuple could be replaced with a `struct` with fields: `locals`, `shared`, `monotonics` and `actors`.

> the chosen type name, "Return", is just a straw man proposal

``` rust
#[init]
fn init(cx: init::Context) -> init::Return {
    // ..
    init::Return {
        locals,
        shared,
        monotonics,
        actors: Actors { actor }
    }
}
```

This also opens the possibility of making the existing `#[local]` and `#[shared]` structs optional.
For example, if one is *only* using `#[actors]` one would write:

``` rust
// no `#[locals], `#[shared]` or `#[monotonic]` items here

#[actors]
struct Actors {
    // ..
    actor: Actor,
    // ..
}

#[init]
fn init(cx: init::Context) -> init::Return {
    // ..
    init::Return {
        // no `shared`, `locals` or `monotonics` field here
        actors,
    }
}
```

However, this would be a **breaking change** -- hence it's *not* part of the main proposal.
As 0.6.0 has not been released at time of writing (there are only alpha pre-releases of it),
it might still be possible to do this breaking change as part of 0.6.0.

## Grouping `Subscription`s differently

For the `memory-watermark` feature, the `Subscription` objects could be grouped differently.
This makes some operations easier and others harder.

### Slice of `Actor`s

Instead of modules; a single `static` variable can be used:

``` rust
pub static ACTORS: &[Actor] = &[
     // populated by the framework
     // one element per field in the `#[actors]` struct
];
```

The app-agnostic `Actor` API:

``` rust
pub struct Actor {
    pub name: &'static str, // e.g. `temperature_monitor`

    pub type_name: &'static str, // e.g. `temperature::Monitor`

    /// `Subscription` is the API presented in the main proposal
    pub subscriptions: &'static [Subscription],
}
```

This API makes it easier to iterate over all the actors, but harder to pick out a specific actor -- that requires checking a string.

## `Actors` struct

Similar single `static` variable approach but using a struct instead of a slice.

``` rust
pub static ACTORS: Actors = /* populated by the framework */;
```

`Actors` would be an *application-specific* struct generated by the framework

``` rust
// Application-specific structure
pub struct Actors {
    pub temperature_monitor: Actor,

     // one field for each field in the `#[actors]` struct
}
```

The `Actor` struct would have the same API as the first alternative minus the `name` field which becomes redundant.

This second alternative sits between the main proposal and alternative 1.

# Unresolved questions
[unresolved]: #unresolved-questions

- The `Poster` type could use a better name (naming is hard)

- The exact crate organization for the new API has not been decided.
  The only hard requirement is that `Receive`, `Post` and `PostSpy` must *not* depend on Cortex-M details so they can be compiled for the host.
  Due to Cargo limitations, it'd be better if `PostSpy` resides in a separate crate rather than in the same crate as `Receive` and `Post` but behind a Cargo feature (Cargo has a `dev-dependencies` concept but not a 'test-specific Cargo features' concept).
