---
title: POSIX Threads and SCHED_FIFO (notes from the trenches)
author: Chan Jian Le
date: 2026-09-05 22:04:31 +0800
categories: [CS]
tags: [cs, linux, cpp]
---

> Notes from a small lab on starting a real-time thread. What the standard
> library refuses to do for you, and a bug that compiles clean, returns
> success, and quietly does nothing at all.

Sometimes you need a thread that runs *on time* rather than *eventually*.
Control loops, audio callbacks, anything driving hardware on a fixed tick.

That means `SCHED_FIFO`. And the moment you go looking for how to set it from
C++, you find the standard library has quietly left the room.

## Scope: This Is A POSIX Story

`SCHED_FIFO` comes from POSIX.1b, the real-time extensions. So this is Unix
territory - Linux, the BSDs, QNX, Solaris. Windows is not POSIX and has no
`SCHED_FIFO`; its nearest relative is `REALTIME_PRIORITY_CLASS` plus
`THREAD_PRIORITY_TIME_CRITICAL`, which is a different model wearing similar
clothes.

Even within POSIX it is *optional* - ask `sysconf(_SC_THREAD_PRIORITY_SCHEDULING)`.
And the tooling in section 6, plus `CAP_SYS_NICE` and `RLIMIT_RTPRIO`, is
Linux-only even though the API is not.

## 1. Real-Time Means Predictable, Not Fast

`SCHED_FIFO` will not make your code faster. It bounds your **worst case**.

A throughput-tuned system can win on average and still be useless here, because
its worst case is unbounded. A loop that hits its deadline 999 times out of
1000 has not "mostly worked" - it failed once, and depending on what it was
driving, that may be the only number anyone remembers.

We are not buying speed. We are buying a ceiling.

## 2. pthreads Is An API, Not A Language

Awkward truth: policy has to be chosen *when the thread is created*, and most
of our nice modern tools cannot say it.

| API | Policy at creation? |
| --- | --- |
| POSIX `pthread_create` + `pthread_attr_t` | **Yes** |
| `boost::thread::attributes` | Yes, but via `native_handle()` - same POSIX calls underneath |
| `std::thread` / `std::jthread` | **No** - `native_handle()` exists only once the thread is running |
| C11 `thrd_create` | **No** - no scheduling API at all |

This is deliberate, not an oversight. `SCHED_FIFO` is POSIX, Windows is not, so
there was never a portable model to standardise. `native_handle()` is the
escape hatch the committee left us on purpose.

The catch is *when* it opens: only after the thread exists, by which point it
is already running under whatever policy it inherited.

## 3. Two Approaches, And Where Failure Surfaces

**A - `std::jthread`, thread elevates itself:**

```cpp
auto entry = [](std::stop_token stop) noexcept {
  try {
    sched_param param{};
    param.sched_priority = fifo_priority();
    check(pthread_setschedparam(pthread_self(), SCHED_FIFO, &param),
          "pthread_setschedparam");
    worker_loop(std::move(stop), "jthread worker");
  } catch (const std::system_error& e) {
    fmt::print(stderr, "jthread worker: {}\n", e.what());
  }
};

std::jthread worker{entry};   // RAII join, stop_token for free
```

**B - `pthread_create` with an attribute object:**

```cpp
SchedAttr attr;                 // RAII wrapper over pthread_attr_t
attr.set_fifo(fifo_priority());

pthread_t thread{};
check(pthread_create(&thread, attr.get(), &pthread_entry, &args),
      "pthread_create");
```

The usual complaint about A is the latency window - the thread runs briefly at
the inherited policy before elevating. True, and mostly not what bites you.

What matters is **where failure surfaces**:

- **A**: elevation fails, and a thread is already running. Non-RT, doing real
  work, and `main` has no idea. Program looks healthy.
- **B**: policy cannot be applied, so `pthread_create` fails. No thread exists.
  The error lands in `main` at startup, impossible to miss.

`SCHED_FIFO` needs `CAP_SYS_NICE`. Forgetting `sudo` is not an exotic edge
case, it is Tuesday. B turns that into a loud startup failure; A turns it into
a program that runs, prints, looks fine, and misses every deadline.

ATTENTION: A is not wrong. The point is that *"I cannot get real-time
guarantees"* is news you want to be incapable of missing, and only one of these
designs gives you that.

Best of both: **`jthread` for lifetime, pthreads for scheduling.** You keep
cooperative cancellation either way - `stop_token` works fine with raw
pthreads, you just pass one in by hand:

```cpp
std::stop_source source;
WorkerArgs args{source.get_token()};
// ...
source.request_stop();
check(pthread_join(thread, nullptr), "pthread_join");
```

## 4. pthreads Return Errors, They Do Not Set errno

Small, easy to miss, and it produces confidently wrong error messages.

pthread functions return the error number directly. So the category is
`std::generic_category()`, not `system_category()`:

```cpp
void check(int rc, const char* what) {
  if (rc != 0) {
    throw std::system_error{rc, std::generic_category(), what};
  }
}
```

Get it wrong and `e.what()` still prints *something* - just from the wrong
error space. Which is exactly why it survives review.

## 5. The Good One: Silent, Plausible Failure

Here is the section I would time-travel to warn myself about.

The hazard in POSIX thread code is not crashing. It is code that **compiles,
returns success, and does something other than what you meant.**

Our lab builds under `-Wall -Wextra -Wpedantic -Wconversion -Wshadow`. Doesn't
help. Watch.

### Constants are not unique

On a stock glibc box:

```console
PTHREAD_INHERIT_SCHED  = 0
PTHREAD_EXPLICIT_SCHED = 1
SCHED_OTHER            = 0
SCHED_FIFO             = 1
SCHED_RR               = 2
```

`PTHREAD_EXPLICIT_SCHED` and `SCHED_FIFO` are **both 1**. `PTHREAD_INHERIT_SCHED`
and `SCHED_OTHER` are **both 0**.

Now suppose you meant `pthread_attr_setschedpolicy` and typed
`pthread_attr_setinheritsched` a second time:

```cpp
pthread_attr_setinheritsched(&attr, PTHREAD_EXPLICIT_SCHED);
pthread_attr_setinheritsched(&attr, SCHED_FIFO);   // meant setschedpolicy
```

The second call hands `1` to a function for which `1` is perfectly valid:

```console
$ gcc -Wall -Wextra -Wpedantic -Wconversion -Wshadow bug.c -o bug
--- compiled clean, zero warnings ---
$ ./bug
both calls returned: rc1=0 rc2=0  (0 == success)
read-back: policy=SCHED_OTHER  inheritsched=EXPLICIT
```

Clean compile. Both calls succeed. Every error check passes. And your thread is
born `SCHED_OTHER` with no real-time guarantee whatsoever, and nothing, anywhere,
mentions it.

### Same family, two more traps

- **Getter where you meant setter.** `pthread_getschedparam` instead of
  `pthread_setschedparam` is valid code, returns `0`, configures nothing. One
  character, no diagnostic.
- **Forgotten `PTHREAD_EXPLICIT_SCHED`.** The attr defaults to
  `PTHREAD_INHERIT_SCHED`, your policy and priority are ignored outright, and
  every setter still returned `0`.

### The lesson

**A successful POSIX call does not mean the intended state was achieved.**

Two defences that actually work:

1. **Keep error checks beside the call they name.** A correct-looking message
   does not prove the correct API was called. Above, the error string said
   `"pthread_attr_setinheritsched"` and it was telling the truth. That *was*
   the bug.
2. **Verify by read-back.** After setting attributes, call the matching getters
   and confirm reality:

```cpp
void verify_fifo(int priority) const {
  int policy{};
  check(pthread_attr_getschedpolicy(&attr_, &policy),
        "pthread_attr_getschedpolicy");
  if (policy != SCHED_FIFO) {
    throw std::logic_error{fmt::format(
        "attr verify: policy is {}, expected SCHED_FIFO", policy_name(policy))};
  }
  // ... same for inheritsched and priority
}
```

That single read-back is the whole difference between catching this in seconds
and losing an afternoon. For anything load-bearing, **set, then verify**.

## 6. Checking It Actually Worked

First trap: `chrt -p` takes a **TID**, not a PID. A process's main thread has
`TID == PID`, so passing a PID silently reports the main thread - usually
`SCHED_OTHER` even when your workers are fine. You will conclude your RT setup
is broken and go rewrite code that was already correct.

What I use:

```bash
ps -Lo pid,tid,cls,rtprio,comm -p $(pgrep -n fifo_demo)
```

`-L` is the whole trick: one row per thread. Without it you see only the main
thread, which is the one thread you do not care about.

### The CLS column

`CLS` is the scheduling class. `cls`, `class`, and `policy` are aliases.

| Code | Policy |
| :--- | :--- |
| `TS` | `SCHED_OTHER` (a.k.a. `SCHED_NORMAL`; CFS/EEVDF) |
| `FF` | `SCHED_FIFO` |
| `RR` | `SCHED_RR` |
| `B` | `SCHED_BATCH` |
| `IDL` | `SCHED_IDLE` |
| `DLN` | `SCHED_DEADLINE` |
| `ISO` | `SCHED_ISO` - in the header, never implemented in mainline |
| `-` | not reported |
| `?` | unknown value |

ATTENTION: `RTPRIO` shows `-` for `TS`, `B`, and `IDL`. That means *no RT
priority at all*, which is not the same as priority zero. `DLN` also shows `-`,
since deadline scheduling is parameterised by runtime/deadline/period instead -
use `chrt -p <tid>` for those.

## Summary

- Real-time buys a bounded worst case, not a lower average
- `std::thread` cannot set policy at creation, and that is a portability
  decision, not an oversight
- `jthread` for lifetime, pthreads for scheduling
- Prefer the design where failure is loud
- `std::generic_category()`, because pthreads return errors instead of setting
  `errno`
- Set, then verify - success is not evidence
- `ps -L`, and `chrt -p` wants a TID

Number five is the one that will actually cost you a day. The compiler, the
warnings, and every return code agreed the code was fine. Only reading the
state back disagreed.

Until next time, keep learning and growing!

## References

- [code](https://github.com/BruceChanJianLe/microstructure/blob/master/practice/cpp/posix-thread-sched-fifo/src/fifo_demo.cpp)
- [pthread_attr_setinheritsched(3)](https://man7.org/linux/man-pages/man3/pthread_attr_setinheritsched.3.html)
- [pthread_attr_setschedpolicy(3)](https://man7.org/linux/man-pages/man3/pthread_attr_setschedpolicy.3.html)
- [sched(7)](https://man7.org/linux/man-pages/man7/sched.7.html)
- [chrt(1)](https://man7.org/linux/man-pages/man1/chrt.1.html)
