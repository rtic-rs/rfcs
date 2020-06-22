- Feature Name: out-pointer-init
- Start Date: 2020-06-22
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Optional "out-pointer" optimisation for the `#[init]` function aiming to reduce total stack usage inside the function.

# Motivation
[motivation]: #motivation

The stack frame size of the init function is roughly `2 * init::LateResources`, this is because the function has to hold the local creation of every resource and allocate stack space to store it in `init::LateResources` too, only to then write it in the static memory anyway. In larger applications, with bigger resources, this can lead to drastically large init functions that can cause problems.


# Detailed design
[design]: #detailed-design

The "out-pointer" pattern which would work in current RTIC code would look like the following, using two resources

```rust
#![deny(unsafe_code)]
#![deny(warnings)]
#![no_main]
#![no_std]

use cortex_m_semihosting::{debug, hprintln};
use lm3s6965::Interrupt;
use panic_semihosting as _;

#[rtic::app(device = lm3s6965)]
const APP: () = {
    struct Resources {
        #[init(MaybeUninit::uninit())]
        big_struct_storage: MaybeUninit<BigStruct>,
        big_struct: &'static mut BigStruct,
    }

    #[init(resources = [big_struct_storage])]
    fn init(c: init::Context) -> init::LateResources {
        let big_struct: &'static mut _ = unsafe {
            // write directly into the static storage
            c.resources
                .big_struct_storage
                .as_mut_ptr()
                .write(BigStruct::new());
            &mut *c.resources.big_struct_storage.as_mut_ptr()
        };
        init::LateResource {
            // assign the reference so we can use the resource
            big_struct: big_struct
        }
    }
};
```

Seeing as rtic using `MaybeUninit` under the hood, it is possible to expose a mechanism to allow direct access to uninitialized resource memory; of course this would be opt in and it would be up to the user to make sure its properly initialised. Perhaps a `#[ManualInit]` attribute could be introduced which would pass the uninitialized data to the init function.

```rust
#![deny(unsafe_code)]
#![deny(warnings)]
#![no_main]
#![no_std]

use cortex_m_semihosting::{debug, hprintln};
use lm3s6965::Interrupt;
use panic_semihosting as _;

#[rtic::app(device = lm3s6965)]
const APP: () = {
    struct Resources {
        #[ManualInit]
        big_struct: BigStruct,
    }

    // compile error if `#[ManualInit]` error specified, but resource not included here
    #[init(resources = [big_struct])]
    fn init(c: init::Context) {
        unsafe {
            // write directly into the static storage
            c.resources
                .big_struct_storage
                .as_mut_ptr()
                .write(BigStruct::new());
            
        }
    }
};
```

# How We Teach This
[how-we-teach-this]: #how-we-teach-this

Add examples, and add a section in the book.

# Drawbacks
[drawbacks]: #drawbacks

Leaves possible unsafety issues on the table, if a user forgets to actually initialise the data. Could we design it in such a way such this is checked?

# Alternatives
[alternatives]: #alternatives

The first PoC example explores that this is currently possible already, but requires the introduction of two variables to do so. 

# Unresolved questions
[unresolved]: #unresolved-questions

Bikeshed the attribute name.
