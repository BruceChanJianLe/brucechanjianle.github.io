---
title: POSIX Threads (where they came from, and whether you still need them)
author: Chan Jian Le
date: 2026-09-08 23:15:00 +0800
categories: [Programming]
tags: [programming, cpp, linux]
---

> A short tour of why `pthread_create` looks the way it does, why C++ eventually
> grew its own threading library instead of just wrapping it, and how to decide
> which of the two you should be writing today.

Every so often I open a header and find `<pthread.h>` staring back at me in a
C++ file, and have to ask whether that is deliberate or just inherited. It is a
fair question, and the answer makes a lot more sense once you know where the
API came from.

## A World Before The Standard

By the late 1980s, running several threads inside one process was not research
anymore. It was shipping. The problem was that everybody shipped a *different*
version of it.

Sun had its own lightweight process library. Mach had C-threads. DEC and the
Open Software Foundation had DCE threads, used widely enough that plenty of
production code depended on them. Windows NT turned up with a thread API that
looked nothing like any of the Unix ones. Every library had its own spelling
for "start a thread", its own idea of what a mutex was, and its own opinion on
what should happen to a thread that returned from its entry function.

If you wrote portable server software, this was miserable. Porting a threaded
application between two Unix vendors often meant rewriting every single
synchronisation call in it.

The IEEE POSIX committee took this on as 1003.4a, part of the broader real-time
extensions effort. It took years, and the draft moved a lot along the way. The
result was ratified in 1995 as **IEEE Std 1003.1c-1995** and folded into
POSIX.1. That is where `pthread_create`, `pthread_mutex_t`, condition
variables, thread-specific data and the rest of the vocabulary come from.  

The `p` is for POSIX.  

## Linux Takes Two Runs At It

Having a standard is not the same as having an implementation.  

Linux's first serious attempt was **LinuxThreads**, written by Xavier Leroy
around 1996. It built threads out of processes created with `clone()` sharing
an address space, plus a hidden manager thread to keep the bookkeeping
straight. It worked, and it carried Linux through years of growing server
workloads, but it was never really conformant. Each thread had its own process
ID, so `getpid()` returned different values inside one program. Signal handling
ignored the standard's rules about process-wide delivery. The manager thread
was a single point of failure, and creating a thread was slow.

Two replacements competed. IBM's **NGPT** used an M:N model, mapping many
user-level threads onto fewer kernel threads. Ulrich Drepper and Ingo Molnar's
**NPTL** took the simpler 1:1 route, one kernel task per thread, and pushed the
hard parts down into new kernel primitives, most importantly the futex.

NPTL won. It shipped with the 2.6 kernel and glibc 2.3.2 around 2003, and it is
what you are using right now. Solaris went through the same arc, starting M:N
and moving to 1:1 in Solaris 9.

The lesson that stuck: fast uncontended locking in user space plus a cheap
kernel wait primitive beats clever user-level scheduling for most real
workloads.

## Why C++ Needed Its Own Answer

For a long time the C++ answer to threading was "use pthreads, or use
Boost.Thread, which wraps pthreads". That was uncomfortable, and it took a
while to articulate exactly why.

In 2004 Hans Boehm published a paper with a deliberately provocative title:
*Threads Cannot Be Implemented As a Library*. The argument is that a threading
library sitting on top of a language whose memory model says nothing about
concurrency is standing on sand. The compiler is free to reorder, cache and
duplicate memory accesses in ways that are perfectly legal for single-threaded
code and catastrophic for multithreaded code. No amount of care *inside* the
library can fix that, because the compiler does not know the library is
special.

The response was C++11, and it brought two things rather than one:

1. A **memory model** that finally defines what it means for two threads to
   touch the same object, with `std::atomic` and a formal happens-before
   relation.
2. A **library**: `<thread>`, `<mutex>`, `<condition_variable>`, `<future>`,
   `<atomic>`.

The second half gets all the attention, but the first half is the one that
actually answers Boehm. On Linux, `std::thread` is generally implemented on top
of pthreads anyway, so this is a layer, not a replacement.

## The Same Program, Twice

Summing two halves of a range. First in C with pthreads:

```c
#include <pthread.h>
#include <stdio.h>

struct work { int lo, hi; long sum; };

static void *worker(void *arg) {
    struct work *w = arg;
    w->sum = 0;
    for (int i = w->lo; i < w->hi; i++) w->sum += i;
    return NULL;
}

int main(void) {
    pthread_t t1, t2;
    struct work a = { 0, 500, 0 };
    struct work b = { 500, 1000, 0 };

    if (pthread_create(&t1, NULL, worker, &a) != 0) return 1;
    if (pthread_create(&t2, NULL, worker, &b) != 0) return 1;

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    printf("%ld\n", a.sum + b.sum);
    return 0;
}
```

Look at the shape of it. Arguments go in and results come out through a
`void *`, so you invent a struct and hand-manage its lifetime. Forget the
`pthread_join` and you leak the thread's resources.

Now the same thing in modern C++:

```cpp
#include <future>
#include <iostream>

long sum_range(int lo, int hi) {
    long s = 0;
    for (int i = lo; i < hi; ++i) s += i;
    return s;
}

int main() {
    auto a = std::async(std::launch::async, sum_range, 0, 500);
    auto b = std::async(std::launch::async, sum_range, 500, 1000);
    std::cout << a.get() + b.get() << '\n';
}
```

Types are checked, the result is a real `long`, and there is nothing to cast.

Locking shows it just as clearly. With pthreads you pair every lock with an
unlock and hope nothing sneaks out in between:

```c
pthread_mutex_lock(&m);
/* if anything returns early here, the mutex stays locked forever */
pthread_mutex_unlock(&m);
```

In C++ the scope guard handles it, and it survives exceptions:

```cpp
std::mutex m;
{
    std::lock_guard lock(m);
    // unlocked automatically on every exit path
}
```

C++20 then added `std::jthread`, which is the one you usually want. It joins in
its destructor, so forgetting to join stops being a bug, and it carries
cooperative cancellation through `std::stop_token`:

```cpp
std::jthread t([](std::stop_token stop) {
    while (!stop.stop_requested()) {
        do_some_work();
    }
});
// destructor requests the stop, then joins
```

## So Which One Should You Use

If you are writing C++, use the standard library. `std::jthread` or
`std::async` by default, `std::thread` if you are stuck on C++11 or C++14, with
`std::mutex`, `std::condition_variable` and `std::atomic` alongside. The
reasons are not stylistic:

- Type-safe, so no `void *` round trips.
- Exception-safe, because RAII guards release locks and join threads while
  unwinding.
- Portable to Windows with no emulation layer.
- It interacts correctly with the C++ memory model, which is the entire point
  of the Boehm paper.
- `pthread_cancel` and C++ do not mix well. Asynchronous cancellation can tear
  a thread down at a point where your invariants do not hold. `std::stop_token`
  cancels at points *you* choose.

Reach for pthreads directly when one of these applies:

- You are writing C, not C++.
- You are on an old or embedded toolchain where the C++ threading support is
  absent or unreliable.
- You need something POSIX exposes and the standard does not: stack size or
  guard pages via `pthread_attr_t`, CPU affinity, real-time scheduling policies
  and priorities, `pthread_atfork`, or robust mutexes.

That last case is not all-or-nothing. `std::thread` and `std::jthread` both
expose `native_handle()`, so you can keep the standard API and still drop down
for the platform-specific bits:

```cpp
std::jthread t(worker);
pthread_setname_np(t.native_handle(), "worker");
```

ATTENTION: what you should *not* do is mix synchronisation primitives carelessly
across the two worlds. Locking a `pthread_mutex_t` with pthread calls while a
`std::condition_variable` waits on a `std::mutex` guarding the same data is an
excellent way to build something that works on your machine and fails in
production.

## Summary

- POSIX threads exist because the early 1990s had too many incompatible
  threading libraries, and the 1995 standard picked one vocabulary for Unix to
  share
- NPTL replaced LinuxThreads in 2003, and 1:1 beat M:N because cheap futexes
  beat clever user-level scheduling
- C++11 needed a memory model, not just an API - a threading library alone
  cannot save you from the compiler
- Write against `<thread>`, and keep `<pthread.h>` for the corners the standard
  does not reach
- `native_handle()` is the sanctioned escape hatch, so it is rarely an
  either/or decision

It is still a C API from 1995, and it shows. But it is also the substrate that
almost everything on Linux runs on, including your C++ standard library. Worth
knowing which layer you are standing on when something goes wrong.

Until next time, keep learning and growing!

## References

- [Threads Cannot Be Implemented As a Library](https://www.hpl.hp.com/techreports/2004/HPL-2004-209.pdf)
- [The Native POSIX Thread Library for Linux](https://akkadia.org/drepper/nptl-design.pdf)
- [pthreads(7)](https://man7.org/linux/man-pages/man7/pthreads.7.html)
- [std::jthread](https://en.cppreference.com/w/cpp/thread/jthread)
