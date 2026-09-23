+++
title = "Rust - In-Place Init for pin-heads"
+++

The Rust project has been working on in-place construction of values. I'm not much of a PL person or language designer but as someone who's been following the debate, I thought it'd be fun to write up a small variant on what's being discussed!

## Basic Proposal

The basic idea I had is a hybrid of Yosh Wuyts's ["placing functions"](https://blog.yoshuawuyts.com/placing-functions) proposal and Olivier Faure's [pinned places](https://poignardazur.github.io/2024/08/16/pinned-places/) ideas that I'd call "return position pinned r-values" (RPPR)[^1]

Here's the syntax:

```rust
struct Cat {
    age: u32,
}

impl Cat {
    fn new(age: u32) -> pin Self {
        Self { age } // ← Constructs `Self` in the caller's frame and pins it.
    }

    fn age(&self) -> &u32 {
        &self.age
    }
}

fn main() {
    let cat = Cat::new(12); // pinned
    assert_eq!(cat.age(), &12);
}
```

Semantically, RPPR works by requiring the caller to supply a destination place for T and pass a hidden out pointer that the callee uses to construct the value (subject to target ABI rules). After construction, the destination place is pinned (but if the value is `Unpin`, it can be moved immediately out of this state).

Here's my design reasoning:

1. I think Rust really got the initialization story right: no constructors, total fields, just create the struct. Maintaining this illusion is a good thing.
2. In-place init is an edge case - in most cases people should just keep returning normal values and trust that the compiler/ABI/CPU gods will pick the most optimal thing. Hence, try to minimize new concepts & surface syntax.
3. Guaranteed value elimination [(GVE)](https://github.com/PoignardAzur/in-place-init-overview/blob/main/solutions/gve-with-init/README.md) is attractive for these reasons but would have ABI consequences. C++, which has similar guaranteed copy-elision semantics, allows implementations to return small value types like `Cat` in registers for performance reasons[^2] This is surprising given that it's supposed to be a guarantee!
4. Hence, some sort of surface syntax is needed for placing functions in Rust. The motivation & closest concept in the language today is `pin`; building on this links to how people solve this today.

## What about Move?

Lots of contributors to the language have been working on a related proposal to add a `Move` trait to the language which would replace `Pin`. The good news is that this proposal is forward compatible with that - in a `Move` world, declaring a type `!Move` is THE way to require in-place construction.

```rust
struct Cat {
    age: u32,
}

impl !Move for Cat {}

impl Cat {
    fn new(age: u32) -> Self {
        Self { age } // ← Constructs `Self` in the caller's frame.
    }
}
```

This ends up looking like GVE but is only guaranteed for `!Move` types (You would need to declare a wrapper type for normal types if you need to support in-place construction.)

## Fallibility

Fallibility seems like the toughest challenge for in-place init. Conceptually the problem as I see it is this: Rust uses a value type `Result<T, Error>` for fallible operations, but there's no requirement that `Result` has the same shape as `T`. Ideally you'd like to pass a hidden `T` out-pointer and allow the error/discriminant to be passed as a side channel.

Although you could imagine a world where you write `Result<pin T, Error>`, I think that's ad hoc and kinda weird and `pin` should only be allowed as a modifier on the (outer) value. The solution might be to bake the fallibility effect into the language ala Swift:

```rust

pub struct PinnedThing { ... }

impl PinnedThing {
    fn new() -> pin Self throws Error;
}

pub struct CoupleOfThings {
    first: PinnedThing,
    second: PinnedThing,
}

impl CoupleOfThings {
    fn new() -> pin Self throws Error {
        Self {
            first: PinnedThing::new()?,
            second: PinnedThing::new()?
        }
    }
}
```

In this case, `Self throws Error` does _not_ desugar to `Result<T, Error>`[^3] However, it does impl `Try` and presumably has some easy way to convert it back to a `Result` (plus some sugar as needed).

## Box constructor

Here we allow function types to declare `pin` on return types as well. Presumably since the language guarantees that this is an out-pointer, you could also do a curry-ish transformation from `impl FnOnce() -> pin T` to `impl FnOnce(*mut T) -> ()`.

```rust
pub struct PinnedThing { ... }

fn make_thing() -> pin PinnedThing;

impl<T> Box<T> {
    pub fn new_with(callback: impl FnOnce() -> pin T) -> pin Self {
        let allocation = Box::<T>::new_uninit().into_pin();
        let ptr: *pin mut T = allocation.as_mut_ptr();
        unsafe {
            *ptr = callback();
            allocation.assume_init()
        }
    }
}

// Compiler would adapt the thunk if needed
let my_box = Box::new_with(|| make_thing());
```

That said, people should still prefer to use `Box::new` for most use cases, both for simplicity and code bloat reasons. (I suppose you could potentially make `Box::new` work with in-place init if you combine inline always + MIR move elimination but then both of these become really load-bearing.)

## Observe address

Similar to GVE, this proposal would lean on MIR move elimination to elide places and make sure that a pinned value isn't moved.

```rust
pub struct SelfRef {
    a: u32,
    addr_of_a: *const u32,
}

fn make_self_ref(value: u32) -> pin SelfRef {
    let pin mut ret = SelfRef {
        a: value,
        addr_of_a: std::ptr::null(),
    };
    ret.addr_of_a = &raw const ret.a;
    // compiler checks that ref remains pinned
    ret 
}

let value = make_self_ref();
assert_eq!(value.addr_of_a, &raw const value.a);
```

## Weird corner/Conclusion

Jumping ahead, it'd be really cool to make this work for C programmers but I'm not sure that it works. Does this need a different sort of primitive?

```rust

impl<T> MaybeUninit<T> {
    // assume_init_pin reuses the location of self as a T
    pub fn assume_init_pin(pin self) -> pin T {
        // assume inhabited
        unsafe { transmute(self) }
    }
}
```

Anyways, this is my proposal - if it's crazy please feel free to tell me to put a pin in it! (but please do so nicely)

[^1]: I know technically Rust doesn't say r-values but RPPV doesn't have quite the same ring, does it?
[^2]: I believe the way around this is to make it not trivially destructible?
[^3]: You could give it the [Herbception ABI](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2544r0.html) even!
