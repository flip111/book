# Panicking

Panicking is a core part of the Rust language. Built-in operations like indexing
are runtime checked for memory safety. When out of bounds indexing is attempted
this results in a panic.

In the standard library panicking has a defined behavior: it unwinds the stack
of the panicking thread, unless the user opted for aborting the program on
panics.

In non-standard programs, however, the panicking behavior is left undefined. A
behavior can be chosen by declaring a `#[panic_handler]` function. This function
must appear exactly *once* in the dependency graph of a program, and must have
the following signature: `fn(&PanicInfo) -> !`, where [`PanicInfo`] is a struct
containing information about the location of the panic.

[`PanicInfo`]: https://doc.rust-lang.org/core/panic/struct.PanicInfo.html

Given that embedded systems range from user facing to safety critical (cannot
crash) there's no one size fits all panicking behavior but there are plenty of
commonly used behaviors. These common behaviors have been packaged into crates
that define the `#[panic_handler]` function. Some examples include:

- [`panic-abort`]. A panic causes the abort instruction to be executed.
- [`panic-halt`]. A panic causes the program, or the current thread, to halt by
  entering an infinite loop.
- [`panic-itm`]. The panicking message is logged using the ITM, an ARM Cortex-M
  specific peripheral.
- [`panic-semihosting`]. The panicking message is logged to the host using the
  semihosting technique.

[`panic-abort`]: https://crates.io/crates/panic-abort
[`panic-halt`]: https://crates.io/crates/panic-halt
[`panic-itm`]: https://crates.io/crates/panic-itm
[`panic-semihosting`]: https://crates.io/crates/panic-semihosting

You may be able to find even more crates searching for the [`panic-handler`]
keyword on crates.io.

[`panic-handler`]: https://crates.io/keywords/panic-handler

A program can pick one of these behaviors simply by linking to the corresponding
crate. The fact that the panicking behavior is expressed in the source of
an application as a single line of code is not only useful as documentation but
can also be used to change the panicking behavior according to the compilation
profile. For example:

``` rust,ignore
#![no_main]
#![no_std]

// dev profile: easier to debug panics; can put a breakpoint on `rust_begin_unwind`
#[cfg(debug_assertions)]
extern crate panic_halt;

// release profile: minimize the binary size of the application
#[cfg(not(debug_assertions))]
extern crate panic_abort;

// ..
```

In this example the crate links to the `panic-halt` crate when built with the
dev profile (`cargo build`), but links to the `panic-abort` crate when built
with the release profile (`cargo build --release`).

## An example

Here's an example that tries to index an array beyond its length. The operation
results in a panic.

``` rust
#![no_main]
#![no_std]

extern crate panic_semihosting;

use cortex_m_rt::entry;

#[entry]
fn main() -> ! {
    let xs = [0, 1, 2];
    let i = xs.len() + 1;
    let _y = xs[i]; // out of bounds access

    loop {}
}
```

This example chose the `panic-semihosting` behavior which prints the panic
message to the host console using semihosting.

``` console
$ cargo run
     Running `qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb (..)
panicked at 'index out of bounds: the len is 3 but the index is 4', src/main.rs:12:13
```

You can try changing the behavior to `panic-halt` and confirm that no message is
printed in that case.
