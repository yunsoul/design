# Threads

After the [MVP](MVP.md), to realize the [high-level goal](HighLevelGoals.md)
of allowing native speed and exposing hardware capabilities, WebAssembly
will provide shared-memory multi-threading.

This document is organized into sections describing:
* the basic [linear memory model](Threads.md#linear-memory-model);
* [pure WebAssembly threads](Threads.md#pure-webassembly-threads);
* [host environment threads](Threads.md#host-environment-threads);
* [Web API interactions](Threads.md#web-api-interactions); and
* a potential [further extension](Threads.md#shared-objects) to allow 
  sharing non-linear memory.

## Linear Memory Model

WebAssembly will provide the low-level buildings blocks for pthreads-style
shared memory: shared memory between threads, atomics and futexes (or
[synchronic][]).

New atomic memory operations, including loads/stores annotated with their atomic
ordering property, will follow the [C++11 memory model][], similarly to the
[PNaCl atomic support][] and the [SharedArrayBuffer][] proposal. Regular loads
and stores will be bound by a happens-before relationship to atomic operations
in the same thread of execution, which themselves synchronize-with atomics in
other threads. Following these rules, regular load/store operations can still be
elided, duplicated, and split up. This guarantees that data-race free code
executes as if it were sequentially consistent. Even when there are data races,
WebAssembly will ensure that the [nondeterminism](Nondeterminism.md) remains
limited and local.

Modules can have global variables that are either shared or thread-local. While
the linear memory could be used to store shared global variables, global
variables are not aliasable and thus allow more aggressive optimization.

  [synchronic]: http://wg21.link/n4195
  [C++11 memory model]: http://www.hboehm.info/c++mm/
  [PNaCl atomic support]: https://developer.chrome.com/native-client/reference/pnacl-c-cpp-language-support#memory-model-and-atomics
  [SharedArrayBuffer]: https://docs.google.com/document/d/1NDGA_gZJ7M7w1Bh8S0AoDyEqwDdRh4uSoTPSNn77PFk

## Pure WebAssembly Threads

Primitive operations would be added for creating and joining threads. Support
for [coroutines](FutureFeatures.md#coroutines) could later include
stack-management operations.

These operations would allow a WebAssembly module to portably create and use
threads on any host environment. Another intended benefit of pure WebAssembly
threads is that an implementation would be able to easily allocate the minimum
resources necessary (a kernel or green thread) without any of the resources
otherwise necessary to support a full 
[host environment thread](Threads.md#host-environment-thread).

## Host Environment Threads

Already in the MVP, WebAssembly [leaves the invocation of 
module exports up to the host environment](Modules.md#imports-and-exports).
In particular, the host determines when and how the exports are called. With
threads, the host would additionally determine on which host thread each export
invocation occurred. A "host thread" is assumed to be equivalent to a pure
thread with "extra stuff" that is created outside the purview of the WebAssembly
spec.

Semantically, this means that the WebAssembly spec would receive a
"host thread id" along with every export call. The first time a given
host thread id was used to invoke an export, a new thread-local state record
would be implicitly created and associated with that host thread id. All future
export invocations on the same host thread id would use the same thread-local
state record. Finally, this host thread would not be joinable; when the host
environment destroyed a host thread, it would garbage collect the associated
thread-local state record.

## Web API interactions

There are two areas integration to consider corresponding to the two types of
threads:
* [host environment threds](Threads.md#web-worker-threads) and
* [pure WebAssembly threads](Threads.md#pure-webassembly-threads-and-imports).

### Web Worker threads

The design of [host environment threads](Threads.md#host-environment-threads)
is intended to allow the host environments to integrate WebAssembly with
preexisting threads. In the Web platform, 
[Web Workers](http://www.w3.org/TR/workers) are the preexisting threads and 
so "host environment thread" would be identified with "Web Worker" on the Web
platform. This provides every host thread a well-defined [execution
context](https://html.spec.whatwg.org/multipage/webappapis.html#calling-scripts)
so that calls to Web APIs from WebAssembly host threads work as if they were
calls from JS.

A key question is how linear memory is shared between Workers since Workers are
completely isolated by design. The proposed solution is to do the same thing as
`SharedArrayBuffer`: share linear memory between workers via `postMessage`. In
the case of WebAssembly, there is no object specifically representing linear
memory&mdash;linear memory is encapsulated by a loaded module&mdash;so the
WebAssembly module itself would be `postMessage`d. This has the added benefit
of giving clear semantic guidance to the implementation to share the generated
machine code between workers to avoid relying on implicit caching logic to
avoid code duplication.

WebAssembly modules can have [imports](Modules.md#imports-and-exports), so
this raises the question of what to do for non-builtin imports when a module
is `postMessage`d. In particular, [ES6 module imports](Modules.md#integration-with-es6-modules)
cannot be shared between workers without introducing fully concurrent semantics
into JS.  The proposed solution is that a shared WebAssembly module re-imports
everything in the target worker. This would produce a fresh and unshared
instance of any ES6 modules in the target Worker, rerunning their top-level
module scripts. ES6 module imports would thus be considered thread-local state.

When a WebAssembly module was received in a target Worker, it would be added
to the Worker's Realm's module registery (which basically maps name to loaded
module). This is the same registry used to "memoize" module loading so that a
diamond import graph creates a diamond instance graph. Leveraging this
memoization, an application can share arbitrary subsets of a loaded module
graph by including multiple modules in the same `postMessage`.

### Pure WebAssembly threads and imports

Unlike the Worker/host threads above, pure WebAssembly threads do not have an
obvious execution context for calls to imported ES6 code or Web APIs: by
design, pure threads do not create a Worker, Realm, Global Object, Execution
Context, or Event Loop and thus have none of the semantic concepts on which ES6
and Web API calls are defined.

First, calls to ES6 module imports from pure WebAssembly threads would be
defined to fault. This avoids introducing concurrent execution into JS and the
need to allocate support JS execution on the pure thread.

To allow Web API calls from pure WebAssembly threads, it is necessary to define
an execution context. One option is to transitively associate a "parent"
host thread with each pure thread. This is well-defined since a module is
necessarily first executed on a host thread (main thread or Worker). Calls to
Web APIs could then be defined to have the same execution context as the 
parent, as if the call was from JS. That being said, for semantic and practical
implementation reasons, Web APIs would likely require an explicit opt-in
attribute (e.g., on the WebIDL interface) to allow invocation from pure threads.

Note that, using the primitive building blocks of shared memory and
synchronization, an application will be able to implement arbitrary synchronous
ES6 and Web API calls from pure threads by proxying to a host thread (albeit
with some latency and contention cost).

## Shared Objects

When [GC](GC.md) and threads are both added to WebAssembly, to avoid
sharing the entire set of host and GC objects (which includes all JS GC
objects) between threads, a practical first step would be to only allow
[reference types](GC.md) in *thread-local* variables. With this constraint,
the GC graph operated on by each (host or pure) WebAssembly thread would
be kept disjoint and thus not significantly different than the current
situation with Workers. This follows from the [design](GC.md#native-gc) of
keeping the GC heap completely disjoint from linear memory.

As demonstrated by popular languages like Java/C#, there are use cases for
shared GC objects though. Even ignoring the use case of compiling languages
with shared GC objects, shared Web API objects would be useful to efficiently
implement shared resources in C/C++. For example, without a shared "mutable
file handle" Web API, an application would need to use a combination of shared
linear memory and a background Worker to shuffle chunks of files into and out
of memory, simulating functionality already provided and heavily optimized by
the OS.

[TODO
* opt-in shareable reference types for globals and cells
* distinguish shareable native GC types which must have shareable fields
]
