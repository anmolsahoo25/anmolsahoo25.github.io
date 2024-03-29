---
title: "ThreadSanitizer + OCaml"
date: 2022-01-01T00:00:00-05:00
draft: false
summary: "I discuss how I tried to instrument the OCaml compiler with
the LLVM ThreadSanitizer race detector. This helps detect races in OCaml
code that would alert end-users to properly synchnorize their code"
---

Races! A word that programmers need to avoid while writing concurrent code.
Thankfully, we have ThreadSanitizer — a race detector developed as
part of LLVM. Now with OCaml on its way to getting multicore support, I thought
it would be a good time to see how we could integrate ThreadSanitizer with the
OCaml compiler. This post would should be a fun trip through parallelism in
OCaml, the OCaml compiler backend, ThreadSanitizer and catching those pesky races!

Code at - [Github repository](https://github.com/anmolsahoo25/ocaml/tree/multicore-pr+tsan)

#### Parallelism in OCaml

For those of you completely new to this world, OCaml is a strongly-typed,
multi-paradigm programming language, of which I enjoy the functional aspects
the most!

Recently, OCaml has been getting close to shared-memory parallelism support.
Put very simply, multicore OCaml will add support to spawn parallel threads
called `Domains` which are mapped to native system threads. The garbage
collector has also been modified to allow these threads to execute in parallel
(with some parts being performed in a stop-the-world fashion). The multicore
OCaml changes and features go well beyond this, but for this blog post, this
should suffice.

A big shoutout to everyone working so hard to make multicore OCaml a reality!

#### Races in OCaml

Well, now that OCaml has parallelism, it also has races. Let's see how this
could manifest in practice - 

```ocaml
type t = { mutable x : int }

let _ =
  let v = { x = 10 } in
  let t1 = Domain.spawn(fun () -> v.x <- 0) in
  let t2 = Domain.spawn(fun () -> v.x) in
  List.map Domain.join [t1; t2]
```

The one above is a very simple example but a good example of a common race.
Mutable fields are one place where races can arise. Other places could be
arrays, C FFI functions and others. We want to detect and avoid such races,
because this could lead to unexpected values in the code!

#### ThreadSanitizer
ThreadSanitizer is a dynamic race detector developed as part of the LLVM library.
It consists of a runtime library which tracks accesses and reports races if
detected and a compiler pass that instruments C/C++ code with the necessary
function calls to signal accesses to the runtime library. It has additional features
that allows detecting misuse of locks as well. You can read more about
ThreadSanitizer on these links - [LLVM Docs](https://clang.llvm.org/docs/ThreadSanitizer.html),
[Github Wiki](https://github.com/google/sanitizers/wiki/ThreadSanitizerCppManual),
[Paper](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/35604.pdf).

For our discussion, it would suffice to know what is needed to instrument OCaml
generated code. ThreadSanitizer requires a call to `__tsan_func_entry` and
`__tsan_func_exit` on function entry and exit respectively. Then, it has functions
called `__tsan_readN` and `__tsan_writeN` which should be called after every
memory access.

In the context of OCaml, there is an extra consideration we need to keep in
mind. Since the OCaml runtime uses a `pthread_mutex` to synchnorize certain
activities such as spawning domains, these would lead to false synchronization
events. Thus races would actually be missed. The solution to this is to use a
ThreadSanitizer feature called `suppressions`, which allows us to suppress
events. Note that we still need to instrument the runtime library to maintain
correct stack frames, allocations, thread creation and racy accesses using
`caml_modify`, but we will suppress the `caml_plat_lock` function to ignore
synchronization events generated by access to the mutex.

#### Instrumenting the OCaml compiler

This section presumes some knowledge of the OCaml compiler internals, but I
will try my best to explain so that everyone can follow along. OCaml code goes
through a bunch of intermediate representations before being emitted as machine
code. Each representation removes a few high-level abstractions and brings it
closer to assembly. For example `Clambda` is a high-level representation with
functional calls, function abstraction and other such elements whereas `Mach`
is a low-level representation which basically reprensents a list of machine
instructions.

One of these representations is `Cmm` where we will implement our
instrumentation pass. `Cmm` terms represent program
expressions, so we can easily instrument the terms without having to worry
about low level details such as register allocation or instruction selection.
At the same time, `Cmm` has access to information such as code symbols,
external function calls, memory allocations etc which is what we require for
instrumenting calls to ThreadSanitizer methods.

A variant of the `Cmm` type is this - 

```ocaml
type expression =
  ...
  | Clet of Backend_var.With_provenance.t * expression * expression
  ...
```

This represents a term such as `let x = 10 in x + 5`.
For example, the above expression could be represented as (the following is not
valid OCaml code, it is just for representation) - 

```
Clet (x, Cconst_int 10, Cop(Caddi, [x,Cconst_int 5]))
```

The operations we are interested in are called `Cload` and `Cstore` -

```ocaml
| Cload of { memory_chunk: memory_chunk ; mutability: Asttypes.mutable_flag ; is_atomic: bool }
| Cstore of memory_chunk * Lambda.initialization_or_assignment
```

We would like to call the necessary `__tsan_read/writeN` functions after these
operations. Thus our instrumentation pass takes a `Cmm` expression and recurses
through the structure, and adds a call to the correct ThreadSanitizer function
after memory accesses. The `memory_chunk` parameter denotes the size of the access
and the argument to the operation is the memory address in question. There are the
two pieces of information we need for calling ThreadSanitizer functions. 

There are a few things we need to keep in mind which are particular to OCaml.
Loads in OCaml are marked to denote if they are to mutable fields or not. Thus,
we only need to instrument mutable loads, as there will not be any stores to
immutable fields. This should reduce the overhead of instrumentation.

Also, for stores we only need to instrument assignment stores, as
initialization stores have a barrier to ensure that they are visible to all
threads in a synchronized fashion.

Thanks to [KC](https://kcsrk.info) for pointing this out!

#### Experiments
All right! Let's see if we can finally detect races now. A very simple example -

```ocaml
type t = { mutable x : int }

let v = { x = 0 }

let _ =
  let t1 = Domain.spawn(fun () -> v.x <- 10; Unix.sleep 1000) in
  let t2 = Domain.spawn(fun () -> v.x; Unix.sleep 1000) in
  List.map Domain.join [t1; t2]
```

Compiling and running this returns -

```
WARNING: ThreadSanitizer: data race (pid=8727)
  Read of size 8 at 0x7f40001ff7c8 by thread T4:
    #0 camlMain__fun_596 <null> (a.out+0x4e21cd)
    #1 <null> <null> (a.out+0x4e218f)
    #2 caml_callback /home/anmol/work/ocaml/ocaml/runtime/callback.c:251:34 (libasmrun_shared.so+0x2ffee)
    #3 domain_thread_func /home/anmol/work/ocaml/ocaml/runtime/domain.c:841:5 (libasmrun_shared.so+0x3b7b6)

  Previous write of size 8 at 0x7f40001ff7c8 by thread T1:
    #0 camlMain__fun_592 <null> (a.out+0x4e214d)
    #1 <null> <null> (a.out+0x4e210f)
    #2 caml_callback /home/anmol/work/ocaml/ocaml/runtime/callback.c:251:34 (libasmrun_shared.so+0x2ffee)
    #3 domain_thread_func /home/anmol/work/ocaml/ocaml/runtime/domain.c:841:5 (libasmrun_shared.so+0x3b7b6)

  Thread T4 (tid=8732, running) created by main thread at:
    #0 pthread_create <null> (a.out+0x44e37b)
    #1 caml_domain_spawn /home/anmol/work/ocaml/ocaml/runtime/domain.c:885:9 (libasmrun_shared.so+0x3b538)
    ...
    #14 main /home/anmol/work/ocaml/ocaml/runtime/main.c:37:3 (libasmrun_shared.so+0x16c75)

  Thread T1 (tid=8729, running) created by main thread at:
    #0 pthread_create <null> (a.out+0x44e37b)
    #1 caml_domain_spawn /home/anmol/work/ocaml/ocaml/runtime/domain.c:885:9 (libasmrun_shared.so+0x3b538)
    ...
    #14 main /home/anmol/work/ocaml/ocaml/runtime/main.c:37:3 (libasmrun_shared.so+0x16c75)

SUMMARY: ThreadSanitizer: data race (/home/anmol/work/ocaml/test/a.out+0x4e21cd) in camlMain__fun_596
```

Let's check another one -

```ocaml
let _ =
  let v = Array.make 4 0 in
  let t1 = Domain.spawn (fun () -> Array.set v 0 0; Unix.sleep 1000) in
  let t2 = Domain.spawn (fun () -> Array.get v 0; Unix.sleep 1000) in
  List.map Domain.join [t1; t2]
```

Similarly, this should print a report denoting a race. Phew, we are able to
now detect races!

_You might be wondering about the `Unix.sleep`. Since the functions passed
to `Domain.spawn` are short-lived, the thread terminates before the racy access
occurs, thus we keep the thread alive with the sleep call. In larger programs,
this should not be necessary._

#### Future work

There is still work left to do on this project. Most importantly, I need to
figure out how to handle compactions. When a compaction happens, objects on the
major heap are moved around. Thus, a race that occurs on a certain location of
an OCaml object will signal ThreadSanitizer with two different addresses before
and after the compaction. This would lead to the race not being reported.

Another part that I left out is the instrumentation of atomic variables. ThreadSanitizer
has support for instrumenting accesses as atomic using memory annotations from
the C/C++ memory model, such as `memory_order_relaxed` and `memory_order_acquire/release`.
In OCaml, all reads and writes to atomic variables are considered to be
`memory_order_acquire` and `memory_order_release` respectively. That should be
straightforward to implement.

Finally, it would be good to run the instrumented compiler on some large projects
such as Lwt. That would be a good test to see how the tool performs in the wild.

#### Acknowledgements
Thanks to Dmitry Vyukov on the ThreadSanitizer Google mailing list for
suggesting to use suppressions.
