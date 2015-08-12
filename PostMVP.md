# Essential Post-MVP Features

Some features are known to be essential and needed as soon as possible but aren't
in the [Minimum Viable Product (MVP)](MVP.md) because there isn't yet a
portably-efficient [polyfill](Polyfill.md) via JavaScript. There is a much
bigger [list of features](FutureFeatures.md) that will be added after these
essential features.

Post-MVP features will be available under [feature tests](FeatureTest.md).

## Threads

See [Threads.md](Threads.md).

## Fixed-width SIMD

Support fixed-width SIMD vectors, initially only for 128-bit wide vectors as
demonstrated in [PNaCl's SIMD][] and [SIMD.js][].

SIMD adds new primitive variable and expression types (e.g., `float32x4`) so it
has to be part of the core semantics. SIMD operations (e.g., `float32x4.add`)
could be either builtin operations (no different from `int32.add`) or exports of
a builtin SIMD module.

  [PNaCl's SIMD]: https://developer.chrome.com/native-client/reference/pnacl-c-cpp-language-support#portable-simd-vectors
  [SIMD.js]: https://github.com/tc39/ecmascript_simd

## Zero-cost Exception Handling

The WebAssembly MVP (compilers and polyfills) may support four no-exception
modes for C++:
* Compiler transforms `throw` to `abort()`.
* Compiler-enforced `-fno-exceptions` mode (note [caveats][]).
* Compiler conversion of exceptions to branching at all callsites.
* In a Web environment exception handling can be emulated using JavaScript
  exception handling, which can provide correct semantics but isn't fast.

These modes are suboptimal for code bases which rely on C++ exception handling,
but are perfectly acceptable for C code, or for C++ code which avoids
exceptions. This doesn't prevent developers from using the C++ standard library:
their code will function correctly (albeit slower at times) as long as it
doesn't encounter exceptional cases.

Post-MVP, WebAssembly will gain support for developer access to stack unwinding,
inspection, and limited manipulation. These are critical to supporting zero-cost
exception handling by exposing [low-level capabilities][].

In turn, stack unwinding, inspection, and limited manipulation will be used to
implement `setjmp`/`longjmp`. This can enable all of the defined behavior of
`setjmp`/`longjmp`, namely unwinding the stack without calling C++
destructors. It does not, however, allow the undefined behavior case of jumping
forward to a stack that was already unwound which is sometimes used to implement
coroutines. Coroutine support is being
[considered separately](FutureFeatures.md#coroutines).

  [caveats]: https://blog.mozilla.org/nnethercote/2011/01/18/the-dangers-of-fno-exceptions
  [low-level capabilities]: https://extensiblewebmanifesto.org
