# avr-progmem

<!-- cargo-sync-readme start -->


Progmem utilities for the AVR architectures.

This Crate provides unsafe utilities for working with with data stored in
the program memory of an AVR micro-controller. And additionally a
'best-effort' safe wrapper struct [`ProgMem`] to simplify working with it.

This crate is implemented only in Rust and some short assembly, it does NOT
depend on the [`avr-libc`] or any other C-library. However, due to the use
of inline assembly, this crate may only be compiled using a **nightly Rust**
compiler.


# AVR Memory

This crate is specifically for [AVR-base micro-controllers][avr] such as
the Arduino Uno (and some other Arduino boards, but not all), which have a
modified Harvard architecture that is the strict separation between program
code and data, while having special instruction to read and write to the
program memory.

While all ordinary data is stored of course in the data domain, where it is
perfectly usable, the harsh constraints of most AVR processors make it very
appealing to use the program memory (also referred to as _progmem_) for
storing constant values. However, due to the Harvard design those values
are not usable with normal instructions (i.e. those emitted from normal
Rust code). Instead, special instructions are required to loaded data from
the program code domain, i.e. the `lpm` (load _from_ program memory)
instruction. And because there is no way to emit it from Rust code, this
crate uses inline assembly to emit that instruction.

However, since there is nothing special about a pointer into program code
which would differentiate it from normal data pointers it is entirely up to
the programmer to ensure that these different 'pointer-type' are not
accidentally mixed. In other words it is `unsafe` in the context of Rust.


# Loading Data from Program Memory

The first part of this crate simply provides a few functions (e.g.
[`read_progmem_byte`]) to load constant data (i.e. a Rust `static` that is
immutable) from the program memory into the data domain, so that
sub-sequentially it is normal usable data, i.e. as owned data on the stack.

Because, as aforementioned, a simple `*const u8` in Rust dose not specify
whether is lives in the program code domain or the data domain, all
functions which simply load a given pointer from the program memory are
inherently `unsafe`.

Notice that using a `&u8` reference might make things rather worse than
safe. Because keeping a pointer/reference/address into the program memory
as Rust reference, might easily cause it to be dereferenced even in safe
code, but since that address is only valid in the program code domain (and
Rust doesn't know about it) it would illegally load the address from the
data memory causing Undefined Behavior!

## Example

```rust
use avr_progmem::read_progmem_byte;

// This static must never be directly dereferenced/accessed!
// So a `let data: u8 = P_BYTE;` is Undefined Behavior!!!
/// Static byte stored in progmem!
#[link_section = ".progmem"]
static P_BYTE: u8 = b'A';

// Load the byte from progmem
// Here, it is sound, because due to the link_section it is indeed in the
// program code memory.
let data: u8 = unsafe { read_progmem_byte(&P_BYTE) };
assert_eq!(b'A', data);
```


# The best-effort Wrapper

Since working with progmem data, is inherently unsafe and rather
difficult to do correctly, this crate introduces the best-effort 'safe'
wrapper [`ProgMem`], that is supposed to only wrap data in progmem, thus
offering only function to load its content using the progmem loading
function.
The latter are fine and safe, given that the wrapper type really contains
date in the program memory. Therefore, to upkeep that invariant, the
constructor is `unsafe`.

Yet to make that also easier, this crate provides the [`progmem!`] macro
(it has to be a macro), which will create a static variable in program
memory for you and wrap it in the `ProgMem` struct. It will ensure that the
static will be store in the program memory by defining the
`#[link_section = ".progmem"]` attribute on it. This makes the load
functions on that struct sound, and additionally prevents users to
accidentally access directly that static, which since it is in progmem
would be fundamentally unsound.

## Example

```rust
use avr_progmem::progmem;

// It will be wrapped in the ProgMem struct and expand to:
// ```
// #[link_section = ".progmem"]
// static P_BYTE: ProgMem<u8> = unsafe { ProgMem::new(b'A') };
// ```
// Thus it is impossible for safe Rust to directly dereference/access it!
progmem! {
    /// Static byte stored in progmem!
    static progmem P_BYTE: u8 = b'A';
}

// Load the byte from progmem
// It is still sound, because the `ProgMem` guarantees us that it comes
// from the program code memory.
let data: u8 = P_BYTE.load();
assert_eq!(b'A', data);
```


# Other Architectures

As mentioned before, this crate is specifically designed to be use with
AVR-base micro-controllers. But since most of us don't write their programs
on an AVR system but e.g. on x86 systems, and might want to test them
there (well as far as it is possible), this crate also has a fallback
implementation for all other architectures that are not AVR, falling back
to a simple Rust `static` in the default data segment. And all the data
loading functions will just dereference the pointed to data, assuming that
they just live in the default location.

This fallback is perfectly safe on x86 and friend, and should also be fine
on all further architectures, otherwise normal Rust `static`s would be
broken. However, it is an important point to know when for instance writing
a library that is not limited to AVR.


[`ProgMem`]: struct.ProgMem.html
[`read_progmem_byte`]: fn.read_progmem_byte.html
[`progmem!`]: macro.progmem.html
[`avr-libc`]: https://crates.io/crates/avr-libc
[avr]: https://en.wikipedia.org/wiki/AVR_microcontrollers


<!-- cargo-sync-readme end -->

# License

Licensed under Apache License, Version 2.0 ([LICENSE](LICENSE) or https://www.apache.org/licenses/LICENSE-2.0).

## Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in this project by you, as defined in the Apache-2.0 license, shall be licensed as above, without any additional terms or conditions.


