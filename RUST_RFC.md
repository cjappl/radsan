# Abstract

Many software projects that utilize Rust are subject to real-time constraints. Software that is written for use in audio, embedded, robotics, and aerospace must adhere to strict deterministic-time execution or face consequences that may be catastrophic. LLVM 20 introduces RealtimeSanitizer a novel approach that detects and reports disallowed non-deterministic execution time calls in real-time contexts.

This RFC proposes that RealtimeSanitizer be integrated into the Rust ecosystem. To serve that end, we propose 4 changes, outlined in this document:

1. RealtimeSanitizer can be enabled in unstable mode - like the other sanitizers
2. The introduction of `nonblocking` (marking a function as real-time constrained) and `blocking` (marking a function as inappropriate for use in a `nonblocking` context)
3. The addition of the `blocking` attribute to `std::sync::Mutex::lock` and `std::sync::RwLock::lock`
4. The addition of the `rtsan_scoped_disabler!` macro

# The problem space

Increasingly, Rust is being used in problem spaces that are real-time constrained, such as audio, robotics, aerospace and embedded. Real-time programming is defined by deadlines - if a solution is not provided by a specific deadline, some consequence may occur. 

For example:

> In an autonomous vehicle perception subsystem, it is not enough to detect a red light. You must detect a red light AND pass it along in N ms, or the car may not stop. This may lead to an accident.

> In audio, you must fill a buffer and pass it back to the operating system within N ms, otherwise your user may hear a click or pop which may damage their audio equipment, or minimally annoy them.

> In aerospace guidance systems if your software doesn't update on a regular tick your simulation of what is happening may diverge from reality. Unfortunately this may also mean your rocket converges with the ground.

Code in these environments must run in a deterministic amount of time. Allocations, locks, and other OS resource access are disallowed because they don't have bound upper execution time. Unbound execution time leads to blown deadlines, and blown deadlines leads to consequences.

A few resources that go into more depth on real-time programming:
* https://en.wikipedia.org/wiki/Real-time_computing
* http://www.rossbencina.com/code/real-time-audio-programming-101-time-waits-for-nothing
* https://www.youtube.com/watch?v=ndeN983j_GQ

# RealtimeSanitizer 

[RealtimeSanitizer](https://clang.llvm.org/docs/RealtimeSanitizer.html) is one approach to detecting and alerting users to these issues when they occur. This new sanitizer has been integrated into LLVM 20. You can explore this tool in Compiler Explorer using the `-fsanitize=realtime` flag.


In clang, a function marked `[[clang::nonblocking]]` is the restricted execution context.  **In these `nonblocking` functions, two broad sets of actions are disallowed:**
## 1. Intercepted calls into libc, such as `malloc`, `socket`, `write`, `pthread_mutex_*` and many more

Each of these actions are known to have non-deterministic execution time. When these actions occur during a `[[clang::nonblocking]]` function, or any function invoked by this function, they print the stack and abort.

[Example of this working in Compiler Explorer](https://godbolt.org/z/sPTh63o67).

The full list of intercepted functions can be found on [GitHub](https://github.com/llvm/llvm-project/blob/main/compiler-rt/lib/rtsan/rtsan_interceptors_posix.cpp), and is continually growing.

## 2. User defined `[[clang::blocking]]` functions

The `[[clang::blocking]]` attribute allows users to mark a function as unsafe for real-time contexts. 

[Example of this working in Compiler Explorer](https://godbolt.org/z/dErqE5nnM)

One classic example of this is a spin-lock `lock` method. Spin locks do not call into a `pthread_mutex_lock`, so they cannot be intercepted. They are still prone to spinning indefinitely, so they are unsafe in real-time contexts. This includes `std::sync::Mutex::lock`, which we will discuss more in a bit.

# Rust

We would like to propose that RTSan be integrated into Rust using similar semantics to clang.

There are a few sub-pieces to consider when integrating RTSan:

## 1. The integration of the sanitizer

Similar to ASan and TSan, we propose adding RTSan as an unstable feature. Enabling RTSan will be done via the same method.

```
RUSTFLAGS=-Zsanitizer=realtime cargo build
```

Much of the heavy lifting in this tool is done in the LLVM IR and runtime library, so the changes to `rustc` front-end should be light. 

## 2. The addition two new attributes to rust - `#[nonblocking]` `#[blocking]`

`#[nonblocking]` defines a scope as real-time constrained. During this scope, one cannot call any intercepted call (`malloc`, `socket` etc) or call any function marked `#[blocking]`.

`#[blocking]` defines a function as unfit for execution within a `#[nonblocking]` function.

```rust
> cat example/src/main.rs
#[nonblocking]
pub fn process() {
    let audio = vec![1.0; 256]; // allocates memory
}

fn main() {
    process();
}

> cargo run --package example 
==16304==ERROR: RealtimeSanitizer: unsafe-library-call
Intercepted call to real-time unsafe function `malloc` in real-time context!
    #0 0x0001052c5bcc in malloc+0x20 (libclang_rt.rtsan_osx_dynamic.dylib:arm64+0x5bcc)
    #1 0x000104cd7360 in alloc::alloc::alloc::h213dba927a6f8af7 alloc.rs:98
    #2 0x000104cd7478 in alloc::alloc::Global::alloc_impl::h7034d3dd14644937 alloc.rs:181
    #3 0x000104cd7bac in _$LT$alloc..alloc..Global$u20$as$u20$core..alloc..Allocator$GT$::allocate::hd5e7c341a83b5ed4 alloc.rs:241
    ... snip ...
    #11 0x000104cd7d30 in std::sys::backtrace::__rust_begin_short_backtrace::h73bbd1f9991c56fb backtrace.rs:154
    #12 0x000104cd72cc in std::rt::lang_start::_$u7b$$u7b$closure$u7d$$u7d$::hf6a09edd6941dbc1 rt.rs:164
    #13 0x000104cf1958 in std::rt::lang_start_internal::h9e88109c8deb8787+0x324 (example:arm64+0x10001d958)
    #14 0x000104cd7298 in std::rt::lang_start::ha4c268826019738b rt.rs:163
    #15 0x000104cd9658 in main+0x20 (example:arm64+0x100005658)
    #16 0x00018d8860dc  (<unknown module>)
    #17 0xf062fffffffffffc  (<unknown module>)

SUMMARY: RealtimeSanitizer: unsafe-library-call alloc.rs:98 in alloc::alloc::alloc::h213dba927a6f8af7
fish: Job 1, 'cargo run --package example --f…' terminated by signal SIGABRT (Abort)
```

This previous example shows that the interceptors written for the RealtimeSanitizer runtime are mostly shared across Rust and C/C++. Rust calls into libc `malloc` for basic allocation operations  so it is automatically intercepted with no additional changes to the Rust version. A vast majority of the unsafe behavior that RTSan detects will be detected in this way.

Users may also mark their own functions as unfit for a `nonblocking` context with the `#[blocking]` attribute, as seen below. This allows for detection of calls that do not result in a system call, but may be non-deterministically delayed.

```rust
#[blocking]
fn spin() {
    loop {}
}


#[nonblocking]
fn process() {
    spin();
}

fn main() {
    process();
}
> cargo run --package example --features rtsan
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.05s
     Running `target/debug/example`
==16364==ERROR: RealtimeSanitizer: blocking-call
Call to blocking function `spin` in real-time context!
    #0 0x0001021a6940 in rtsan::notify_blocking_call::ha111b6bcc1e1c566+0x24 (example:arm64+0x100002940)
    #1 0x0001021a6864 in example::spin::h3f7c52e56f3fcffa main.rs:1
    #2 0x0001021a6880 in example::process::hace23451997fd8f2 main.rs:9
    #3 0x0001021a6840 in example::main::h700c0ae2eb60a977 main.rs:16
    #4 0x0001021a670c in core::ops::function::FnOnce::call_once::hd7f394a2ba7ce532 function.rs:250
    #5 0x0001021a6824 in std::sys::backtrace::__rust_begin_short_backtrace::h73bbd1f9991c56fb backtrace.rs:154
    #6 0x0001021a67ec in std::rt::lang_start::_$u7b$$u7b$closure$u7d$$u7d$::hf6a09edd6941dbc1 rt.rs:164
    #7 0x0001021bebbc in std::rt::lang_start_internal::h9e88109c8deb8787+0x324 (example:arm64+0x10001abbc)
    #8 0x0001021a67b8 in std::rt::lang_start::ha4c268826019738b rt.rs:163
    #9 0x0001021a68b4 in main+0x20 (example:arm64+0x1000028b4)
    #10 0x00018d8860dc  (<unknown module>)
    #11 0x4c3d7ffffffffffc  (<unknown module>)

SUMMARY: RealtimeSanitizer: blocking-call (example:arm64+0x100002940) in rtsan::notify_blocking_call::ha111b6bcc1e1c566+0x24
fish: Job 1, 'cargo run --package example --f…' terminated by signal SIGABRT (Abort)

```

## 3. The addition of `blocking` to `std::sync::Mutex::lock` and `std::sync::RwLock::lock`

As stated in the previous section, many of the interceptors "just work". Often, the Rust runtime will call into the libc runtime to allocate memory (`malloc` et al), interface with the networking stack (`socket` et al) or do I/O (`read`, `write`, `open`, `close` et al).

A couple places where this is not the case is `std::sync::Mutex::lock` and `std::sync::RwLock::lock`. Locks are disallowed in real-time contexts but because these methods do not call in to `pthread_mutex_lock` initially, the RTsan runtime cannot automatically detect its usage.

To fix this, we propose adding the `#[blocking]` attribute to this method in the rust standard library. 

With these three points integrated, RTSan would be functional and available for use in the Rust ecosystem.

## 4. The addition of the `rtsan_scoped_disabler!` macro

It will be important to allow users to opt-out of rtsan detection for a specific scope. This may be useful if the end user thinks RTsan has a false positive, or it happens in third-party code they don't control.

We propose addition of the `rtsan_scoped_disabler!` macro:
```rust
#[nonblocking]
fn process() {
    rtsan_scoped_disabler!(
        let audio = vec![1.0; 256]; // report is suppressed
    )
}
```

When rtsan is not enabled, this will become a no-op, so it will be safe for users to leave in their code. 

### Run-time suppression list
There is one other way to opt-out of rtsan checking which will automatically work with Rust.

Users may specify suppression lists, which may be passed in via an environment variable
```
> cat suppressions.supp
call-stack-contains:*spin*
> RTSAN_OPTIONS=suppressions=suppressions.supp cargo run
```

### `no_sanitize`
Another approach we could take is similar to the ASan and TSan `no_sanitize` attribute. We advocate for the scoped disabler macro, as it allows users to specify a more specific scope to disable the tool in. This means users will not have to extract unsafe code into helper functions to disable them at the function level.

To match the other sanitizers, adding in `no_sanitize` could be considered instead of/in addition to the macro, depending on input on this RFC.

# Other considerations

## Supported systems

Currently RTSan works on amd64 and x86_64 processors on Mac and Linux OSs. We are exploring supporting Windows, and there is no limitation (other than dev resources) to supporting more processors. If this support gets added in the future, Rust would again get it "for free".

## Additional functions marked `#[blocking]`

There are presumably more functions we should mark blocking in the Rust standard library, but we propose starting small and giving it a try with `std::sync::Mutex::lock`. It is the hope that most of the libc interceptors will cover most other use cases and we can add them as necessary.

## The names of the attributes

The naming of the attributes (nonblocking, blocking) match what they are in `clang`. While we **could** change them, we want to initially propose sticking to the precedent set there. This would reduce user confusion, and prevent having to translate documentation.

## Other features of the run-time library
As discussed with the suppression list example above, there are a number of other runtime options the RTSan run-time library supports. We recommend checking out the [official docs](https://clang.llvm.org/docs/RealtimeSanitizer.html) for more examples.

## Future work
Much of the future work will come in the RTSan runtime library, which Rust will get "for free". This includes adding more interceptors, improved diagnostics and any other future features. 

## Support

We have seen a solid amount of industry support from big companies in:
1. Audio (in Rust and C++)
2. Aerospace (in Rust and C++)
3. Robotics
4. Embedded
5. Autonomous vehicles

In presenting this tool at CppCon and AudioDevCon, we have been asked "When is it coming to Rust?" more times than I can remember. 

# Open questions
* Should we support `no_sanitize`, `rtsan_scoped_disabler`, or both?
* Are there other methods in the standard library deserving of the `#[blocking]` attribute?

# Conclusion

We hope to get approval for integrating RealtimeSanitizer into the Rust ecosystem. With this tool accessible, writing real-time safe code in Rust will be safer and easier than ever before. Looking forward to any feedback and discussions.

Thanks for considering,
Chris, David, and Stephan
