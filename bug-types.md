This file is meant to settle and document how the Gato programming language will eliminate or mitigate as many types of bugs and vulnerabilities as possible. I used the [Wikipedia article on software bugs](https://en.wikipedia.org/wiki/Software_bug#Types) as a starting point to build this list of bug types, and I've included links to dedicated Wikipedia articles in most section titles. If you see something wrong or missing, please [open an issue](https://github.com/gato-lang/gato/issues/new) or pull request.

## [Stack overflow](https://en.wikipedia.org/wiki/Stack_overflow)

Resources to understand the stack:
- [Kernel Stack and User Space Stack](https://www.baeldung.com/linux/kernel-stack-and-user-space-stack)
- [Where is the stack memory allocated from for a Linux process?](https://stackoverflow.com/questions/17671423/where-is-the-stack-memory-allocated-from-for-a-linux-process)
- `RLIMIT_STACK` in `man 2 setrlimit` ([Linux](https://man.archlinux.org/man/core/man-pages/setrlimit.2.en), [OpenBSD](https://man.openbsd.org/setrlimit.2), [NetBSD](https://man.netbsd.org/setrlimit.2))
- [`man 3 alloca`](https://man.archlinux.org/man/alloca.3.en)

Vulnerabilities to prevent:
- [The Stack Clash](https://www.openwall.com/lists/oss-security/2017/06/19/1)

Existing solutions: I don't know of any language which completely solves this problem. According to its wiki, [Haskell can overflow in non-obvious ways](https://wiki.haskell.org/Stack_overflow). Interpreted programs in languages like JavaScript and Python typically can't overflow the stack allocated by the OS, but only because the interpreter enforces a recursion depth limit ([Python allows changing the limit, but this is unsafe](https://docs.python.org/3/library/sys.html#sys.setrecursionlimit)). Rust doesn't seem to have any protection against stack overflows, it's easy to trigger one in the [Rust Playground](https://play.rust-lang.org/). At the time I'm writing this, Zig hasn't yet implemented [safe recursion](https://github.com/ziglang/zig/issues/1006).

Envisioned solutions:
- The compiler is in charge of preventing a stack overflow. During compilation, it keeps track of a program's call graph (i.e. which functions call which functions), taking into account that functions can have [callbacks](https://en.wikipedia.org/wiki/Callback_\(computer_programming\)). This enables it to detect cycles in the graph (both direct and indirect recursions), and automatically prevent them from resulting in a stack overflow, either by removing the cycles entirely (i.e. the calls between the functions in a *compiled* module would form a [directed acyclic graph](https://en.wikipedia.org/wiki/Directed_acyclic_graph)) or by injecting mitigations into the compiled code at any point where a cycle begins (at a small performance cost). Having all the necessary information, the compiler injects instructions to ask the operating system for the amount of stack space a program needs when it's launched and when it enters recursions (if they haven't been removed). If the necessary stack space can't be obtained, a program exits with a proper error code and message.
- Calling a library written in an unsafe language automatically happens in a subprocess. [`sigaltstack`](https://man.archlinux.org/man/sigaltstack.3p.en) is used to handle stack overflows in such subprocesses.

## Unintentionally [infinite loops](https://en.wikipedia.org/wiki/Infinite_loops)

Examples:
- [Cloudflare hit an unintentionally infinite loop during the rewrite of an NGINX module in Rust](https://blog.cloudflare.com/rust-nginx-module/#challenges-encountered). (The code sample published by Cloudflare contains an `unsafe` block, but unintentionally infinite loops can also happen in “safe” Rust code.)
- The [`aiohttp` Python module had a Denial of Service vulnerability involving unintentionally infinite looping](https://github.com/aio-libs/aiohttp/security/advisories/GHSA-5m98-qgg9-wh84).

Envisioned solutions:
- Encourage explicit limits on loops, through syntactic sugar that makes it easy to define the maximum number of iterations and/or maximum run time of a loop.
- If Gato ends up getting a `while` loop keyword, the compiler only allows its use if it can prove that the loop will end.
- Looping through recursive function calls is only allowed if the compiler can prove that the call chain will end and won't result in an unhandled exception due to a stack overflow.
- Calling a function written in an unsafe language automatically happens in a subprocess which is interrupted if it exceeds its maximum allowed run time.

## Missing or excessively long timeouts

Envisioned mitigation: the maximum run time of a function can be properly declared and easily enforced. In simple cases the maximum run time of a function can be determined during compilation, but more complex programs require keeping track of elapsed time during execution.

## Unintentionally high algorithmic complexity

Examples: [Accidentally Quadratic](https://accidentallyquadratic.tumblr.com/). A particularly tricky one was [Rust hash iteration+reinsertion](https://accidentallyquadratic.tumblr.com/post/153545455987/rust-hash-iteration-reinsertion).

Envisioned mitigation: the language supports declaring the costs of a function. The compiler tries its best to detect inconsistencies between the declared and actual costs.

## [Division by zero](https://en.wikipedia.org/wiki/Division_by_zero) and other nonsensical mathematical operation (e.g. addition of amounts in different currencies)

Envisioned solution:
1. The language supports defining precisely which values a variable is expected to hold, not merely the type of values or how many of them there are. This enables static analysis to detect some bugs that can't be caught by traditional type checkers, including division by zero, because the analyzer knows whether zero is one of the possible values for the divisor, whereas a type checker can only confirm that the divisor is of a compatible numerical type.
2. In addition to precisely defining the values that a single function argument can have, the language supports defining constraints spanning multiple arguments (a.k.a. [contracts](https://en.wikipedia.org/wiki/Design_by_contract)). This enables detecting nonsensical operations through static analysis. Run time enforcement of preconditions is also possible and results in an exception being raised when a precondition is violated.

## [Off-by-one](https://en.wikipedia.org/wiki/Off-by-one_error)

Envisioned mitigation: the index of the first element in a container is 1, not 0. Coupled with previously mentioned language features, this enables detecting at compile time the subset of off-by-one errors that result in out-of-bounds access attempts.

## Arithmetic [overflow](https://en.wikipedia.org/wiki/Integer_overflow) and [underflow](https://en.wikipedia.org/wiki/Arithmetic_underflow)

Envisioned solution: an overflow or underflow is an error by default. Wrapping or saturating must be explicitly requested by the programmer. Syntactic sugar is provided to facilitate that. During static analysis, a computation which can be proven not to overflow or underflow is marked as safe. Unconstrainted numerical types, like Python's `int`, are available for use cases when defining limits is deemed undesirable or unnecessary by the programmer.

## [Rounding errors](https://en.wikipedia.org/wiki/Round-off_error)

Envisioned mitigation: the ability to declare the expected precision of important values, combined with the previously mentioned ability to specify the exact range of values a variable holds, enables static analysis to detect unsafe computations. (I have no expertise or experience in this area, so I might be wrong.)

## [Memory leaks](https://en.wikipedia.org/wiki/Memory_leak)

TODO

## [Handle leaks](https://en.wikipedia.org/wiki/Handle_leak)

TODO

## Uncoordinated parallel computing (non-atomic operations on shared resources without locking, a.k.a. [race condition](https://en.wikipedia.org/wiki/Race_condition))

TODO

## [Deadlock and livelock](https://en.wikipedia.org/wiki/Deadlock)

TODO

## [Timing attacks](https://en.wikipedia.org/wiki/Timing_attack)

Envisioned mitigation: the language supports declaring that a function's running time must not depend on the values of specific variables. The compiler does all it can to enforce that requirement, which includes trying to prevent the CPU from executing some instructions on the secrets in variable time.

## [Code injection](https://en.wikipedia.org/wiki/Code_injection)

TODO

## [ANSI terminal vulnerabilities](https://dgl.cx/2023/09/ansi-terminal-security)

Envisioned mitigation: refuse to print legacy control characters when writing to a terminal, except through functions designed to do it in a safe way. This should prevent programs written in Gato from being abused to transmit an exploit to a terminal.
