- Feature Name: `generators`
- Start Date: 2019-10-28
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Add experimental support (behind a Cargo feature) for generators to Real Time
for The Masses.

# Motivation
[motivation]: #motivation

Support generator sugar (i.e. `yield`) to ease the creation of interrupt-driven
state machines.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The proposal introduces the concept of *generator tasks*. A generator task is
basically never-ending code with (explicit) suspension points (`yield`). Run to
completion semantics also apply to generator tasks: when a generator task is
triggered, it starts executing from its last suspension point to the next
suspension point. Apart from being written using generator syntax, generator
tasks have the same capabilities as plain old tasks: they have priorities
(priority-based scheduling also applies to them), they can access resources
(`lock` API) and they can spawn other tasks (`spawn` & `schedule` APIs).

A function with return type `impl Generator<Yield = (), Return = !>` and
annotated with the `#[task]` attribute is a generator task *constructor*. This
function will run *once* after `init` returns and before any task can start; its
return value is the generator that will continuously be resumed by an interrupt
signal (hardware generator task) or by the software (software generator task).

The program below features a generator task. This generator task will be resumed
by `idle` through the `spawn` API.

``` rust
// yes, you need all these feature gates
#![feature(generator_trait)]
#![feature(generators)]
#![feature(never_type)]
#![feature(type_alias_impl_trait)]

#[rtfm::app(/* .. */)]
const APP: () = {
    // ..

    #[init]
    fn init(cx: init::Context) {
        println!("init");
    }

    // generator task
    #[task(priority = 1, resources = [x])]
    fn a(cx: a::Context) -> impl Generator<Yield = (), Return = !> {
        println!("A");

        move || loop {
            println!("B");
            yield;

            cx.resources.x.lock(|x: &mut i64| {
                // .. (cannot `yield` here) ..
            });

            println!("C");
            yield;
        }
    }

    // non-generator task (plain task)
    #[task(priority = 2, resources = [x])]
    fn b(cx: b::Context) { /* .. */ }

    #[idle(spawn = [a])]
    fn idle(cx: idle::Context) -> ! {
         println!("idle");

         cx.spawn.a();
         cx.spawn.a();
         cx.spawn.a();

         loop { /* .. */ }
    }
};
```

This program prints:

``` text
init
A
idle
B
C
B
```

## Intended restrictions

Although not shown in the signature of the generator constructor the returned
generator must satisfy the `'static` bound (its an implicit bound); that is all
the references it captures must have a `'static` lifetime. For this reason in
most cases one will need to write the generator as `move || { /* .. */ }`. Later
in this document it is explained why this `'static` bound is required.

Also it's not possible to suspend the generator (`yield`) in the middle of
critical section (`lock`). This operation would leave the system in an abnormal
dynamic priority that causes low priority tasks, and `idle`, to never be
triggered or resumed. This operation does not break memory safety but it ruins
the scheduling policy of the framework so it's not allowed at all, not even via
a `unsafe` API.

## Task local data

In plain tasks, task local data -- `static mut` variables declared at the
beginning of the task -- is used to preserve state across different executions
of a task handler.

In generator tasks one can use stack variables (`let` statements) to preserve
state across suspension points (`yield`).

``` rust
#[rtfm::app(/* .. */)]
const APP: () = {
    // ..

    #[task]
    fn a(cx: a::Context) {
        // transformation -- non-'static lifetime
        static mut COUNT: i32 = 0;
        let count: &mut i32 = COUNT;

        println!("{}", *count);
        *count += 1;
    }

    // can be rewritten as
    #[task]
    fn b(cx: b::Context) -> impl Generator<Yield = (), Return = !> {
        || {
            let mut count = 0;

            loop {
                println!("{}", count);
                count += 1;
                yield;
            }
        }
    }
};
```

In generator tasks, `static mut` variables declared at the beginning of its
constructor are transformed into `&'static mut` references that can be moved
into the generator.

``` rust
#[rtfm::app(/* .. */)]
const APP: () = {
    // ..

    #[task]
    fn a(cx: a::Context) -> impl Generator<Yield = (), Return = !> {
        // transformation -- 'static lifetime
        static mut BUFFER: [u8; 128] = [0; 128];
        let buffer: &'static mut [u8; 128] = BUFFER;

        move || loop {
            // .. use `buffer` here ..
        }
    }
};
```

## Highest priority access
[highest-priority-access]: #highest-priority-access

With non-generator tasks, when a resource is contended by many tasks the top
priority task gets a *direct* mutable reference (`&mut-`) to the resource data
and the rest of lower priority tasks get a resource proxy. We cannot provide
this convenience in generator tasks because they have suspension points that can
cause a mutable reference to outlive the execution of the interrupt handler.
This situation is easier to visualize with an example:

``` rust
#[rtfm::app(..)]
const APP: () = {
    struct Resources { x: i64 }

    #[task(resources = [x], priority = 2)]
    fn a(cx: a::Context) -> impl Generator<Yield = (), Return = !> {
        move || loop {
            // (NOTE this is *not* a self-referential borrow)
            let x: &mut i64 = cx.resources.x;

            *x = 1;

            yield; // task `b` can run before `a` is resumed

            if *x == 1 { // <- the compiler may assume this always is true
                // ..          because the exclusive borrow (&mut -) spans
                //             *over* the suspension point (`yield`)
            }
        }
    }

    #[task(resources = [x], priority = 1)]
    fn b(cx: b::Context) {
        cx.resources.x.lock(|x| *x += 1);
    }
};
```

Therefore even the highest priority generator tasks will need to use the
closure-based `lock` API to access their resources. This ensures exclusive views
(`&mut-`) into resource data don't outlive the interrupt handler that calls into
the generator. To be sound (have no UB), the previous example would need to be
rewritten as:

``` rust
#[rtfm::app(..)]
const APP: () = {
    struct Resources { x: i64 }

    #[task(resources = [x], priority = 2)]
    fn a(cx: a::Context) -> impl Generator<Yield = (), Return = !> {
        move || loop {
            cx.resources.x.lock(|x| {
                *x = 1;

                // this would be a compiler error
                // yield;
            });

            yield; // task `b` can run before `a` is resumed

            // the compiler cannot make any assumption about the value of `x`
            if cx.resources.x.lock(|x| *x) == 1 {
                // ..
            }
        }
    }

    #[task(resources = [x], priority = 1)]
    fn b(cx: b::Context) {
        cx.resources.x.lock(|x| *x += 1);
    }
};
```

This limitation does not apply to share-only (`&-`) resources. It's OK to take a
shared reference to these from within a generator task and keep the reference
alive across a suspension point. Therefore it's not required to use the `lock`
API on these resources.

``` rust
#[rtfm::app(..)]
const APP: () = {
    struct Resources { x: i64 }

    #[task(resources = [&x])]
    fn a(cx: a::Context) -> impl Generator<Yield = (), Return = !> {
        move || loop {
            let x: &i64 = cx.resources.x;

            println!("{}", x);

            yield; // OK to yield here

            println!("{}", x);
        }
    }
};
```

## Current limitations

No baseline information is available to generator tasks. That means the
`Context` struct will contain neither the `started` or the `scheduled` field.
These limitations are not intended but a product of current limitations in the
Rust language.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This section discusses some implementation details that will have an effect on
the public API exposed by the RTFM framework.

## No BASEPRI optimizations

In *non-generator* tasks the `lock` operation is optimized to omit some BASEPRI
operations in nested locks. This is implemented by shadowing the BASEPRI
register on the stack. All `lock` operations use this stack variable, instead of
reading the register, and based on its value decide to write to BASEPRI or not.
The value of this variable can be traced at compile time so the compiler is able
to optimize it away along with unnecessary BASEPRI writes.

In code the optimization looks like this:

Given the user code,

``` rust
#[task(binds = UART0, priority = 1, resources = [x, y])]
fn uart0(cx: uart0::Context) {
    let (mut x, mut y) = (cx.resources.x, cx.resources.y);

    x.lock(|_| {
        y.lock(|_| {
             // ..
        });
    });
}
```

The framework produces this:

``` rust
mod resources {
    // repeat for resource `y`
    struct x<'a> { priority: &'a Cell<u8> }

    impl rtfm::Mutex for x {
        fn lock(&mut self, f: impl FnOnce(&mut _) -> R) -> R {
            const CEILING: u8 = 3;
            let current = self.priority.get();
            if current < CEILING {
                self.priority.set(CEILING);
                BASEPRI::write(ceiling);
                let r = f(/* .. */);
                BASEPRI::write(current);
                self.priority.set(current);
                r
            } else { // avoid BASEPRI write
                f(/* .. */);
            }
        }
    }
}

#[no_mangle]
fn UART0() {
    // tracks the dynamic priority of the task
    let priority = &Cell::new(1); // starts at the static priority of 1

    // call into user code
    uart0(uart0::Context {
         resources: uart0::Resources {
             x: resources::x { priority },
             y: resources::y { priority },
         }
    });
}
```

This approach can not be translated to generator tasks because resources are
captured by the generator *by value* rather than reconstructed each time the
generator is resumed.

To elaborate: the generator is stored in a static variable so its captures
(captured variables) can *not* point to any particular stack frame -- that stack
frame may be gone the next time the generator is resumed. In other words, if
the captures are references then they must satisfy the `: 'static` bound. The
consequence is that if we want to shadow the BASEPRI register we would need to
use a static variable to track its value. But if we do that then the compiler
will no longer be able to perform the same optimizations as today --
optimizations around static variables are more restricted.

Thus different code needs to run when the `lock` method is called from within a
generator task. To implement this we'll use different set of resource proxies
in generator tasks; these proxies will be Zero Sized Types (ZST) to keep the
generator small in size (remember that resources are stored in the generator;
thus they'll be stored in a static variable). Because the same resource may be
accessed from generator tasks and non-generator tasks in the same program we'll
need more than one `resources` module. This requirement is easier to visualize
with an example:

> NOTE: the names for the new modules are up for debate

``` rust
#[rtfm::app(/* .. */)]
const APP: () = {
    #[task(binds = UART0, priority = 1, resources = [x])]
    fn uart0(cx: uart0::Context) {
        let x: resources::x = cx.resources.x;
        //     ^^^^^^^^^^^^ resource proxy type
    }

    #[task(binds = UART1, priority = 2, resources = [x])].
    fn uart1(_: uart1::Context) -> impl Generator</* .. */> {
        let x: gresources::x = cx.resources.x; // this is a different type
        //     ^
    }

    #[task(binds = UART2, priority = 3, resources = [x])]
    fn uart2(_: uart2::Context) -> impl Generator</* .. */> {
        let x: gresourcesd::x = cx.resources.x; // this is also a different type
        //     ^         ^
    }
};
```

Here each task access the same resource data but does so through different
resource proxy types. The `resources::x` proxy is the proxy we have today; its
`lock` method uses the `priority` stack variable to optimize away some BASEPRI
operations. The `gresources::x` proxy is the proxy described earlier in this
section; its `lock` method reads BASEPRI and uses the BASEPRI_MAX register.
`gresourcesd::x` is yet another proxy type that's handed only to the tasks with
the highest priority; its `lock` method is a no-op (it doesn't modify or read
the BASEPRI register) -- we explained why this no-op `lock` is required in a
[previous section](#highest-priority-access).

## Generator size optimizations

Share-only resources (`&x` in the `resources` list) in generators can be
implemented in two ways.

The obvious way is to store a `&'static` reference in the generator: the
`Context.resources.share_only_resource` value provided by the framework would
have type `&'static T`; the generator would then store a copy of this value.
This has a static memory overhead of one pointer per share-only resource,
generator pair -- even though the `&'static` pointer stored in the generator
remains constant for the whole execution of the program the compiler will not be
able to optimize to promote it into a constant (`const`) value.

The other approach is to store a Zero Sized Type (ZST) resource proxy in the
generator. With this approach it is guaranteed that the proxies won't use any
memory space at runtime. This document proposes using this second approach. The
relevant code for this approach is shown below:

User code:

``` rust
#[rtfm::app(/* .. */)]
const APP: () = {
    struct Resources { key: [u8; 128] }

    #[task(resources = [&key])]
    fn foo(cx: foo::Context) -> impl Generator<Yield = (), Return = !> {
        let key: resources::key = cx.resources.key;
        move || loop {
            // .. use `key` ..
        }
    }
};
```

Macro expansion:

``` rust
mod resources {
    pub struct key {
        _not_send_or_sync: PhantomData<*mut ()>,
    }
}

mod foo {
     pub struct Context {
         pub resources: Resources,
         // ..
     }

     pub struct Resources {
         pub key: resources::key,
         // ..
     }
}

const APP: () = {
    // initialized at runtime
    static mut key: MaybeUninit<[u8; 128]> = MaybeUninit::uninit();

    impl core::ops::Deref for resources::key {
        type Target = [u8; 128];

        fn deref(&self) -> &Self::Target {
            // tasks always observe this in the initialized state
            unsafe { &*key.as_ptr() }
        }
    }

    // ..
};
```

# Drawbacks
[drawbacks]: #drawbacks

The implementation of this feature depends on 4 unstable Rust feature behind the
following feature gates: `never_type` (the `!` type), `generators` (the `yield`
operator), `generator_trait` (the `core::ops::Generator` trait) and
`type_alias_impl_trait` (`type T = impl Trait`). As these features are deemed
unstable, they may change, in syntax or semantics, in any new nightly release.
Therefore keeping generator support working on the latest nightly is likely to
significantly increase the maintenance effort.

# Alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Non-zero-cost baseline information

In the main proposal baseline information is not available to generator tasks.
It is possible to make this information available to generator tasks *today* but
the resulting implementation will *not* be zero cost. The suggested API is as
follows:

``` rust
#[rtfm::app(/* .. */)]
const APP: () = {
    #[task(binds = UART0)]
    fn foo(cx: foo::Context) -> impl Generator<Yield = (), Return = !> {
        move || loop {
            let now = cx.started.get();

            yield;

            let then = cx.started.get();
            // ..
        }
    }
};
```

In this hardware generator task the instant at which the task was triggered /
resumed is obtained by calling the `get` method on the `started` value. This API
can be implemented using a static variable; the interrupt handler would set the
variable before resuming the generator; and `started::get` would read the
updated variable.

``` rust
mod foo {
    pub struct Context {
        pub started: Started,
        // ..
    }

    // ..
}

const APP: () = {
    static mut fooI: MaybeUninit<Instant> = MaybeUninit::uninit();

    impl foo::Started {
        fn get(&self) -> Instant {
            unsafe { fooI.as_ptr().read() }
        }
    }

    #[no_mangle]
    fn UART0() {
        let now = Instant::now();
        fooI.as_mut_ptr().write(now);
        // prevent the compiler from reordering the previous write
        core::sync::compiler_fence(Ordering::SeqCst);
        // .. then resume the generator ..
    }
};
```

In the future, with more language support for generators, it will be possible to
make baseline information available to tasks in a truly zero cost fashion. See
the [future possibilities][future-possibilities] section.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Pick better names for the `gresources` and `gresourcesd` modules.

# Future possibilities
[future-possibilities]: #future-possibilities

This section contains potential additions to the main proposal. These additions
may be blocked on an unimplemented language features and / or may require more
design work before they can be formally proposed as RFCs.

## Resumption arguments

It is expected that the [`Generator.resume`][resume] method will gain an
argument in the future. This resumption argument will appear in generator syntax
as the return value of the `yield` operator. The following code snippet is
made-up syntax for this hypothetical feature:

``` rust
fn foo() -> impl Generator<Resume = i32, Yield = (), Return = !> {
    |x: i32| {
        println!("{}", x)
        let y: i32 = yield ();
        println!("{:?}", (x, y))

        loop {
            println!("{}", yield ());
        }
    }
}

fn bar(foo: Pin<&mut impl Generator<Resume = i32, Yield = (), Return = !>>) {
    foo.resume(0); // prints "0"
    foo.resume(1); // prints "(0, 1)"
    foo.resume(2); // prints "2"
}
```

[resume]: https://doc.rust-lang.org/1.38.0/core/ops/trait.Generator.html#tymethod.resume

We'll use this syntax / API to explain how this feature could be used to add
more features to generator tasks.

### Software tasks inputs

With resumption arguments we can implement software generator tasks that take
inputs. The input to the task would be passed to the generator as a resumption
argument and would become the return value of the `yield` operator.

The following example showcases how to rewrite a software task that takes an
input as a generator task:

``` rust
#[rtfm::app(/* .. */)]
const APP: () = {
    #[task]
    fn foo(cx: foo::Context, input: i32) { /* .. user code .. */ }

    // `foo` rewritten as a generator task becomes
    #[task]
    fn bar(cx: bar::Context) -> impl Generator<Resume = i32, Yield = (), Return = !> {
        //                                     ^^^^^^^^^^^^
        |mut input: i32| loop {
            // .. user code ..
            input = yield ();
        }
    }
};
```

The relevant part of the macro expansion is shown below:

``` rust
const APP: () = {
    #[no_mangle]
    fn UART0() {
        while let Some((i, task)) = RQ1.dequeue() { // drain ready queue
            match task {
                T::foo => {
                    // get next task input
                    let input: i32 = foo_INPUTS.as_ptr().offset(i).read();
                    // ..
                    some_generator.resume(input);
                    //                    ^^^^^ addition
                }
                // ..
            }
        }
    }
};
```

### Baseline information

A similar approach can be used to pass baseline information to tasks. The code
snippet below shows how to access baseline information in a plain task vs in a
generator task.

``` rust
#[rtfm::app(/* .. */)]
const APP: () = {
    #[task(binds = UART0)]
    fn uart0(cx: uart0::Context) {
        let started = cx.started;
        // .. user code ..
    }

    // can be rewritten as
    #[task(binds = UART1)]
    fn uart1(
        cx: uart1::Context,
    ) -> impl Generator<Resume = Instant, Yield = (), Return = !> {
        |mut started: Instant| loop {
            // .. user code ..
            started = yield ();
        }
    }
};
```

If a software generator task has baseline information and receives inputs then
the resume argument would need to become a structure that holds both pieces of
information.
