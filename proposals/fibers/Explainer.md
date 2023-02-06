# Fiber Oriented Stack Switching

## Motivation

Non-local control flow (sometimes called _stack switching_) operators allow applications to manage their own computations. Technically, this means that an application can partition its logic into separable computations and can manage (i.e., schedule) their execution. Pragmatically, this enables a range of control flows that include supporting the handling of asynchronous events, supporting cooperatively scheduled threads of execution, and supporting yield-style iterators and consumers.

This proposal refers solely to those features of the core WebAssembly virtual machine needed to support non-local control flow and does not attempt to preclude any particular strategy for implementing high level languages. We also aim for a reasonably minimal spanning set of concepts[^a]: the author of a high level language compiler should find all the necessary elements to enable their compiler to generate appropriate WebAssembly code fragments.

Outlined here are the core features that should enable producers to target the new functionality this proposal offers.
Over time, we might extend these features should producers identify more functionality that would provide substantial benefit without significantly expanding the scope of this proposal.

[^a]: _Minimality_ here means something slightly more than a strictly minimal set. In particular, if there is a feature that _could_ be implemented in terms of other features but that would incur significant penalties for many applications then we likely include it.

## Stacks and Jump References

The main concepts in this proposal are *fibers* and *jump references*.
A fiber is a resource that is used to store a linear sequence of call frames; conceptually, these frames are placed sequentially in memory, though behind the scenes engines might implement a fiber through a series of segments.
A jump reference, when in a valid state, refers to a call frame within a fiber where control can be transferred to.
Due to the linear nature of fibers, whenever control is transferred to a particular call frame, all later call frames *in that fiber* are ensured to be no longer accessible.
Thus, although this proposal does not provide direct references to fibers, their role as a resource is important for understanding the formal semantics and performance of this proposal.

### The `jumpref` type

This proposal introduces one and only one type: `jumpref`.
This type is unparameterized.
It denotes a "jump reference", and could similarly be called a continuation reference or a jump buffer.

### The state of a jump reference

When created, a jump reference refers to a frame within a fiber that has been set up to support non-local control transfers.
However, when (and only when) control is transferred to or above that frame within that fiber, the jump reference becomes permanently *invalidated*.

### The state of fibers

A fiber is inherently a resource.
In this proposal, it is the responsibility of the engine to manage this resource.
In particular, when portions of a fiber become unreachable, the engine is responsible for performing any associated cleanup.
What this proposal ensures, though, is that unreachability can be recognized using just reference counting.
Furthermore, many instructions combined with the linear nature of fibers make it clear that certain portions of a fiber will no longer be reachable.

## Instructions

We first introduce an instruction for creating jump references and an instruction for transferring control using jump references *within* a fiber.
We then introduce an instruction for creating fibers and transferring control using jump references *across* fibers.
We finally introduce instructions for interacting with exceptions.
For convenience and readability, we use `unreachable` as a pseudo-type that in actuality is typed much like instructions such as `br`.

### Creating jump references

```
jump.call_with_new $func $label* : [ti*] -> [to*]
```
where `$func : [ti* jumpref] -> [to*]`

This instruction allocates a new jump reference and calls `$func` with that fresh reference, along with the arguments on the value stack.
The jump reference refers to the current frame, and the table of labels specify how the jump reference handles a non-local control transfer.

### Long jumps

```
jump.br_long index $labeltype : [t* jumpref] -> unreachable
```
where `$labeltype = [t*]` (or a function type where the result type is ignored)

This instruction transfers control to the label at `index` in the given jump reference's table within the frame referenced by the jump reference.
The instruction traps if the jump reference is invalid (i.e. the frame no longer exists), the frame belongs to a different fiber than control is currently on, the index is invalid, or `$labeltype` does not match the type of the corresponding label.

Note that this control transfer requires the engine to invalidate all jump references to frames between the current and target frames.
There are simple techniques for doing so in time linear to the number of such references (rather than call frames), enabling fast implementations of non-local control transfer (along the lines of what one would expect from `setjmp`/`longjmp` in C).

### Spawning fibers

```
fiber.spawn $func : [t*] -> unreachable
```
where `$func : [t*] -> unreachable`

`fiber.spawn` allocates a new fiber and transfers control to it, calling `$func` on the new fiber with the arguments on the value stack.
If `$func` would return or throw an exception, it instead traps.

Note that this control transfer means the current frame of the current fiber becomes no longer accessible.
Thus the engine can clean it up.
In fact, the engine can clean up all frames up to the most recent jump reference.
If there are no such jump references, it can even clean up the entire fiber.
It can do so immediately or do so in parallel or defer until the associated resources are required.

Design note: Prior proposals separated allocation from initial transfer, but in reviewing existing implementations of runtimes with stack switching we found that it was common for the runtime to not actually allocate a physical fiber until it was time to actually transfer to control to the new fiber.
Integrating this observation into this proposal removed the need to determine how to create a new fiber in a suspended state, which was a point of complexity in all prior proposals (where no proposal, for example, could create a fiber that was able handle a thrown exception).

#### Switching fibers

```
fiber.br index $labeltype : [t* jumpref] -> unreachable
```
where `$labeltype = [t*]` (or a function type where the result type is ignored)

This instruction transfers control to the label at `index` in the given jump reference's table within the frame referenced by the jump reference.
The instruction traps if the jump reference is invalid (i.e. the frame no longer exists), the frame belongs to the current fiber, the target jump reference is *not* the most recent jump reference on its fiber, the index is invalid, or `$labeltype` does not match the type of the corresponding label.

Note that, like `fiber.spawn`, this instruction makes the current frame of the current fiber no longer accessible.

Extension/variation: `fiber.br_long` relaxes the requirement that the target jump reference be the most recent jump reference on its fiber.

#### Exceptions

```
jump.rethrow_long : [jumpref] -> unreachable
fiber.rethrow : [jumpref] -> unreachable
fiber.rethrow_long : [jumpref] -> unreachable
```
where the instruction is within a `catch`/`catch_all` block

Rethrows the caught exception from the site of `jump.call_with_new` referenced by the jump reference, with restrictions corresponding to those of the obvious corresponding instruction.

Design note: there is no `jump.throw` or the like because the same functionality can be implementing by `jump.br_long` and the like (devoting a chosen index to the exception at hand).

## JS API

Conceptually speaking, a call to JS from a wasm fiber should be able to take place on that fiber.
This could cause a call from within a JS function to return before the called function completes.
JS is not written with this possibility in mind, and as such it can violate many important program invariants that would otherwise hold.
Consequently, our JS API imposes restrictions to prevent such out-of-order returns.

Semantically speaking, when wasm calls into JS that then calls into wasm, the fiber remains the same.
In particular, if the embedder needs to switch to a special embedder fiber in order to execute JS code (e.g. because only embedder fibers are protected by virtual memory and safe to call potentially-overflowing C++ from), then it is the embedders responsibility to switch back to the relevant wasm fiber if that JS calls back into wasm.
For a given thread of control, there is an obvious chronological order between incomplete JS-to-wasm calls.
If an incomplete JS-to-wasm call *that is not the most recent* would return or throw an exception, it instead traps.
When a trap occurs (for any reason), control is transferred to the fiber with the most recent incomplete JS-to-wasm call, and the trap is converted to a JS exception and thrown from that call.
Note that, if control was on a different fiber, that fiber's contents must remain in tact *except* for all frames that are no longer accessible (which necessarily includes the frame of the trapping instruction).

## Examples

We look at three examples in order of increasing complexity and sophistication: a yield-style generator, cooperative threading and handling asynchronous I/O.

### Yield-style generators

The so-called yield style generator pattern consists of a pair: a generator function that generates elements and a consumer that consumes those elements. When the generator has found the next element it yields it to the consumer, and when the consumer needs the next element it waits for it. Yield-style generators represents the simplest use case for stack switching in general; which is why we lead with it here.

#### Generating elements

In Kotlin, one can write

```
fun fibonacci(): Iterator<Int> = sequence {
  var i: Int = 0
  var j: Int = 1
  while (true) {
    yield(i)
    val k: Int = i+j
    i = j
    j = k
  }
}.iterator()
```

With this proposal, every time you call `fibonacci.iterator()`, Kotlin can allocate a new fiber that will execute through the above loop each time you call `next()` on the iterator, pausing and providing a value once it hits a `yield` operation.
Due to JVM limitations, Kotlin already supports its coroutines via state-machine conversion, but the team has expressed significant interest in using stack-switching should they be made available in WebAssembly.

While Kotlin's runtime provides a great deal of infrastructure enabling users to have significant control over how coroutines are executed and scheduled, here we will boil this down to two key pieces of information that the runtime provides: what fiber the generator is running on, and whom it should switch to when it yields the next value (which can vary each time it yields).
In WebAssembly, supposing for simplicity that this information is given explicitly as arguments, this generator can be written with the following:

```
(func $boxInt (param i32) (result $KotlinObject))
(type $Iterator (struct ($hasNext   ([(ref $Iterator)] -> [i32]))
                        ($next      ([(ref $Iterator)] -> [(ref $KotlinObject)]))))
(type $FiberIterator (struct (sub $iterator)
                             ($hasNext    ([(ref $Iterator)] -> [i32]))
                             ($next       ([(ref $Iterator)] -> [(ref $KotlinObject)]))
                             ($generate   ([jumpref] -> unreachable))
                             ($finished   i32)
                             ($got_next   i32)
                             ($saved_next (ref null $KotlinObject))
                             ($gen_next   jumpref)))
```

`$boxInt` above simply boxes integers into Kotlin objects.
The type `$Iterator` is a simplified representation of the `Iterator` interface with two methods: `hasNext(): Boolean` and `next(): Object`.
The type `$FiberIterator` is the class implementing `Iterator` using stack-switching.
It adds a method `$generate`, which is instantiated with the relevant generating function to be called the first time the iterator is actually used, passing it the `jumpref` to yield the first value to.

```
(func $fibonacci (result (ref $Iterator))
  (return (struct.new $FiberIterator (func.ref $FiberIterator_hasNext)
                                     (func.ref $FiberIterator_next)
                                     (func.ref $fibonacci_generate)
                                     (i32.const 0)
                                     (i32.const 0)
                                     (ref.null $KotlinObject)
                                     (ref.null jumpref)))
)
```

The `fibonacci` function allocates a `FiberIterator`.
The only part of this allocation that is specific to `fibonacci` is the reference to `$fibonacci_generate`, which we will discuss after first presenting `FiberIterator`'s implementations of the `Iterator` methods `hasNext` and `next`.

```
(func $FiberIterator_next (param $this (ref $Iterator))
                          (result i32)
                          (local $self (ref $FiberIterator))
                          (local $next (ref null $KotlinObject))
   (local.set $self (ref.cast $FiberIterator (local.get $this)))
   (if (call $FiberIterator_hasNext (local.get $self))
     (then
       (local.set $next (struct.get $saved_next (local.get $self)))
       (struct.set $saved_next (local.get $self) (ref.null $kotlinObject))
       (struct.set $got_next (i32.const 0))
       (return (local.get $next))
     )
   (else
     (unreachable) ;; throw some Kotlin exception
   )
)
```

The `FiberIterator` implementation of `next` above is just boiler plate.
It invokes `hasNext`, which updates the state of `FiberIterator`, and then checks that state and modifies it appropriately.

```
(func $FiberIterator_hasNext (param $this (ref $Iterator))
                             (result i32)
                             (local $self (ref $FiberIterator))
   (local.set $self (ref.cast $FiberIterator (local.get $this)))
   (if (struct.get $finished (local.get $self))
     (return (i32.const 0))
   )
   (if (struct.get $got_next (local.get $self))
     (return (i32.const 1))
   )
   (block $finished []
     (block $generated [(ref null $KotlinObject) jumpref]
       (jump.call_with_new $FiberIterator_gen_next $finished $generated (local.get $self))
       (unreachable)
     ) ;; $generated : [(ref null $KotlinObject) jumpref]
     (struct.set $saved_next (local.get $self)) ;; pops $KotlinObject off stack
     (struct.set $gen_next (local.get $self)) ;; pops jumpref off stack
     (struct.set $got_next (i32.const 1))
     (return (i32.const 1))
   ) ;; $finished : []
   (struct.set $finished (i32.const 1))
   (return (i32.const 0))
)
```

The above implementation of `hasNext` is more interesting because, if a next value in fact needs to be generated, it creates a jump reference for receiving that value on the current stack.
That jump reference is set up to handle two situations: the generator is finished, and the generator found a value.
In either case, it updates the state of the `FiberIterator` and returns appropriately.

```
(func $FiberIterator_gen_next (param $self (ref $FiberIterator))
                              (param $consume_next jumpref)
                              (local $gen_next jumpref)
  (local.get $gen_next (struct.get $gen_next (local.get $self)))
  (if (ref.is_null (local.get $gen_next))
    (then
      (fiber.spawn $FiberIterator_spawn (local.get $self) (local.get $consume_next))
    )
    (else
      (struct.set $gen_next (ref.null jumpref))
      (fiber.switch 0 [jumpref] (local.get $consume_next) (local.get $gen_next))
    )
  )
)
```

The intermediary function above then uses that new jump reference.
If there is not already a fiber and associated jump reference generating the values, it spawns a new fiber.
Otherwise it switches to the fiber generating the values.
In either case, the generating fiber is handed the up-to-date jump reference to transfer control back to.
 
```
(func $FiberIterator_spawn (param $self (ref $FiberIterator))
                           (param $consume_next jumpref)
  (return_call_ref (struct.get $generate (local.get $self)) (local.get $self) (local.get $consume_next))
)
```

The above intermediary function is called when the generating fiber is first created.
It simply looks up the `$generate` function the `FiberIterator` was allocated with and tail calls it.

```
(func $fibonacci_generate (param $consume_next jumpref)
                          (local $i i32) (local $j i32) (local $k i32)
  (local.set $i (i32.const 0))
  (local.set $j (i32.const 1))
  (loop $loop
    (block $gen_next [jumpref]
      (jump.call_with_new $FiberIterator_yield $gen_next (local.get $consume_next) (call $boxInt (local.get $i))
      (unreachable)
    ) ;; $gen_next : [jumpref]
    (local.set $consume_next) ;; pop jumpref off stack
    (local.set $k (i32.add (local.get $i) (local.get $j)))
    (local.set $i (local.get $j))
    (local.set $j (local.get $k))
    (br $loop)
  )
  ;; unreachable, but the following is included for clarity
  (fiber.switch 0 [] (local.get $consume_next))
)
```

In the case of `fibonacci`, that function is `$fibonacci_generate`, given above.
Note that every use of `yield` in the `fibonacci` generator has been implemented with a `jump.call_with_new $FiberIterator_yield`, providing the label of the control point immediately after the yield.
This control point then updates the `jumpref` to yield control to and proceeds to generate more values.

```
(func $FiberIterator_yield (param $consume_next jumpref)
                           (param $next (ref null $KotlinObject))
                           (param $gen_next jumpref)
  (fiber.switch 1 [(ref null $KotlinObject) jumpref] (local.get $next) (local.get $gen_next) (local.get $consume_next))
)
```

The above function `$FiberIterator_yield` simply switches to the consumer, handing it the yielded object as well as the new jump reference to switch back to in order to get more values.
Notice that it uses index `1`.
This is because index `0` is for when the generator has finished.
Although our `fibonacci` example never finishes generating values, we show that—should it finish—it would switch using index `0`, providing nothing (including no jump reference to switch back to as there are no more values to generate).

Note that this approach generalizes well to more complex examples, where say there are multiple ways to yield values (e.g. `yieldInt`, `yieldDouble`, ...), or where there are "bidirectional" effects such as `remove()`—informing the generator to remove the last value that was generated (though that only makes sense for mutable-data-structure generators).

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
