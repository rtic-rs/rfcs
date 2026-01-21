- Feature Name: `Logger support`
- Start Date: 2019-12-28
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Add support for interrupt priority based logging in RTFM (ala [`cortex-m-funnel`]).

# Motivation
[motivation]: #motivation

One of the most time consuming issues of embedded programming is debugging. And especially when it comes to event based systems where debugging can get in the way of timing requirements. Debugging is often done through logging with `printf`-like methods, and the logger implementation will often impede real-time requirements, causing the system to behave differently with and without logging enabled.
This often comes from logger implementations having global critical sections for accessing the log buffers, but with RTFM we can have seamless support for logging without critical sections by leveraging the design and ideas behind [`cortex-m-funnel`].

The idea behind it is that there exists one logging queue per interrupt priority, and a logging function always writes to the currently running priority which in turn means that no critical sections are needed (see [`cortex-m-funnel`] for implementation details).

Adding this feature will help users immensely, as figuring out logging is not a trivial task. This will also help adoption of RTFM as **we made logging easy**.

# Detailed design
[design]: #detailed-design

The important thing is to define how large the buffers should be. Either we can go with the simplest case (and have all buffers the same size):

```rust
#[app(logger = 256)]
// ...
```

Or we can have a base size, together with possible specialization (bikeshed of API needed):

```rust
// Base size of 256 bytes, but the queue for priority 7 is of 1024 bytes
#[app(logger = [base = 256, 7 = 1024])]
// ...
```

The final implementation of the logging is very simple. Simply, based on the analysis, fill out the [`cortex-m-funnel`] macro.

One thing we should be careful with is that the logger should be **external** to RTFM so it is easy to include in libraries. Just as the [`log`] crate is today.

## Should the log be part of task definition?

It is my view that it should not. We should follow the common design of [`log`] and [`cortex-m-funnel`], where the log macros can be added anywhere without propagating it down through the call tree. This is the key feature that makes it easy to use, just have RTFM provide the information to properly initialize the queues for the logger.

# How We Teach This
[how-we-teach-this]: #how-we-teach-this

This should be added to the book as its own `Safe logging` chapter.

# Drawbacks
[drawbacks]: #drawbacks

* Increase of complexity of RTFM, but I would argue that this is of much greater help than not to have it.

# Alternatives
[alternatives]: #alternatives

* Use `cortex-m-funnel` manually, however this would be easy to get wrong

# Unresolved questions
[unresolved]: #unresolved-questions

* API design

[`cortex-m-funnel`]: https://github.com/japaric/cortex-m-funnel
[`log`]: https://github.com/rust-lang/log
