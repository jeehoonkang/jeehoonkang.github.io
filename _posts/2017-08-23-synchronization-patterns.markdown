---
layout: post
title:  "Relaxed-Memory Concurrency Synchronization Patterns"
---

Recently I've been analyzing various data structures and algorithms in relaxed-memory (aka
weak-memory) concurrency. The plan is to identify **synchronization patterns** that is used again
and again, and explain concurrent programs in terms of the discovered synchronization patterns. This
post reports the observations I've made so far. Most notably, and quite surprisingly, it seems most
concurrent data structures and algorithms can be explained with only **three** synchronization
patterns!

Why I am doing it? Because I badly needed it when I started learning concurrent programming. I read
source codes of many data structures and articles that explain why they are correct. Each data
structure made sense, in very subtle and yet intriguing reasons, but a question always remained:
"how could they think of this data structure?" It becomes even worse when relaxed-memory concurrency
comes to play: most articles on shared-memory concurrency, from blog posts to published papers,
ignore the relaxed behaviors due to hardware and compiler optimizations, saying only that fences
should be somehow properly inserted to implementation for restricting the relaxed behaviors. In
short, at least to me, the whole concurrency business just looked like a black magic. I hoped some
light be shed on it.

Before delving into the synchronization patterns I've found, as a background study, I'll first
review shared-memory concurrency and its relaxed behaviors. (For experts: I will deviate from the
conventional explanation based on the "happens-before" relation, for the reasons I will explain
later.)



## Shared-Memory Concurrency and Relaxed Behaviors

Let's start with a single-threaded program. A memory basically **looks like** a map from addresses
to values:

```rust
memory: Map<Address, Value>
```

This should be the case regardless of whether there are caches in the memory hierarchy, or whether
CPUs and compilers reorder load and store instructions to the memory. In fact, all these
"optimizations" should guarantee that the memory appears to be a map from addresses to values for
single-threaded programs.

So far so good. Now let's think of a multi-threaded program. There are multiple threads that share a
single memory so that a value stored by a thread can be loaded by another one. Unfortunately, due to
all the hardware and compiler optimizations introduced for single-threaded programs, multi-threaded
programs cannot enjoy the luxury of fantasy that the memory is just a map. To see this, let's
consider the following program:

```rust
fn main() {
    let a: AtomicUsize = 0;
    let b: AtomicUsize = 0;

    let th1 = fork(|| {
        a.store(42, Relaxed);
        b.load(Relaxed)
    });

    let th2 = fork(|| {
        b.store(42, Relaxed);
        a.load(Relaxed)
    });

    assert(th1.join() == 42 || th2.join() == 42);
}
```

Let's say it's written in an imaginary language whose syntax resembles that of [Rust][rust]. `a` and
`b` are unsigned integers (`AtomicUsize`) which are initially `0`, a forked thread `th1` stores 42
to `a` and then loads from `b`, and another forked thread `th2` stores 42 to `b` and then loads from
`a`. `Relaxed` roughly means the loads and stores should be compiled to plain load and store
instructions in the assembly. The question of the assertion is, is it always true that either the
value loaded from `a` equals to 42, or that loaded from `b` equals to 42? At first glance, it seems
trivially so, since 42 should be stored to either `a` or `b` before loading a value from either of
them. **However**, it is not the case for mysterious reasons, and the assertion may fail by reading
0 from both `a` and `b`.[^reorder] The moral of this story is that **the abstraction of memory as a
map is broken**, and programs are allowed to exhibit more behaviors, even in unexpected ways from
the programmer's point of view.

[^reorder]: The CPU may reorder the store and load instructions in `th1` and `th2`, effectively
    loading 0 from `a` and `b` before storing 42 to them. Or it can be due to cache: each thread
    writes to and reads from its own cache, and the caches may not be in sync and have different
    opinions on the values stored in particular addresses. You know what? The compilers may also
    reorder the instructions, just because CPU may do that.

In order to tame these behaviors, a program should be properly **synchronized**. CPU architectures
such as x86-64, ARMv7/8, POWER, and programming languages such as C, C++, LLVM, Rust support
synchronization primitives in addition to plain load and store instructions. Unfortunately, these
primitives had been really [**difficult** to understand][problems], and even more difficult to
program with. As a result, only a small group of experts have spoken these primitives and
constructed concurrency libraries for the others. Still, these experts often misuse the primitives,
and even worse, the ISA and language standards have bugs in their specifications!

In order to resolve this issue, last year my collaborators and I
developed [a promising semantics for relaxed-memory concurrency][promising] that clearly explains
the semantics of C/C++ synchronization primitives. Now I will explain some of its concepts, namely
coherence and view, that collectively answer the very question of memory consistency models: which
value(s) a thread can read from the memory? As you will see, these concepts will play a key role in
explaining what is synchronization and how the synchronization patterns work.


### Coherence and View

Let's first see how a memory looks like. A memory in the promising semantics is a map from addresses
to **a set of values** uniquely ordered by a **timestamp**:

```rust
memory: Map<Address, Map<Timestamp, Value>>
```

We call each value a **message**. When a thread stores a value to an address, a message with the
stored value is added to the address; when a thread loads from an address, the value of a message of
the address is returned. It has two implications: (1) threads may be able to concurrently read
different values from the same address; but (2) at least, it is guaranteed that there is "some"
global order (namely the timestamp) of the writes to the same address. We call this the **"coherence
order"** (or "modification order" in the C/C++ standards lingo). For example, the messages of the
address `x` in a memory may look like:

```
          [x=1@10]  [x=42@20] [x=37@30] [x=2@40]  [x=3@50]
x --------+---------+---------+---------+---------+-------
```

where there are 5 messages of the address `x`: value 1 at the timestamp 10, 42 at 20, 37@30, 2@40,
and 3@50. It is worth noting that timestamps of different addresses are not related at all, since a
coherence order is specific to a single address.

When loading from and storing to the memory, threads should respect the coherence order: once a
thread acknowledged a message, it cannot read from and write to a timestamp before the message
w.r.t. the coherence order. More precisely, each thread maintains a **thread view**, which is a map
from addresses to timestamps that tracks the maximum timestamp it acknowledged for each address:

```rust
view: Map<Address, Timestamp>
```

And the thread view interacts with load and store instructions as follows. Suppose a thread's view
on the address `x` is at the timestamp `t1` (`view[x] == t1`). In other words, the thread somehow
acknowledged the message of the address `x` at timestamp `t1`, and nothing later. When it reads a
message at timestamp `t2` from `x`, `t1 <= t2` should hold, and at the same time `view[x]` is
updated to `t2` (since the message at `t2` is just acknowledged). When it writes a message at
timestamp `t2` to `x`, `t1 < t2` should hold, and at the same time `view[x]` is updated to `t2`
(since the message at `t2` is automatically acknowledged). In short, **loads and stores update a
thread's view**, and **view should be non-decreasing**.

This coherence condition is arguably a minimal requirement of sanity, so (almost) all CPUs and
compilers guarantee this by default, even in the absence of any synchronization primitives.[^alpha]
(For experts: the coherence condition of the promising semantics is roughly comparable to the
coherence axioms in the C/C++ standards. See [the paper][promising] for more detailed comparison.)

[^alpha]: A notable exception is C/C++ non-atomic loads: they may not respect the coherence
    order. See [the promising semantics paper][promising] for more details. Also, I heard that Alpha
    processors do not guarantee coherence.


### What is Synchronization?

Coherence is good, but with it alone programmers cannot construct concurrent programs. This is
mainly because the coherence condition applies to a single address at a time: acknowledgment of a
message of an address `x` has nothing to do with the thread's view on another address `y`. It is
synchronization that **relates acknowledgment of multiple addresses**.

I analyzed various real-world concurrent data structures and algorithms, and classified the
synchronization patterns used in them into three categories: **positive piggybacking
synchronization**, **negative piggybacking synchronization**, and **interleaving
synchronization**. In the remaining of this post, I will explain these three patterns and explain a
few concurrent data structures with them as running examples.

Let's begin with the most basic and well-understood pattern: positive piggybacking
synchronization. (For experts: think `Release`-`Acquire` synchronization. I deliberately used
general terms rather than C/C++-specific terms, e.g. "positive" and "piggybacking" rather than
"`Release`-`Acquire`", for implying that these patterns apply to CPUs and other languages as well.)



## Pattern 1: Positive Piggybacking Synchronization

In this pattern, we relate multiple addresses by **piggybacking the knowledge on an address to
another address**. For example, consider the following program:

```rust
fn main() {
    let data: AtomicUsize = 0;
    let flag: AtomicUsize = 0;

    let th1 = fork(|| {
        data.store(42, Relaxed);
        flag.store(1, Release)
    });

    let th2 = fork(|| {
        if (flag.load(Acquire)) {
            assert(data.load(Relaxed) == 42);
        }
    });
}
```

Here, the thread `th1` writes 42 to `data`, and then set the `flag`. The thread `th2` checks if the
`flag` is set, and if so, assert that `data` always contains 42. In order for successful assertion,
knowledge on the `data` variable should be piggybacked on the `flag` variable.

This synchronization is the job of the `Release` and `Acquire` ordering annotations in the load and
store instructions to `flag`. When a thread writes a message with the `Release` ordering, the
thread's view is annotated in the written message; and when a reads a message with the `Acquire`
ordering, the message's view is acknowledged by the reading thread. In order to do so, we attach a
view to memory's messages:

```
memory: Map<Address, Map<Timestamp, (Value, View)>>
```

Let's see what happens for the example above. Suppose `th1` stored `data = 42` at timestamp `10`,
and `flag = 1` at timestamp 5, as described in the timeline below. At the time of writing `flag =
1`, the thread `th1`'s view is `[data@10, flag@5]`, and this view is annotated in the message
`flag@5` because of the `Release` ordering. When `th2` reads `flag = 1`, its view becomes `[data@10,
flag@5]` thanks to the `Acquire` ordering, forcing the later load from `data` to read `42` (or a
message written in a later timestamp).

```
              [data=42@10]
data ---------+-----------

         [flag=1@5, view=[data@10, flag@5]]
flag ----+---------------------------------
```


### Example: Spinlock

This positive piggybacking synchronization pattern is used to implement a spinlock, which provides
**mutual exclusion** (mutex) among the **critical sections** (or code regions) in multiple
threads. By "mutual exclusion" I mean (1) there is a total order over critical sections; and (2) the
view at the end of a critical section should be acknowledged at the beginning of a later critical
section. (For experts: it is different from the conventional requirement that a critical section
happens before a later one.)

The following is an implementation of spinlock. Function `new()` creates a new spinlock, `lock()`
marks the beginning of a critical section, and `unlock()` marks the end of it:

```rust
struct Spinlock {
    lock: AtomicUsize,
}

impl Spinlock {
    fn new() -> Spinlock {
        Spinlock { lock: 0, }
    }

    fn lock(&self) {
        while (self.lock.compare_and_swap(0, 1, Acquire) != 0) {}
    }

    fn unlock(&self) {
        self.lock.store(0, Release);
    }
}
```

A spinlock consists of an integer (`AtomicUsize`) variable `lock`, which represents whether it is
locked by a critical section or not. As described in the timeline below, if its last value
w.r.t. the coherence order is `0`, it is not locked; if it is `1`, it is locked:

```
[UNLOCKED]
     (init) (lock)-(unlock)
     [0]    [1]    [0]
lock +!!!!!!+------+-------

[LOCKED]
     (init) (lock)-(unlock)   (lock)
     [0]    [1]    [0]        [1]
lock +!!!!!!+------+!!!!!!!!!!+-----

[UNLOCKED, again]
     (init) (lock)-(unlock)   (lock)----(unlock)
     [0]    [1]    [0]        [1]       [0]
lock +!!!!!!+------+!!!!!!!!!!+---------+-------
```

The `new()` function initializes a spinlock by assigning `0` to the internal `lock` variable,
meaning that it is not locked at the beginning.

The `lock()` function spins until successfully updating the variable from `0` to `1`. In order to
guarantee that only one thread can update a variable from `0` to `1`, the `compare_and_swap()`
function resolves the race over the `lock` variable. In order for a thread to win the race and enter
a critical section, the thread should be able to read the old value (`0` in this case) and then
subsequently write a new value (`1` in this case), with no existing messages between the old and the
new value. If it is the case, the timestamps between the old and the new value are marked as
unusable in the future (represented as `!` in the timeline), and the old value is returned. For
example, a successful `lock()` may change the memory from `[UNLOCKED]` to `[LOCKED]` in the above
timeline. Thanks to the exclusive nature of `compare_and_swap()` (and the invariant that the
neighbor of only the last message is not marked as unusable), at any time, only one thread can
successfully update the `lock` variable. If a thread loses the race, `compare_and_swap()` returns a
value in the memory just like `load()`, letting the `lock()` function try again.

The `unlock()` function simply stores `0` to the `lock` variable. For example, `unlock()` may change
the memory from `[LOCKED]` to `[UNLOCKED, again]`.

Now let's see how the above implementation guarantees mutual exclusion. First, the view at the end
of a critical section is annotated in the `lock` variable thanks to the `Release` ordering of
`unlock()`'s store to the `lock` variable. Then the annotated view is acknowledged at the beginning
of the next critical section due to the `Acquire` ordering of `lock()`'s `compare_and_swap()`. This
(positive) piggybacking synchronization on the `lock` variable transitively sends the view of a
critical section to later ones.


### Variations

There are other kinds of piggybacking synchronization than `Release`-`Acquire` synchronization. In
theory, you can think of the following dimensions of variation:

- **The bridge of piggybacking.** In `Release`-`Acquire` synchronization, a `Release`-write to the
  piggybacking address (`flag` in the example) and an `Acquire`-read from the address form a
  "bridge" of piggybacking. We may call this WR (write-read) bridge. You can imagine RRc (read-read
  via coherence) bridge, whose two reads from the piggybacking address are ordered by the coherence
  order; RWc bridge of a read and a coherence-later write to the piggybacking address; WRc bridge;
  and WWc bridge. (For experts: you may call C/C++ release sequence "(WR)* bridge" ("*" means the
  Kleene Star).)

- **Synchronization marker**. In the example, the `Release` and `Acquire` orderings are annotated in
  the store and load instructions. Instead, you can mark a program point earlier than the bridge's
  write as `Release`, and mark a program point later than the bridge's load as `Acquire`; we call
  these markers "fences". For example, the example could have been written in this way:

  ```rust
  // ... unchanged

  let th1 = fork(|| {
      data.store(42, Relaxed);
      fence(Release);
      flag.store(1, Relaxed)
  });

  let th2 = fork(|| {
      if (flag.load(Relaxed)) {
          fence(Acquire);
          assert(data.load(Relaxed) == 42);
      }
  });
  ```

  Here, the view at the time of `fence(Release)` is acknowledged at the time of `fence(Acquire)`,
  thereby successfully asserting `data.load(Relaxed) == 42`. You may mix `Release` store with
  `Acquire` fence or `Release` fence with `Acquire` load for achieving the same thing.

In reality, not all these combinations are realized in CPUs and languages, partially because they
are not efficiently implementable. For what is worth, C/C++ supports (1) synchronization via WR (and
WRu) bridges annotated with `Release` and `Acquire` orderings, namely the ordinary
`Release`-`Acquire` synchronization; (2) its variations with fences; and (3) synchronization via
RRc, RWc, WRc, WWc bridges in the presence of `SeqCst` fences in both sides, where `SeqCst` is the
strongest and the most expensive ordering in C/C++.


### Note

Positive piggybacking synchronization is a relatively well-understood pattern compared to the other
two. Many of the earliest concurrent data structures, e.g. [Treiber stack][treiber]
and [Michael-Scott queue][msqueue], rely only on this pattern. In C/C++, the notion of
"happens-before" relation, which is at the heart of the memory model, is built upon this pattern
only. While there is no satisfactory program logic for relaxed-memory concurrency in
general, [a program logic for release/acquire fragments of C/C++][ralogic] had been successfully
developed.

This pattern is "positive" in the sense that **the synchronization depends on positive
information**, namely the acknowledgment of a message. For the example program, since `th2` observed
`flag = 1`, it should have also observed `data = 42`. The next pattern is also based on
piggybacking, but it is "negative": it depends on negative information, namely the absence of
acknowledgment.



## Pattern 2: Negative Piggybacking Synchronization

For the example program above, acknowledgment of `flag = 1` implies that of `data = 42`. The
contraposition is also true: the absence of the acknowledgment of `data = 42` implies that of the
acknowledgment of `flag = 1`. This pattern is used in ["sequence lock"][seqlock]: an optimized
implementation of reader-writer lock. Note that a reader-writer lock is a mechanism for protecting
data which guarantees mutual exclusion among writers, while providing a designated, optimized method
for reading, but not writing, the protected data. For correctness, the reader should be **atomic**
in the sense that it should not observe intermediate modification of data by a concurrent writer.


### Example: Sequence Lock

The following is an implementation of sequence lock. Function `new()` creates a new sequence lock,
`writer_lock()` and `writer_unlock()` marks the beginning and the end of a critical section in which
the writer can access the protected data, and `read()` returns the protected data:

```rust
struct<T> Seqlock<T: Copy> {
    seq: AtomicUsize,
    data: Atomic<T>,
}

impl<T: Copy> Seqlock<T> {
    fn new(data: T) -> Seqlock<T> {
        Seqlock { seq: 0, data: Atomic::new(data), }
    }

    fn writer_lock(&self) -> (usize, &mut T) {
        loop {
            let seq = self.seq.load(Relaxed);
            if (seq & 1 != 0) { continue };

            if (self.seq.compare_and_swap(seq, seq + 1, Acquire) != seq) { continue };

            fence(Release);
            return (seq + 2, self.data as &mut T); // It's not Rust exactly..
        }
    }

    fn writer_unlock(&self, seq: usize) {
        self.seq.store(seq, Release);
    }

    fn read(&self) -> T {
        loop {
            let seq1 = self.seq.load(Acquire);
            if (seq1 & 1 != 0) { continue };

            let result = self.data.load(Relaxed);
            fence(Acquire);

            let seq2 = self.seq.load(Relaxed);
            if (seq1 != seq2) { continue };

            return result;
        }
    }
}

// Using a seqlock
fn main() {
    let seqlock = Seqlock::new(...);

    let th1 = fork(|| {
        let (seq_next, val) = seqlock.writer_lock();
        ... // writer's critical section
        seqlock.writer_unlock(seq_next);
    });

    let th2 = fork(|| {
        let val = seqlock.read();
    });
}
```

The `new()`, `writer_lock()`, and `writer_unlock()` functions guarantee mutual exclusion for the
same reason with spinlock, but using even numbers instead of `0` for representing unlocked states
and odd numbers instead of `1` for representing locked states. Below is an example timeline for the
`seq` variable. For example, a writer updates `seq` from 0 to 1 and then writes `2` to `seq`. Let's
call it `W2`. Similarly, the writer that writes `4` to `seq` is `W4`, and so on:

```
                                             (R4)
    (init) (W2: lock)-(unlock) (W4: lock)----(unlock)  (W6: lock)--(unlock)
    [0]    [1]        [2]      [3]           [4]       [5]         [6]
seq +!!!!!!+----------+!!!!!!!!+-------------+!!!!!!!!!+-----------+-------
```

The `read()` function is atomic for the following reasons. Suppose a reader `R4` observed `seq1 =
seq2 = 4`. I will show that `R4` reads the data written by `W4`.

First, the view at the end of `W4` is sent to the beginning of `R4` by positive piggybacking
synchronization from `writer_unlock()`'s `Release`-write (`self.seq.store(seq, Release)`) to
`read()`'s `Acquire`-load (`let seq1 = self.seq.load(Acquire)`). In particular, we know that the
view on data at the end of `W4` <= view on data at the beginning of `R4`.

Second, the view on data at the end of `R4` <= the view on data at the end of `W4`. Otherwise, a
part of the data `R4` read from came from a writer later than `W4`. By positive piggybacking
synchronization on data from the later writer's `writer_lock()`'s `fence(Release)` to `read()`'s
`fence(Acquire)`, the message of `seq = 5` or later should have been acknowledged after `read()`'s
`fence(Acquire)`. But it is a contradiction, because the reader observed that `seq2 = 4`. In other
words, by negative piggybacking synchronization on data, `seq2 = 4` means the view on data at the
end of `R4` <= the view on data at the end of `W4`.

Therefore throughout the execution of `R4`, its view on data should equal to the view on data at the
end of `W4`. So `R4` reads exactly the data completely written by `W4`.

(For experts: it is worth noting that `R4` does not happen before `W6`: we only know that `R4`'s
view on data <= the view on data at the beginning of `W6`. It is just sufficient for a reader-writer
lock to be correct. In my opinion, the specifications of some data structures, including sequence
lock, are more naturally expressible with views than with the happens-before relation. It's the
reason I prefer explaining the synchronization patterns with views.)


### Note

Recall that in the synchronization patterns introduced so far, knowledge on an address is
piggybacked on another address, reusing the coherence order of the "another address" as a
bridge. But in some cases, we need a stronger synchronization by **ordering arbitrary program
points** from different threads. This is the job of interleaving fences.



## Pattern 3: Interleaving Synchronization

Interleaving fences (the `SeqCst` fence in C/C++, and the most heavyweight fences in CPU
architectures) mark the program points that should be totally ordered. When a thread `th1` executes
an interleaving fence before another thread `th2` executes another interleaving fence w.r.t. the
total order, `th1`'s view before executing the fence should be acknowledged by `th2` after executing
the fence. In order to do so, the promising semantics maintains a global view for interleaving
fences:


```rust
static mut interleaving_view: View // we need `unsafe` accessor in Rust, but..
```

When a thread executes an interleaving fence, it calculates the (address-wise) max of its view and
the global interleaving view, and set the max view to its own view and the global interleaving view:

```rust
fn execute_interleaving_fence(&mut thread) {
    let view = max(thread.view, interleaving_view);
    thread.view = interleaving_view = view;
}
```

Interleaving synchronization is very powerful: in fact, it subsumes both forms of piggybacking
synchronizations. However, in my opinion, it is more difficult to reason about than piggybacking
synchronizations, as combinatorial explosion occurs when analyzing all possible interleavings. I
would recommend to use this pattern only if the power of interleaving is actually necessary.

This is the only kind of synchronization supported in what is called "sequentially consistent
semantics", or "interleaving semantics", where all the instructions are interleaving by default
(thereby allowing no relaxed behaviors). For this reason, I believe sequentially consistent
semantics is difficult to reason about, at least as much as relaxed-memory concurrency semantics. I
know many of you cannot agree with me on this matter; sequential consistency is regarded as the
ideal and the easiest semantics for shared-memory concurrency for decades. But I believe the scene
has changed a little bit: now we can explain the synchronization patterns in terms of views.

Interleaving synchronization is used, for example, in [Peterson's algorithm][peterson], which is an
early mutual exclusion algorithm, and [work-stealing deque by Chase and Lev][chaselev]. In the
remaining of this section, I will analyze Peterson's algorithm in more details.


### Example: Peterson's Mutual Exclusion

The following is an implementation of Peterson's mutual exclusion algorithm. As opposed to spinlock
and sequence lock, Peterson's algorithm I am presenting here supports only two threads:

```rust
fn main() {
    let flag: [AtomicBool; 2];
    let turn: AtomicUsize = 0;

    fn lock(id: Usize) {
        flag[id].store(true, Relaxed);
        fence(SeqCst);                 // A
        turn.store(1 - id, Relaxed);
        fence(SeqCst);                 // B
        while (flag[1 - id].load(Acquire) && turn.load(Relaxed) == 1 - id) {}
    }

    fn unlock(id: Usize) {
        flag[id].store(false, Release);
    }

    let th0 = fork(|| {
        lock(0);
        // critical section
        unlock(0);
    });

    let th1 = fork(|| {
        lock(1);
        // critical section
        unlock(1);
    });
}
```

Peterson's algorithm guarantees mutual exclusion for the following reasons. In `th0`'s and `th1`'s
call to `lock()`, there are four `SeqCst` fences: `th0`'s first fence (`A0`), `th0`'s second
`fence(SeqCst)` (`B0`), `th1`'s first fence (`A1`), and `th1`'s second fence (`B1`). We will analyze
every possible order of these fences, but without loss of generality, it is sufficient to analyze
the following orders only:

- `A0` -> `B0` -> `A1` -> `B1`.

  By interleaving property, `flag[0] = true` and `turn = 1` should be acknowledged after `A1`. So
  `th1` should write `turn = 0` after `turn = 1` w.r.t. the coherence order, and it should spin
  until `th0` write `flag[0] = false` in `unlock()`. By positive piggybacking synchronization on
  `flag[0]`, the view at the end of `th0`'s critical section is sent to the beginning of `th1`s
  critical section.

- `A0` -> `A1` -> `B0` -> `B1` or `A0` -> `A1` -> `B1` -> `B0`.

  By interleaving property, `flag[0] = true` and `flag[1] = true` should be acknowledged after both
  `B0` and `B1`. Without loss of generality, suppose that `th0` wrote `turn = 1` before `th1` did
  `turn = 0` w.r.t. the coherence order. Then `th1`'s `lock()` should spin until `th0` writes
  `flag[0] = false` in `unlock()`. By positive piggybacking synchronization on `flag[0]`, the view
  at the end of `th0`'s critical section is sent to the beginning of `th1`s critical section.



## Case Study: Crossbeam

So far I identified three relaxed-memory concurrency synchronization patterns, and explained a few
data structures with the mix of those three patterns. Now let's analyze something bigger:
the [Crossbeam][crossbeam] library for epoch-based concurrent memory reclamation scheme in Rust. For
more information on this library, I refer you
to [Aaron Turon's introduction to Crossbeam][crossbeam-turon]. I wrote down
a [Crossbeam RFC][crossbeam-rfc] that explains why Crossbeam's implementation is correct w.r.t. the
C/C++ memory model. (In fact, many of the ideas I am presenting here was conceived when I was
writing the RFC.) I believe now you can read it, and find where, how, and which patterns are used in
Crossbeam :smile:



## Future Work

I tried to identify synchronization patterns as comprehensively as possible, but probably there
should be more patterns yet to be discovered. Most notably, I omitted some emerging synchronization
primitives, which have a potential to be a basis for new and faster synchronization patterns:

- **Data-dependent piggybacking synchronization.** For example, `Consume` loads in C/C++ is a
  variant of `Acquire` loads where the effect of `Acquire` (acknowledging the `Release`d view) takes
  place only for the instructions that depend on the `Consume`-loaded value. For example, consider
  the following program:

  ```rust
  fn main() {
      let data: AtomicUsize = 0;
      let ptr: AtomicUsize = 0;

      let th1 = fork(|| {
          data.store(42, Relaxed);
          ptr.store(&data as usize, Release)
      });

      let th2 = fork(|| {
          let p = ptr.load(Consume) as &AtomicUsize;
          if (!p.is_null()) {
              assert(p.load(Relaxed) == 42);    // dependent on `p`, safe
              assert(data.load(Relaxed) == 42); // independent from `p`, unsafe
          }
      });
  }
  ```

  Here, `ptr` is read with the `Consume` annotation. Since the assertion `p.load(Relaxed) == 42`
  depends on the value `p` read from `ptr`, the piggybacking synchronization happens and the
  assertion should succeed. On the other hand, since the assertion `data.load(Relaxed) == 42` does
  not **syntactically** depend on `p`, piggybacking synchronization does not happen, and
  `data.load(Relaxed)` can read the initial value `0` and fail the assertion.

  `Consume` or `READ_ONCE` is faster than `Acquire` in relaxed CPU architectures such as ARM and
  Power, and is actually used in the Linux kernel (in the name of `READ_ONCE`). Unfortunately, we
  currently [do not know of a good semantics for that][consume]. However, I believe its usage falls
  into the positive/negative piggybacking synchronization patterns.

- **Strongly-synchronizing load/store instructions.** C/C++ allows `SeqCst` annotations for load and
  store instructions. These instructions are intended to be more strongly synchronizing than those
  annotated with just `Release` and `Acquire`. Even, in order to support more efficient compilation
  of them, the ARMv8 architecture introduced the `LDA` (load-acquire) and `STL` (store-release)
  instructions (and their variants). However, as discussed in [this paper][scfix], the semantics of
  `SeqCst` loads and stores have been severely broken, and the only fix proposed so far is too
  complicated. Fixing its semantics and identifying its usage patterns is an important future work.

- **System-wide synchronization.** The `sys_membarrier` syscall on Linux and
  `FlushProcessWriteBuffers` on Windows essentially perform a `SeqCst` fence on all CPU cores. As
  discussed in [a Crossbeam RFC][sysfence], these system-wide fences may be used to eliminate a
  fence in a critical path at the expense of introducing a system-wide fence in a cold
  path. Identifying its usage patterns is also an important future work.

- **Synchronizing heterogeneous systems.** So far I focused only on on-board inter-CPU
  synchronization, but these days more hardware devices, including GPU, NIC (Network Interface
  Controller) cards, and maybe TPU (Tensorflow Processing Unit) are synchronizing with each
  other. Is there any unique usage pattern in these heterogeneous systems?



## Conclusion

I hope this post got its job done: clarifying the essence of relaxed-memory concurrency by
identifying synchronization patterns, thereby helping you start writing concurrent programs. I hope
concurrency is no longer a black magic to you as was it to me two years ago, but a disciplined
subject of modern systems programming. Happy hacking concurrency!



### Edit

I would like to thank @foollbar, @stjepang, @Vtec234, Benjamin Fry, Derek Dreyer, and Gil Hur for
their helpful comments to an earlier version of this post.


[promising]: http://sf.snu.ac.kr/promise-concurrency
[scfix]: http://plv.mpi-sws.org/scfix/
[treiber]: http://domino.research.ibm.com/library/cyberdig.nsf/0/58319a2ed2b1078985257003004617ef?OpenDocument
[msqueue]: http://dl.acm.org/citation.cfm?id=248106
[ralogic]: http://plv.mpi-sws.org/igps/
[seqlock]: http://www.hpl.hp.com/techreports/2012/HPL-2012-68.pdf
[peterson]: https://en.wikipedia.org/wiki/Peterson%27s_algorithm
[chaselev]: http://www.di.ens.fr/~zappa/readings/ppopp13.pdf
[crossbeam]: https://github.com/crossbeam-rs
[crossbeam-turon]: https://aturon.github.io/blog/2015/08/27/epoch/
[crossbeam-rfc]: https://github.com/crossbeam-rs/rfcs/blob/master/text/2017-07-23-relaxed-memory.md
[sysfence]: https://github.com/crossbeam-rs/rfcs/blob/master/text/2017-05-23-epoch-gc.md#system-wide-fences
[consume]: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0371r1.html
[rust]: https://www.rust-lang.org/
[problems]: https://link.springer.com/chapter/10.1007/978-3-662-46669-8_12
