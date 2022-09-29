# Fiber Oriented Stack Switching

## Motivation

Non-local control flow (sometimes called _stack switching_) operators allow applications to manage their own computations. Technically, this means that an application can partition its logic into separable computations and can manage (i.e., schedule) their execution. Pragmatically, this enables a range of control flows that include supporting the handling of asynchronous events, supporting cooperatively scheduled threads of execution, and supporting yield-style iterators and consumers.

This proposal refers solely to those features of the core WebAssembly virtual machine needed to support non-local control flow and does not attempt to preclude any particular strategy for implementing high level languages. We also aim for a reasonably minimal spanning set of concepts[^a]: the author of a high level language compiler should find all the necessary elements to enable their compiler to generate appropriate WebAssembly code fragments.

[^a]: _Minimality_ here means something slightly more than a strictly minimal set. In particular, if there is a feature that _could_ be implemented in terms of other features but that would incur significant penalties for many applications then we likely include it.

## Fibers and Events

The main concepts in this proposal are the _fiber_ and the _event_. A fiber is a first class value which denotes the resource used to support a computation. An event is used to signal a change in the flow of computation from one fiber to another. 

### The `fiber` concept

A `fiber` is a resource that is used to enable a computation that has an extent in time and has identity. The latter allows applications to manage their own computations and support patterns such as async/await, green threads and yield-style generators. 

Associated with fibers is a new (unparameterized) reference type: `fiber`.

#### The state of a fiber

A fiber is inherently stateful; as the computation proceeds the fiber's state
reflects that evolution. The state of a fiber includes function call frames and
local variables that are currently live within the computation as well as
whether the fiber is currently being executed. In keeping with the main design
principles of WebAssembly, the frames and local variables represented within a
fiber are not accessible other than by normal operations.

In addition to the frames and local variables, it is important to know whether a
fiber is resumable or not. In essence, only fibers that have been explicitly
suspended may be resumed, and only fibers that are either currently executing or
have resumed computations that are executing may be suspended.

In general, validating that a suspended fiber may be resumed is a constant-time
operation but validating that an executing fiber may be suspended involves
examining the chain of fiber resumptions.
In practice, for applications conforming to the recommended resume/switch/suspend usage model (discussed below), that validation will also be constant-time.

Even though a fiber may be referenced after it has completed, such references
are not usable and the referenced fibers are asserted to be _moribund_[^b]. Any
attempt to switch to a moribund fiber will result in a trap.

[^b]: However, [an alternate approach](#using-tables-to-avoid-gc-pressure) suggests an alternate approach for managing moribund fibers.

#### Fibers, ancestors and children

This design ensures an important relationship across fibers that is parallel to function calls (as they were before this proposal).

Prior to this proposal, when one activation of a function calls another function there are only three ways control could exit the newly created activation: returning, throwing an exception, or trapping.
Aside from trapping, all control transfers up the call-stack pass through the "outer" activation, and the outer activation has some way of catching it (e.g. using the returned value, or using `catch_all`).
This property helps a wasm application maintain invariants, such as keeping its shadow stack aligned with its wasm stack.

With this proposal, that property is no longer true because the inner activation can suspend up or switch to another fiber without the outer activation having any ability to observe or influence the transfer of control.
Left unchecked, this would make WebAssembly a hostile environment—where programs have to proactively defend against such unexpected transfers of control—as well as an incomposable environment—where one program's use of non-local control can easily accidentally inferfere with another program's use of non-local control.

To prevent this, our design maintains a _control parent_ relationship between fibers (with _control ancestor_ being the transitive closure thereof).
When one fiber is the control parent of another, our design guarantees that control cannot transfer up the control stack—aside from trapping— without the control parent being able to observe/interced *unless* the control child is given a reference to one of its control ancestors.
In other words, if you call a function *that you know has no reference to the current fiber or any of its ancestors*, then you have the same control guarantees as before this proposal.

WebAssembly is always called from within some embedding.
For performance reasons, it is useful for WebAssembly call frames to be able to reside on the embedder stacks.
But these stacks might not be suspendable, and they are not fibers.
One property of the composability of this proposal is that a WebAssembly execution has no way of observing if it is executing on such a stack versus on a fiber—a WebAssembly execution can only observe if it is executing on a given fiber or a control descendent thereof.
Furthermore, any "good" WebAssembly application will only ever attempt to transfer control between fibers it can guarantee are appropriately related—the notion of control parent is only used to dynamically enforce the minimal "good" behavior needed for reliably composing WebAssembly programs, and to specify the semantics of traps (and exceptions).
At some points we will abuse terminology and refer to such a stack as a _rooted fiber_ because control can transfer to and from it like fibers, even though it is not a fiber and control cannot transfer above it.
Every executing fiber has a rooted fiber as its greatest control ancestor.

### The `event` concept

An event is an occurrence where execution is explicitly transferred between fibers.
Events also represent a communication opportunity between fibers: an event may communicate
data as well as signal a control transfer.

#### Event declaration
Every change in computation is associated with an _event description_: whenever a fiber is suspended, the suspending fiber uses an event to signal both the reason for the suspension and to communicate any necessary data. Similarly, when a fiber is resumed, an event is used to signal to the resumed fiber the reason for the resumption.

An event description has a predeclared `tag` which determines the type of event
and what values are associated with the event. Event tags are declared:

```
(tag $e (param t*))
```
where the parameter types are the types of values communicated along with the event. Event tags may be exported from modules and imported into modules.

When execution switches between fibers, the originating fiber must ensure that values corresponding to the event tag's signature are on the value stack prior to the switch. After the switch, the values are available to the newly executing fiber, also on the value stack, and will no longer be available to the originating fiber.

## Primitive Instructions

We introduce instructions for creating, mounting, switching, dismounting, and releasing fibers.

#### `fiber.new` Create a new fiber

The `fiber.new` instruction creates a new fiber.
The instruction has the form `fiber.new $return_event ($on_event $resume_func)* : [ti*] -> [fiberref]` where
* `$return_event` is an event tag of type `[to*]`
* Each `$on_event` is an event tag of some type `[te*]`, and each corresponding `$resume_func` is a function of type `[ti* fiberref te*] -> [to*]`

The result of the `fiber.new` instruction is a `fiber` which is the identifier for the newly created fiber.

The new fiber is created in a suspended state without a control parent.
When the fiber is resumed with an event with tag `$resume_event`, the list is checked (in order) for a matching `$on_event`.
If found, the corresponding `$resume_func` is called on the fiber with arguments given by the `ti*` provided to `fiber.new` (so that each newly created fiber is a closure), the `fiberref` of the newly created fiber (so that a fiber conveniently knows its own identity), and the `te*` payload of the event the fiber was resumed with.

If `$resume_event` throws an (unhandled) exception or traps, the fiber becomes moribund and the exception/trap is propagated to the fiber's control parent.
If `$resume_func` returns (necessarily with values of type `to*`), the fiber becomes moribund and the fiber's control parent is resumed with an event with tag `$return_event` and payload given by the returned values.
In either case, the fiber no longer has a control parent. (Moribund fibers never have a control parent.)

#### `fiber.mount` Mount a suspended fiber

When a WebAssembly application's typical exported function is called, that function typically will be executing on a rooted fiber.
Even if the fiber were not rooted, a typical application should not need to transfer control to above the caller.
That is, just like prior to this proposal, any unhandled exceptions thrown (or traps raised) by the exported function should be handled by the caller of the exported function.
And, just like how at present exporting functions often need to set up a shadow stack or an exception handler to be used during the execution of its (internal) functions, a fiber-switching application needs to set up the fiber to run.
This is the purpose of `fiber.mount`.

The `fiber.mount` instruction has the form `fiber.mount $resume_event ($on_event $handle_label)* : [tr* fiberref] -> []` where
* `$resume_event` is an event tag of type `[tr*]`
* Each `$on_event` is an event tag of some type `[te*]`, and each corresponding `$handle_label` is a label of type `[te*]`

Supposing the given `fiberref` currently has no control parent and is not moribund, the instruction *sets the current fiber as the given fiber's control parent* and resumes the given fiber using the event with tag `$resume_event` and payload given by the given `tr*` values.
This leaves the current fiber in a suspended state.
When the current fiber is resumed with an event with some tag `$dismount_event`, a matching `$on_event` is searched for (in order), and control is transferred to its corresponding label.

If the given `fiberref` has a control parent or is moribund, `fiber.mount` traps.
If, however, the given `fiberref` does not have a matching handler for the `$resume_event`, then the instruction following `fiber.mount` is executed.
This enables runtimes-implemented-in-wasm to attempt to use a more efficient event (e.g. with an unboxed integer as its payload) and fall back to a less efficient event (e.g. with a boxed integer as its payload), which has applications to efficient implementation of polymorphic languages.

The are two very important properties of `fiber.mount`:
1. It works even if the current fiber is rooted.
2. It sets the current fiber as the control parent of the given fiber

Taken together, these make `fiber.mount` act much like "calling" the given fiber, and it maintains the control hierarchy that facilitates composing WebAssembly programs securely and abstractly.
However, as we demonstrate next, the given fiber has a lot more flexibility with what it can do below this mounting point than a function.

#### `fiber.switch` Switch to a different fiber

In native implementations of runtimes with stack-switching, the implementation simply switches between its stacks.
There is no concern for a control hierarchy because the operation system executes the runtime in a localized environment.
The role of `fiber.mount` is to essentially set up that localized environment within a setting that does need to maintain a control hierarchy.
But once set up, all code running on an "internal" fiber of an application can freely `fiber.switch` between the application's internal fibers, just as if the application were implemented natively.

The `fiber.switch` instruction has the form `fiber.switch $resume_event ($on_event $handle_label)* : [tr* fiberref fiberref] -> []` where
* `$resume_event` is an event tag of type `[tr*]`
* Each `$on_event` is an event tag of some type `[te*]`, and each corresponding `$handle_label` is a label of type `[te*]`

The `fiber.switch` instruction takes two fibers: the first is the fiber to suspend and the second is the fiber to switch to.
Exucting this instruction first dynamically checks important preconditions to maintaining the control hierarchy.
1. The suspending fiber must be the currently executing fiber *or a control ancestor thereof*.
   Requiring the suspending fiber in the first place ensures the application was explicitly given the capability to suspend the current fiber.
   Permitting control ancestors allows the application to provide another application a fiber-switching callback while also permitting the other application to itself use fiber-switching between its own fibers (and without having to know how the other application is implemented).
   Explicitly specifying the suspending fiber also prevents ambiguity as to which control ancestor to suspend to *even when there are multiple control ancestors originating from the current application* (which can happen in the presence of multiple callbacks or nested back-and-forths between applications).
   (Of course, most applications will not use such callbacks, in which case the suspending fiber will be the current fiber. Engines are expected to optimize for this exceedingly common case, so that nearly all switches in practice will be constant-time and will not enter a loop.)
2. The resuming fiber must not be moribund and must either have no control parent or must itself be the suspending fiber.
   (The latter addresses a corner case. Engines can easily implement the following steps in a manner that avoids needing to explicitly check for this corner case.)

Supposing those preconditions are satisfied (otherwise the instruction traps), `fiber.switch` *transfers the suspending fiber's control parent to the resuming fiber* and resumes it using the event with tag `$resume_event` and payload given by the given `tr*` values.
This leaves the current fiber in a suspended state *with no control parent*.
When the current fiber is resumed with an event with some tag `$dismount_event`, a matching `$on_event` is searched for (in order), and control is transferred to its corresponding label.
If the resuming fiber does not have a matching handler for the `$resume_event`, then the control parent is not transferred and instead the instruction following `fiber.mount` is executed.

Although it may be viewed as being a combination of `fiber.dismount` (discussed next) and `fiber.mount`, there is an important architectural distinction (besides the obvious performance distinction): the signaling event. Under the common hierarchical organization, a suspending fiber does not know which fiber will be resumed. This means that the signaling event has to be of a form that the fiber's manager is ready to process. However, with a `fiber.switch` instruction, the fiber's manager is not informed of the switch and does not need to understand the signaling event.
This, in turn, means that a fiber manager may be relieved of the burden of communicating between fibers. I.e., `fiber.switch` supports a symmetric coroutining pattern.

The effect of this instruction is that, once an exported function sets up the application's runtime including mounting some fiber, the internal functions (running within that setup runtime) will switch between the application's fibers aribtrarily.
Each switch transfers the control parent, so these fibers will always be the control child of the mounting point.
Thus, when such a fiber is done, its root function will return and control will transfer to that mounting point with whatever event tag the fiber was created with.
In this way, this design supports a design that is as flexible as native stack-switching and nearly as easy to use, with most code using `fiber.switch` and just a few key and obvious spots in the application's runtime-implemented-in-wasm using `fiber.mount`.

#### `fiber.dismount` Dismount an active fiber

Although technically implementable with the above instructions, architecturally `fiber.dismount` serves as the final component of the recommended mount/switch/dismount usage pattern.
Switch can be seen as a "horizontal" transfer within the control hierarchy, whereas mount and dismount are "vertical" transfers.
Mount transfers downward (extending the hierarchy), and dismount transfers upwards (shortening the hierarchy).

The `fiber.dismount` instruction has the form `fiber.dismount $resume_event ($on_event $handle_label)* : [tr* fiberrer] -> []` where
* `$resume_event` is an event tag of type `[tr*]`
* Each `$on_event` is an event tag of some type `[te*]`, and each corresponding `$handle_label` is a label of type `[te*]`

The `fiber.dismount` instruction takes one fiber: the fiber to suspend.
Like with `fiber.switch`, this instruction first dynamically checks that the suspending fiber is the currently executing fiber or a control ancestor thereof, trapping otherwise.
`fiber.dismount` *unsets the suspending fiber's control parent* and resumes the former control parent using the event with tag `$resume_event` and payload given by the given `tr*` values.
This leaves the current fiber in a suspended state with no control parent.
When the current fiber is resumed with an event with some tag `$dismount_event`, a matching `$on_event` is searched for (in order), and control is transferred to its corresponding label.
If the former control parent does not have a matching handler for the `$resume_event`, then the control parent is not transferred and instead the instruction following `fiber.mount` is executed.

This instruction is to be used when the application wants to give control back to its caller before it has finished its work.
The two common scenarios for this are when the application needs to wait for some result, e.g. from some Promise returned (as an `externref`) from an asynchronous JavaScript API, or when the application recognizes too much time has passed and the event loop should be allowed to continue executing (registering some callback with the browser to be invoked later in order to complete the work).

#### `fiber.release` Destroy a suspended fiber

The `fiber.release` instruction has the form `fiber.release : [fiberref] -> []` and makes the given fiber and all its control descendents moribund, clearing them of any associated computational resources.
The identified fiber must have no control parent, otherwise the instruction traps.
The instruction does not trap if the given fiber is already moribund.

Since fiber references are wasm values, the reference itself remains valid. However, the fiber itself is now in a moribund state that cannot be resumed.

The `fiber.release` instruction is primarily intended for situations where a fiber manager needs to eliminate unneeded fibers.
Note that it does not trigger any unwinding of the given fiber, as that would require transferring control to the given fiber.

## Utility Instructions

The following can be expressed using the above instructions but help with some common cases while also providing optimization opportunities that would be hard to identify if broken down into multiple above instructions.

#### `fiber.mount_new`/`fiber.switch_new` Create and execute a new fiber

The `fiber.mount_new` and `fiber.switch_new` instructions create a new fiber and immediately enter it.

The first has the form `fiber.mount_new $return_event $fiber_func ($on_event $handle_label)* : [ti*] -> [fiberref to*]`.
It creates a new fiber and immediately mounts it, executing `$fiber_func` on the fiber with the given `ti*` values as arguments.
If the current fiber is resumed with `$return_event` (say because `$fiber_func` returns), control is transferred to the instruction following `fiber.spawn` with the returned values (along with a reference to the fiber that was allocated); all other events are processed according to the given event-handler table (except each label has an additional first `fiberref` parameter providing the reference to the fiber that was allocated).
`fiber.switch_new` is similar but also takes a `funcref` to switch from.

While it is easy to imitate these operations with `fiber.new` and `fiber.mount`/`fiber.switch`, applications are encouraged to use these operations when it is likely that the given fiber will return without switching to another fiber or dismounting.
Engines might support optimistically executing `$fiber_func` on the *current* fiber, only allocating a new fiber and moving stack frames to it if/when an actual switch/dismount first occurs.

#### `fiber.dismount_release`/`fiber.switch_release` Leaving and releasing a fiber

The `fiber.dismount_release` and `fiber.switch_release` instructions are used when a fiber has finished its work and wishes to inform another fiber of its final results without necessarily returning from its root function.
It has the obvious form and semantics.

Besides being a convenience, these instructions combine well with `fiber.mount_new` and `fiber.switch_new`.
In particular, if the current fiber was the newly allocated fiber and its frames are still on the fiber that allocated it, those frames can be released without being moved.

## Examples

We look at three examples in order of increasing complexity and sophistication: a yield-style generator, cooperative threading and handling asynchronous I/O.

### Yield-style generators

The so-called yield style generator pattern consists of a pair: a generator function that generates elements and a consumer that consumes those elements. When the generator has found the next element it yields it to the consumer, and when the consumer needs the next element it waits for it. Yield-style generators represents the simplest use case for stack switching in general; which is why we lead with it here.

#### Generating elements of an array
We start with a simple C-style pseudo-code example of a generator that yields for every element of an array. For explanatory purposes, we introduce a new `generator` function type and a new ``yield` statement to C:
```
void generator arrayGenerator(fiber *thisTask,int count,int els){
  for(int ix=0;ix<count;ix++){
    thisTask yield els[ix];
  }
}
```
The statement:
```
thisTask yield els[ix]
```
is hypothetical code that a generator might execute to yield a value from a generator.

In WebAssembly, this generator can be written:
```
(type $generator (ref fiber i32))
(tag $identify (param ref $generator))
(tag $yield (param i32))
(tag $next)
(tag $end-gen)

(func $arrayGenerator (param $thisTask $generator) (param $count i32) (param $els i32)
  (handle $on-init
    (fiber.suspend (local.get $thisTask) $identify (local.get $thisTask))
    (event $next)  ;; we are just waiting for the first next event
  )

  (local $ix i32)
  (local.set $ix (i32.const 0))
  (loop $l
    (local.get $ix)
    (local.get $count)
    (br_if $l (i32.ge (local.get $ix) (local.get $count)))

    (handle
      (fiber.suspend (local.get $thisTask)
          $yield 
          (i32.load (i32.add (local.get $els) 
                             (i32.mul (local.get $ix)
                                      (i32.const 4))))))
      (event $on-next ;; set up for the next $next event
        (local.set $ix (i32.add (local.get $ix) (i32.const 1)))
        (br $l)
      )
    ) ;; handle
  ) ;; $l
  ;; nothing to return
  (return)
)
```
When a fiber suspends, it must be in a `handle` context where one or more `event` handlers are available to it. In this case our generator has two such `handle` blocks; the first is used during the generator's initialization phase and the second during the main loop.

Our generator will be created in a running state&mdash;using the `fiber.spawn` instruction. This means that one of the generator function's first responsibilities is to communicate its identity to the caller. This is achieved through the fragment:
```
(handle $on-init
  (fiber.suspend (local.get $thisTask) $identify (local.get $thisTask))
  (event $next)  ;; we are just waiting for the first next event
)
```
The `$arrayGenerator` suspends itself, issuing an `identify` event that the caller will respond to. The generator function will continue execution only when the caller issues a `$next` event to it. Suspending with the `$identify` event allows the caller to record the identity of the generator's fiber.

During normal execution, the `$arrayGenerator` is always waiting for an `$next` event to trigger the computation of the next element in the generated sequence. If a different event were signaled to the generator the engine would simply trap.

Notice that the array generator has definite knowledge of its own fiber&mdash;it is given the identity of its fiber explictly. This is needed because when a fiber suspends, it must use the identity of the fiber that is suspending. There is no implicit searching for which computation to suspend.

The end of the `$arrayGenerator`&mdash;which is triggered when there are no more elements to generate&mdash;is marked by a simple `return`. This will terminate the fiber and also signal to the consumer that generation has finished.

#### Consuming generated elements
The consumer side of the generator/consumer scenario is similar to the generator; with a few key differences:

* The consumer drives the overall choreography
* The generator does not have a specific reference to the consumer; but the consumer knows and manages the generator. 

As before, we start with a C-style psuedo code that uses a generator to add up all the elements generated:
```
int addAllElements(int count, int els[]){
  fiber *generator = arrayGenerator(count,els);
  int total = 0;
  while(true){
    switch(generator resume next){
      case yield(El):
        total += El;
        continue;
      case end:
        return total;
    }
  }
}
```
>The expression `generator resume next` is new syntax to resume a fiber with an
>identified event (in this case `next`); the value returned by the expression is
>the event signaled by the resumed fiber when it suspends.

In WebAssembly, the `addAllElements` function takes the form:
```
(func $addAllElements (param $count i32) (param $els i32) (result i32)
  (local $generator $generator)
  (handle
    (fiber.spawn $arrayGenerator (local.get $count) (local.get $els))
    (trap) ;; do not expect generator to return yet 
    (event $identify (param ref $generator)
      (local.set $generator)
    )
  )

  (local $total i32)
  (local.set $total i32.const 0)
  (loop $l
    (handle $body
      (fiber.resume (local.get $generator) $next ())
      ;; The generator returned, exit $body, $lp
      (event $yield (param i32)
        (local.get $total) ;; next entry to add is already on the stack
        (i32.add)
        (local.set $total)
        (br $l)
      )
      (event $end ;; not used in this example
        (br $body)
      )
    )
  )
  (local.get $total)
  (return)
)
```
The first task of `$addAllElements` is to establish a new fiber to handle the generation of elements of the array. We start the generator running, which impies that we first of all need to wait for it to report back its identity. `fiber.spawn` is like `fiber.resume`, in that if the function returns, the next instruction will execute normally. However, we don't expect the generator to return yet, so we will trap if that occurs.

The main structure of the consumer takes the form of an unbounded loop, with a forced termination when the generator signals that there are no further elements to generate. This can happen in two ways: the generator function can simply return, or it can (typically) `fiber.retire` with an `$end` event. Our actual generator returns, but our consumer code can accomodate either approach.

>In practice, the style of generators and consumers is dictated by the toolchain. It would have been possible to structure our generator differently. For example, if generators were created as suspended fibers&mdash;using `fiber.new` instructions&mdash;then the initial code of our generator would not have suspended using the `$identify` event. However, in that case, the top-level of the generator function should consist of an `event` block: corresponding to the first `$next` event the generator receives.
>In any case, generator functions enjoy a somewhat special relationship with their callers and their structure reflects that.

Again, as with the generator, if an event is signaled to the consumer that does not match either event tag, the engine will trap.

### Cooperative Coroutines

Cooperative coroutines, sometimes known as _green threads_ allow an application to be structured in such a way that different responsibilities may be handled by different computations. The reasons for splitting into such threads may vary; but one common scenario is to allow multiple sessions to proceed at their own pace.

In our formulation of green threads, we take an _arena_ based approach: when a program wishes to fork into separate threads it does so by creating an arena or pool of fibers that represent the different activities. The arena computation as a whole only terminates when all of the threads within it have completed. This allows a so-called _structured concurrency_ architecture that greatly enhances composability[^c].

[^c]: However, how cooperative coroutines are actually structured depends on the source language and its approach to handling fibers. We present one alternative; many languages don't use structured concurrency techniques and collect all green threads into a single pool.

Our `$arrayGenerator` was structured so that it was entered using a `fiber.spawn` instruction; which implied that the `$arrayGenerator`'s first operation involved suspending itself with an `$identify` event.

In our threads example, we will take a different approach: each green thread
will be associated with a function, but will be created as a suspended fiber.
This allows the fiber arena manager to properly record the identity of each
thread as it is created and to separately schedule the execution of its managed
threads.

#### Structure of a Green Thread
We start with a sketch of a thread, in our C-style pseudo-code, that adds a collection of generated numbers, but yielding to the arena scheduler between every number:
```
void fiber adderThread(fiber *thisThred, fiber *generatorTask){
  int total = 0;
  while(true){
    switch(thisThred suspend pause_){
      case cancel_thread:
        return; // Should really cancel the generator too
      case go_ahead_fiber:{
        switch(generator resume next){
          case yield(El):
            total += El;
            continue;
          case end:
            // report the total somewhere
            thisThred suspend finish_(total);
            return;
        }
      }
    }
  }
}
```
Note that we cannot simply use the previous consumer function we constructed because we want to pause the fiber between every number. A more realistic scenario would not pause so frequently.

The WebAssembly version of `adderThread` is straightforward:
```
(tag $pause_)
(tag $yield_ (param i32))
(tag $go_ahead)
(tag $cancel_thread)
(type $thread ref fiber)

(func $adderThread (param $thisThred $thread) (param $generator $generator)
  (local $total i32 (i32.const 0))
  (event $cancel_thread)
    (local.get $generator)
    (fiber.release) ;; kill off the generator fiber
    (local.get $total) ;; initially zero
    (return)
  )
  (event $go_ahead           ;; event block at top-level of function
    (local.set $total (i32.const 0))
    (loop $l
      (handle $body
        (handle $gen
          (fiber.resume (local.get $generator) $next)
          (br $body)           ;; generator returned, so we're done
          (event $yield (param i32)
            (local.get $total) ;; next entry to add is already on the stack
            (i32.add)
            (local.set $total)
            (fiber.suspend (local.get $thisThred) $pause_)
          )
          (event $end
            (br $body)
          )
        )
        (event $go_ahead
          (br $l)
        )
        (event $cancel_thread
          (br $body)         ;; strictly redundant
        )
      ) ;; $body
    ) ;; $l
    (fiber.retire (local.get $thisThred) ($finish_ (local.get $total)))
  ) 
)
```
The fiber function is structured to have an essentially empty top-level
body&mdash;it only consists of `event` handlers for the fiber protocol. This
protocol has two parts: when a fiber is resumed, it can be either told to
continue execution (using a `$go_ahead` event), or it can be told to cancel
itself (using a `$cancel_thread` event).

This is at the top-level because our green thread fibers are created initially in suspended state.

The main logic of the fiber function is a copy of the `$addAllElements` logic, rewritten to accomodate the fiber protocol.

A lot of the apparent complexity of this code is due to the fact that it has to embody two roles: it is a yielding client of the fiber manager and it is also the manager of a generator. Other than that, this is similar in form to `$addAllElements`.

>Notice that our `$adderFiber` function uses `fiber.release` to terminate the generator fiber. This avoids a potential memory leak in the case that the `$adderFiber` is canceled.
>In addition, the `$adderFiber` function does not return normally, it uses a `fiber.retire` instruction. This is used to slightly simplify our fiber arena manager.

#### Managing Fibers in an Arena
Managing multiple computations necessarily introduces the concept of
_cancellation_: not all the computations launched will be needed. In our arena
implementation, we launch several fibers and when the first one returns we will
cancel the others:
```
int cancelingArena(fiber fibers[]){
  while(true){
    // start them off in sequence
    for(int ix=0;ix<fibers.length;ix++){
        switch(fibers[ix] resume go_ahead){
        case pause_:
            continue;
        case finish_(T):{
            for(int jx=0;jx<fibers.length;jx++){
              cancel fibers[jx]; // including the one that just finished
          }
          return T
        }
      }
    }
  } // Loop until something interesting happens
}
```
>We don't include, in this example, the code to construct the array of fibers.
>This is left as an exercise.

The WebAssembly translation of this is complex but not involved:
```
(fun $cancelingArena (param $fibers i32)(param $len i32)(result i32)
  (local $ix i32)
  (local $jx i32)
  (loop $l
    (local.set $ix (i32.const 0))
    (loop $for_ix
      (handle
        (fiber.resume 
          (table.get $task_table (i32.add (local.get $fibers)(local.get $ix)))
          $go_ahead)
        (event $pause_
          (local.set $ix (i32.add (local.get $ix)(i32.const 1)))
          (br_if $for_ix (i32.ge (local.get $ix) (local.get $len)))
          (br $l)
        )
        (event finish_ ;; We cancel all other fibers
          (local.set $jx (i32.const 0))
          (loop $for_jx
            (handle $inner_jx
              (br_if $inner_jx (i32.eq (local.get $ix)(local.get $jx)))
              (table.get $task_table (i32.add (local.get $fibers)(local.get $jx)))
              (fiber.resume $cancel_thread) ;; cancel fibers != ix
              (event $finish_ ;; only acceptable event
                (local.set $jx (i32.add (local.get $jx)(i32.const 1)))
                (br_if $for_jx (i32.ge (local.get $jx)(local.get $len)))
                (return) ;; total on stack
              )
              (event $pause
                (trap)
              )
            )
          )
        )
      )
    )
  )
)
```
The main additional complications here don't come from threads per se; but rather from the fact that we have to use a table to keep track of our fibers. 

### Asynchronous I/O
In our third example, we look at integrating fibers with access to asynchronous APIs; which are accessed from module imports. 

On the web, asynchronous functions use the `Promise` pattern: an asynchronous I/O operation operates by first of all returning a `Promise` that 'holds' the I/O request, and at some point after the I/O operation is resolved a callback function attached to the `Promise` is invoked.

>While non-Web embeddings of WebAssembly may not use `Promise`s in exactly the same way, the overall architecture of using promise-like entities to support async I/O is widespread. One specific feature that may be special to the Web is that it is not possible for an application to be informed of the result of an I/O request until after the currently executing code has completed and the browser's event loop has been invoked.

#### Our Scenario
The JavaScript Promise Integration API (JSPI) allows a WebAssembly module to call a `Promise`-returning import and have it result in the WebAssembly module being suspended. In effect, using the JSPI results in the entire program being suspended.

However, we would like to enable applications where several fibers can make independant requests to a `fetch` import and only 'return' when we have issued them all. Specifically, our example will involve multiple fibers making `fetch` requests and responding when the requests complete.

This implies a combination of local scheduling of tasks, possibly a _tree_ of schedulers reflecting a hierarchical structure to the application, and, as we shall see, some form of multiplexing of requests and demultiplexing of responses. This aspect is perhaps unexpected but is forced on us by the common Web browser embedding: only the browser's outermost event loop is empowered to actually schedule tasks when I/O activities complete.

#### A `fetch`ing Fiber
On the surface, our fibers that fetch data are very simple:
```
async fetcher(string url){
  string text = await simple_fetch(url);
  doSomething(text);
}
```
In another extension to the C language, we have invented a new type of function&mdash;the `async` function. In our mythical extension, only `async` functions are permitted to use the `await` expression form. Our intention is that such a function has an implicit parameter: the `Fiber` that will be suspended when executing `await`.

#### Importing `fetch`
The actual `fetch` Web API is quite complex; and we do not intend to explore that complexity. Instead, we will use a simplified `simple_fetch` that takes a url string and returns a `Promise` of a `string`. (Again, we ignore issues such as failures of `fetch` here.)

Since it is our intention to continue execution of our application even while we are waiting for the `fetch`, we have to somewhat careful in how we integrate with the browser's event loop. In particular, we need to be able to separate which fiber is being _suspended_&mdash;when we encounter the `fetch`es `Promise`&mdash;and which fiber is resumed when the fetch data becomes _available_.

We can express this with the pseudo code:
```
function simple_fetch(client,url){
    fetch(url).then(response => {
      scheduler resume io_notify(client,response.data);
    });
    switch(client.suspend async_request){
      case io_result(text): return text;
    }
  }
}
```
Notice how the `simple_fetch` function invokes `fetch` and attaches a callback to it that resumes the `scheduler` fiber, passing it the identity of the actual `client` fiber and the results of the `fetch`. Before returning, `simple_fetch` suspends the `client` fiber; and a continuation of _that_ suspension will result in `text` being delivered to the client code.

It is, of course, going to be the responsibility of the `scheduler` to ensure that the data is routed to the correct client fiber.

#### A note about the browser event loop
It is worth digging in a little deeper why we have this extra level of indirection. Fundamentally, this arises due to a limitation[^d] of the Web browser architecture itself. The browser's event loop has many responsibilities; inluding the one of monitoring for the completion of asynchronous I/O activities initialized by the Web application. In addition, the _only_ way that an application can be informed of the completion (and success/failure) of an asynchronous operation is for the event loop to invoke the callback on a `Promise`.

This creates a situation where our asynchronous WebAssembly application must ultimately return to the browser before any `fetch`es it has initiated can be delivered. However, this kind of return has to be through our application's own scheduler. And it must also be the case that any resumption of the WebAssembly application is initiated through the same scheduler.

In particular, if the browser's event loop tries to directly resume the fiber that created the `Promise` we would end up in a situation that is very analogous to a _deadlock_: when that fiber is further suspended, or even if if completes, the application as a whole will stop&mdash;because other parts of the application are still waiting for the scheduler to be resumed; but that scheduler was waiting to be resumed by the browser's event loop scheduler.

[^d]: A better phrasing of this might be an unexpected consequence of the browser's event model. This limitation does not apply, for example, to normal desktop applications running in regular operating systems.

The net effect of this is that, for browser-based applicatios, we must ensure that we _multiplex_ all I/O requests through the scheduler and _demultiplex_ the results of those requests back to the appropriate leaf fibers. The demultiplex is the reason why the actual callback call looks like:
```
scheduler resume io_notify(client,response.data)
```
This `resume` tells the scheduler to resume `client` with the data `response.data`. I.e., it is a kind of indirect resume: we resume the scheduler with an event that asks it to resume a client fiber. One additional complication of this architecture is that the scheduler must be aware of the types of data that Web APIs return.

It is expected that, in a non-browser setting, one would not need to distort the architecture so much. In that case, the same scheduler that decides which fiber to resume next could also keep track of I/O requests as they become available.

#### An async-aware scheduler
Implementing async functions requires that a scheduler is implemented within the language runtime library. This is actually a consequence of having a special syntax for `async` functions.

Our async-aware scheduler must, in addition to scheduling any green threads under its control, also arrange to suspend to the non-WebAssembly world of the browser in order to allow the I/O operations that were started to complete. And we must also route the responses to the appropriate leaf fiber.

A simple, probably naïve, scheduler walks through a circular list of fibers to run through. Our one does that, but also records whenever a fiber is suspending due to a `Promise`d I/O operation:

```
typedef struct{
  Fiber f;
  ResumeProtol reason;
} FiberState;
Queue<FiberState> fibers;
List<Fiber> delayed;

void naiveScheduler(){
  while(true){
    while(!fibers.isEmpty()){
      FiberState next = fibers.deQueue();
      switch(next.f resume next.reason){
        case pause:
          fibers.push(FiberState{f:next,reason:go_ahead});
          break;
        case async_request:{
          delayed.put(next);
          break;
        }
      }
      if(fibers.isEmpty() || pausing){
        switch(scheduler suspend global_pause){
          case io_notify(client,data):{
            reschedule(client,data);
          }
        } 
      }
    }
  }
}
```

#### Reporting a paused execution
The final piece of our scenario involves arranging for the WebAssembly application itself to pause so that browser's eventloop can complete the I/O operations and cause our code to be reentered.

This is handled at the point where the WebAssembly is initially entered&mdash;i.e., through one of its exports. For the sake of exposition, we shall assume that we have a single export: `main`, which simply starts the scheduler:
```
void inner_main(fiber scheduler){
  startScheduler(scheduler);
}
```
In fact, however, our application itself should be compatible with the overall browser architecture. This means that our actual toplevel function returns a `Promise`:
```
function outer_main() {
  return new Promise((resolve,reject) => {
    spawn Fiber((F) => {
      try{
        resolve(inner_main(F));
      } catch (E) {
        reject(E);
      }
    });
  }
}
```
We should not claim that this is the only way of managing the asynchronous activities; indeed, individual language toolchains will have their own language specific requirements. However, this example is primarily intended to explain how one might implement the integration between WebAPIs such as `fetch` and a `Fiber` aware programming language.

#### The importance of stable identifiers
One of the hallmarks of this example is the need to keep track of the identities of different computations; possibly over extended periods of time and across significant _code distances_.

For example, we have to connect the scheduler to the import in order to ensure correct rescheduling of client code. At the time that we set up the callback to the `Promise` returned by `fetch` we reference the `scheduler` fiber. However, at that moment in time, the `scheduler` fiber is still technically running (i.e., it is not suspended). Of course, when the callback is invoked by the event loop the scheduler is suspended.

This correlation is only possible because the identity of a fiber is stable&dash;regardless of its current execution state.

#### Final note

Finally, the vast majority of the code for this scheduler is _boilerplate_ code that is manipulating data structures. We leave it as an exercise for the reader to translate it into WebAssembly.

## Frequently Asked Questions

### Why are we using 'lexical scoping' rather than 'dynamic scoping'
A key property of this design is that, in order for a WebAssembly program to switch fibers, the target of the switch is explicitly identified. This so-called lexical scoping approach is in contract with a dynamic approach&mdash;commonly used for exception handling&mdash;where the engine is expected to search the current evaluation context to decide where to suspend to (say).

#### Implementation
In a lexically-scoped design, the engine is explicitly told by the program where to transfer control to.
Thus, the only additional obligation the engine has to implement, besides the actual transfer of control, is validation that the target is _valid_ from the current control point.

In a dynamically-scoped design, the engine has to search for the transfer target. This search is typically not a simple search to specify and/or implement since the _criteria_ for a successful search depends on the language, both current and future.

By requiring the program to determine the target, the computation of this target becomes a burden for the toolchain rather than for the WebAssembly engine implementor.

#### Symmetric coroutining (and its cousin: task chaining)
With symmetric coroutines you can have a (often binary) collection of coroutines that directly yield to each other via application-specific messages. We saw a simple example of this in our [generator example](#generating-elements-of-an-array).

Similar patterns arise when chaining tasks together, where one computation is intended to be followed by another. Involving a scheduler in this situation creates difficulties for types (the communication patterns between the tasks is often private and the types of data are not known to a general purpose scheduler).

A lexically-scoped design more directly/simply/efficiently supports these common horizonal control-transfer patterns than a dynamically-scoped design.

#### Composing Components
In applications where multiple _components_ are combined to form an application the risks of dynamic scoping may be unacceptable. By definition, components have a _shared nothing_ interface where the only permitted communications are those permitted by a common _interface language_. This includes prohibiting exceptions to cross component boundaries&mdash;unless via coercion&mdash;and switching between tasks.

By requiring explicit fiber identifiers we make the task (sic) of implementing component boundaries more manageable when coroutining is involved. In fact, this is envisaged in the components design by using _streaming_ and _future_ metaphors to allow for this kind of control flow between components.

### What is the difference between first class continuations and fibers?
A continuation is semantically a function that, when entered with a value, will finish an identified computation. In effect, continuations represent snapshots of computations. A first class continuation is reified; i.e., it becomes a first class value and can be stored in tables and other locations.

The snapshot nature of a continuation is especially apparent when you compare delimited continuations and fibers. A fiber may give rise to multiple continuations&mdash;each time it suspends[^e] there is a new continuation implied by the state of the fiber. However, in this proposal, the fiber is reified wheras continuations are not. 

One salient aspect of first class continuations is _restartability_. In principal, a reified continuation can be restarted more than once&mdash;simply by invoking it. 

It would be possible to achieve the effect of restartability within a fibers design&mdash;by providing a means of _cloning_ fibers. 

However, this proposal, as well as others under consideration, does not support the restartability of continuations or cloning of fibers.

[^e]: It can be reasonably argued that a computation that never suspends represents an anti-pattern. Setting up suspendable computations is associated with significant costs; and if it is known that a computation will not suspend then one should likely use a function instead of a fiber.

### Can Continuations be modeled with fibers?
Within reason, this is straightforward. A fiber can be encapsulated into a function object in such a way that invoking the function becomes the equivalent of entering the continuation. This function closure would have to include a means of preventing the restartability of the continuation.

### Can fibers be modeled with continuations?
Within reason, this too is straightforward. A fiber becomes an object that embeds a continuation. When the fiber is to be resumed, the embedded continuation is entered.

Care would need to be taken in that the embedded continuation would need to be cleared; a more problematic issue is that, when a computation suspends, the correct fiber would have to be updated with the appropriate continuation.

### How are exceptions handled?

Fibers and fiber management have some conceptual overlap with exception handling. However, where exception handling is oriented to responding to exceptional situations and errors, fiber management is intended to model the normal&mdash;if non-local&mdash; flow of control.

When an I/O operation fails (say), and a requesting fiber needs to be resumed with that failure, then the resuming code (perhaps as part of an exception handler) resumes the suspended fiber with a suitable event. In general, all fibers, when they suspend, have to be prepared for three situations on their resumption: success, error and cancelation. This is best modeled in terms of an `event.switch` instruction listening for the three situations.

One popular feature of exception handling systems is that of _automatic exception propagation_; where an exception is automatically propagated from its point of origin to an outer scope that is equipped to respond to it. This proposal follows this by allowing unhandled exceptions to be propagated out of an executing fiber and into its resuming parent.

However, this policy is generally incompatible with any form of computation manipulation. 

The reason is that, when a fiber is resumed, it may be from a context that does not at all resemble the original situation; indeed it may be resumed from a context that cannot handle any application exceptions. This happens today in the browser, for example. When a `Promise` is resumed, it is typically from the context of the so-called microtask runner. If the resumed code throws an exception the microtask runner would be excepted to deal with it. In practice, the microtask runner will silently drop all exceptions raised in this way.

A more appropriate strategy for handling exceptions is for a specified sibling fiber, or at least a fiber that the language runtime is aware of, to handle the exception. This can be arranged by the language runtime straightforwardly by having the failing fiber signal an appropriate event.

There is a common design element between this proposal and the exception handling proposal: the concept of an event. However, events as used in fiber oriented computation are explicitly intended to be as lightweight as possible. For example, there is no provision in events as described here to represent stack traces. Furthermore, events are not first class entities and cannot be manipulated, stored or transmitted.

### How do fibers fit in with structured concurrency?
The fiber-based approach works well with structured concurrency architectures. A relevant approach would likely take the form of so-called fiber _arenas_. A fiber arena is a collection of fibers under the management of some scheduler. All the fibers in the arena have the same manager; although a given fiber in an arena may itself be the manager of another arena.

This proposal does not enfore structured concurrency however. It would be quite possible, for example, for all of the fibers within a WebAssembly module to be managed by a single fiber scheduler. It is our opinion that this degree of choice is advisable in order to avoid unnecessary obstacles in the path of a language implementer.

### Are there any performance issues?
Stack switching can be viewed as a technology that can be used to support suspendable computations and their management. Stack switching has been shown to be more efficient than approaches based on continuation passing style transformations[^f].

[^f]:Although CPS transformations do not require any change to the underlying engine; and they more readily can support restartable computations.

A fiber, as outlined here, can be viewed as a natural proxy for the stack in stack switching. I.e., a fiber entity would have an embedded link to the stacks used for that fiber. 

Furthermore, since the lifetime of a stack is approximately that of a fiber (a deep fiber may involve multiple stacks), the alignment of the two is good. In particular, a stack can be discarded precisely when the fiber is complete&mdash;although the fiber entity may still be referenced even though it is moribund.

On the other hand, any approach based on reifing continuations must deal with a more difficult alignment. The lifetime of a continuation is governed by the time a computation is suspended, not the whole lifetime. This potentially results in significant GC pressure to discard continuation objects after their utility is over.

### How do fibers relate to the JS Promise integration API?
A `Suspender` object, as documented in that API, corresponds reasonably well with a fiber. Like `Suspender`s, in order to suspend and resume fibers, there needs to be explicit communication between the top-level function of a fiber and the function that invokes suspension.

A wrapped export in the JS Promise integration API can be realized using fibers
quite straightforwardly: as code that creates a fiber and executes the wrapped
export. This can be seen in the pseudo-JavaScript fragment for the export
wrapper[^g]:
```
function makeAsyncExportWrapper(wasmFn) {
  return function(...args) {
    return new Promise((resolve,reject) => {
      spawn Fiber((F) => {
        try{
          resolve(wasmFn(F,args));
        } catch (E) {
          reject(E);
        }
      })
    })
  }
}
```
[^g]: This code does not attempt to depict any _real_ JavaScript; if for no other reason than that we do not anticipate extending JavaScript with fibers.

Similarly, wrapping imports can be translated into code that attaches a callback to the incoming `Promise` that will resume the fiber with the results of the `Promise`:
```
function makeAsyncImportWrapper(jsFn) {
  return function(F,...args) {
    jsFn(...args).then(result => {
      F.resume(result);
    });
    F.suspend()
  }
}
```
However, as can be seen with the [asynchronous I/O example](#asynchronous-io), other complexities involving managing multiple `Promise`s have the combined effect of making the JSPI itself somewhat moot: for example, we had to multiplex multiple `Promise`s into a single one to ensure that, when an I/O `Promise` was resolved, our scheduler could be correctly woken up and it had to demultiplex the event into the correct sub-computation.

### How does one support opt-out and opt-in?
The fundamental architecture of this proposal is capability based: having access to a fiber identifier allows a program to suspend and resume it. As such, opt-out is extremely straightforward: simply do not allow such code to become aware of the fiber identifier.

Supporting full opt-in, where only specially prepared code can suspend and resume, and especially in the so-called _sandwich scenario_ is more difficult. If a suspending module invokes an import that reconnects to the module via another export, then this design will allow the module to suspend itself. This can invalidate execution assumptions of the sandwich filler module.

It is our opinion that the main method for preventing the sandwich scenario is to prevent non-suspending modules from importing functions from suspending modules. Solutions to this would have greater utility than preventing abuse of suspendable computations; and perhaps should be subject to a different effort.

### Why does a fiber function have to suspend immediately?

The `fiber.new` instruction requires that the first executable instruction is an `event.switch` instruction. The primary reason for this is that, in many cases, eliminates an extraneous stack switch.

Fibers are created in the context of fiber management; of course, there are many flavors of fiber management depending on the application pattern being used. However, in many cases, the managing software must additionally perform other bookkeeping fibers (sic) when creating sub-computations. For example, in a green threading scenario, it may be necessary to record the newly created green thread in a scheduling data structure.

By requiring the `fiber.new` instruction to not immediately start executing the new fiber we enable this bookkeeping to be accomplished with minimal overhead.

However, this proposal also includes the `fiber.spawn` instruction to accomodate those language runtimes that prefer the pattern of immediately executing new fibers.

### Isn't the 'search' for an event handler expensive? Does it involve actual search?
Although there is a list of `event` blocks that can respond to a switch event,
this does not mean that the engine has to conduct a linear search when decided
which code to execute.

Associated with each potential suspension point in a `handle` block one can construct a table of possible entry points; each table entry consisting of a program counter and an event tag. When a fiber is continued, this table is 'searched' for a corresponding entry that matches the actual event.

Because both the table entries and any potential search key are all determined statically, the table search can be implemented using a [_perfect hash_](https://en.wikipedia.org/wiki/Perfect_hash_function) algorithm. I.e., a situation-specific hash algorithm that can guarantee constant access to the correct event handler.

As a result, in practice, when switching between fibers and activating appropriate `event` blocks, there is no costly search involved. 

### How does this concept of fiber relate to Wikipedia's concept
The Wikipedia definition of a [Fiber](https://en.wikipedia.org/wiki/Fiber_(computer_science)) is[^h]:

>In computer science, a fiber is a particularly lightweight thread of execution.

[^h]: As of 8/5/2022.

Our use of the term is consistent with that definition; but our principal modification is the concept of a `fiber`. In particular, this allows us to clarify that computations modeled in terms of fibers may be explicitly suspended, resumed etc., and that there may be a chain of fibers connected to each other via the resuming relationship.

Our conceptualization of fibers is also intended to support [Structured Concurrency](https://en.wikipedia.org/wiki/Structured_concurrency). I.e., we expect our fibers to have a hierachical relationship and we also support high-level programming patterns involving groups of fibers and single exit of multiple cooperative activities.

## Open Design Issues

This section identifies areas of the design that are either not completely resolved or have significant potential for variability.

### The type signature of a fiber
The type signature of a fiber has a return type associated with it. This is not an essential requirement and may, in fact, cause problems in systems that must manage fibers.

The reason for it is to accomodate the scenario of a fiber function returning. Since all functions must end at some point, establishing a reasonable semantics of what happens then seems important.

Without the possiblity of returning normally, the only remaining recourse for a fiber would be to _retire_. We expect that, in many usage scenarios, this would be the correct way of ending the life of a fiber.

### Exceptions are propagated
When an exception is thrown in the context of a fiber that is _not_ handled by the code of the fiber then that exception is propagated out of the fiber&mdash;into the code of the resuming parent.

The biggest issue with this is that, for many if not most applications, the resuming parent of a parent is typically ill-equipped to handle the exception or to be able to recover gracefully from it.

The issue is exacerbated by the fact that functions are _not_ annotated with any indication that they may throw an exception let alone what type of exception they may throw.

### Fibers are started in suspended/running state
This proposal allows fibers to be created in a suspended state or they can be created and immediately entered when `spawn`ed.

Allowing fibers to be created in suspended state causes significant architectural issues in this design: in particular, because such a fiber has no prior history of execution (it *is* a new fiber), the fiber function has to be structured differently to account for the fact that there will be a resume event with no corresponding suspend event.

On the other hand, requiring fibers to be started immediately on creation raises its own questions. In particular, if the spawner of a fiber also needs to record the identity of the fiber then the fiber must immediately suspend with some form of `identify` event. We saw this in the generator example. There are enough applications where this would result in a significant performance penalty, for example in a green threading library that is explicitly managing its fiber identities.

For this reason, we support both forms of fiber creation. However, this also represents a compromise and added cost for implementation.

### Using tables to avoid GC pressure

One of the consequences of having first class references to resources is that there is a potential for memory leaks. In this case, it may be that an application 'holds on' to a `fiber` references after that `fiber` has terminated.

When a `fiber` terminates, the stack resources it uses may be released; however, the reference itself remains. Recall that, in linear memory WebAssembly, any application that wishes to keep a reference to a `fiber` has three choices: to store the reference in a global variable, to store it in a table or to keep it in a local variable.

An alternative approach could be based on _reusing_ `fiber` references. In particular, if we allow a moribund fiber to be reused then the issue of garbage collecting old `fiber` references becomes a problem for the toolchain to address: it would become responsible for managing the `fiber` references it has access to.

A further restriction would enhance this: if the only place where a `fiber` reference could be stored was in a table, then, if the default value for a `fiber` table entry were a moribund `fiber`, complete reponsibility for managing `fiber` references could be left to the toolchain. 
