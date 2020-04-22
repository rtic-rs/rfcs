- Feature Name: use-mod-over-const
- Start Date: 2020-04-22
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

With stable support for attributes on modules the const workaround has a contender.

# Motivation
[motivation]: #motivation

As noted in the rtfm::app documentation: 

"This attribute must be applied to a const item of type ().
The const item is effectively used as a *mod item*: its value must be a block that contains items commonly found in modules, like functions and static variables."

Now with stable support in 1.42 for using actual mod the workaround using a const can be removed.

# Detailed design
[design]: #detailed-design

[Stabilize attribute macros on inline modules #64273](https://github.com/rust-lang/rust/pull/64273) with milestone 1.42.

Instead of `const APP: () = {` it is now possible to write just `mod APP {`.

```
//! examples/smallest.rs

#![no_main]
#![no_std]

use panic_semihosting as _; // panic handler
use rtfm::app;

#[app(device = lm3s6965)]
mod APP {}
```

However, since the macros operating on the mod remove the `mod` during code generation, the code within the module will see outside the module, which is not idiomatic.

Implementation, change parsing to expect `mod` instead of `const` and adapt tests and examples.

# How We Teach This
[how-we-teach-this]: #how-we-teach-this

Documentation and books needs to be updated.

# Drawbacks
[drawbacks]: #drawbacks

Requires a bump of MSRV to 1.42 from current 1.36.

Non-idiomatic rust?

# Alternatives
[alternatives]: #alternatives

Possible to keep using const, it is not breaking as far as I know.

# Unresolved questions
[unresolved]: #unresolved-questions

Namespace changes and non-idiomatic rust.
