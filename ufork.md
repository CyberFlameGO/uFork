# uFork (Actor Virtual Machine)

The **uFork** Actor Virtual Machine performs
interleaved execution of threaded instruction streams.
Instruction streams are not assumed to be
arranged in consecutive memory locations.
Instead, each instruction contains a "pointer"
to the subsequent instruction,
or multiple pointers in the case of conditionals, etc.

This is combined with a lightweight computational context
(such as IP+SP) that makes it efficient
to enqueue the context after each instruction.
Then the next context can be dequeued and executed.
Thus an arbitrary number of instruction streams can be executed,
interleaved at the instruction level.

This interleaved instruction execution engine
is used to service an actor message-event queue.
If the target of the event at the head of the queue
is not already busy handling a prior event,
a new computational context (continuation, or "thread")
is created (and added to the continuation queue)
to execute instructions in the target actor's behavior.
If the event target is busy,
the event is recycled to the tail of the queue.
Since asynchronous actor messages may be arbitrarily delayed,
this does not change the semantics.

Effects caused by an actor's behavior (send, create, and become)
are applied to the system in an all-or-nothing transaction.
The instruction stream defining the actor's behavior
will end with a "commit" or "abort" instruction,
at which time transactional effects will either be applied or discarded.
Since these instructions have no "next instruction" field,
there is nothing to put on the continuation queue
and the stream ends (the "thread" dies).

The blog post
"[Memory Safety Simplifies Microprocessor Design](http://www.dalnefre.com/wp/2022/08/memory-safety-simplifies-microprocessor-design/)"
describes this architecture,
and the rationale behind it.

## Virtual Machine Semantics

The **uFork** _virtual machine_ is designed to support machine-level actors.
All instructions execute within the context of an actor handling a message-event.
There is no support for procedure/function call/return.
Instead actors are used to implement procedure/function abstractions.
There is no support for address arithmetic or load/store of arbitrary memory.
Mutation is always local to an actor's private state.
Immutable values are passed between actors via message-events.
External events (such as "interrupts")
are turned into message-events.

There are currently two variations of the uFork VM.
A [proof-of-concept implementation](c_src/vm.md) written in C,
and a [more robust implementation](ufork-wasm/vm.md) written in Rust/WASM.
The instruction set and internal representation has evolved
so the two implementations are no longer compatible.

## Garbage Collection

Cell memory (quads) are subject to machine-level garbage collection.
The garbage-collected _heap_ ranges from `START` up to (not including) `cell_top`.
The floor (currently `START`) may be moved upward to include additional "reserved" cells.
The ceiling (held in the variable `cell_top`) is extended upward
to expand the pool of available memory,
up to a limit of `CELL_MAX`.
The bootstrap image initially occupies cells up to `CELL_BASE`,
which determines the initial value of `cell_top`.

The garbage-collector maintains a _mark_ for each cell in the heap.
The mark can have one of four possible values:

  * `GC_GENX`: This cell is in use as of Generation X
  * `GC_GENY`: This cell is in use as of Generation Y
  * `GC_SCAN`: This cell is in use, but has not been scanned
  * `GC_FREE`: This cell is in the free-cell chain {t:Free_T}

The _current generation_ alternates between `GC_GENX` and `GC_GENY`.
Cells in the range \[`START`, `CELL_BASE`\) are initially marked `GC_GENX`.

### GC Algorithm

Garbage collection is concurrent with allocation and mutation.
An increment of the garbage collector algortihm runs between each instruction execution cycle.
The overall algorithm is roughly the following:

1. Swap generations (`GC_GENX` <--> `GC_GENY`)
2. Mark each cell in the root-set with `GC_SCAN`
    1. If a new cell is added to the root-set, mark it with `GC_SCAN`
3. Mark each newly-allocated cell with `GC_SCAN`
4. While there are cells marked `GC_SCAN`:
    1. Scan a cell, for each field of the cell:
        1. If it points to the heap, and is marked with the _previous_ generation, mark it `GC_SCAN`
    2. Mark the cell with the _current_ generation
5. For each cell marked with the _previous_ generation,
    1. Mark the cell `GC_FREE` and add it to the free-cell chain

## Inspiration

  * [SectorLISP](http://justine.lol/sectorlisp2/)
  * [Ribbit](https://github.com/udem-dlteam/ribbit)
    * [A Small Scheme VM, Compiler and REPL in 4K](https://www.youtube.com/watch?v=A3r0cYRwrSs)
  * [The LISP 1.5 Programmer's Manual](https://www.softwarepreservation.org/projects/LISP/book/LISP%201.5%20Programmers%20Manual.pdf)
  * [Schism](https://github.com/schism-lang/schism)
  * [A Simple Scheme Compiler](https://www.cs.rpi.edu/academics/courses/fall00/ai/scheme/reference/schintro-v14/schintro_142.html#SEC271)
  * [Parsing Expression Grammars: A Recognition-Based Syntactic Foundation](https://bford.info/pub/lang/peg.pdf)
    * [OMeta: an Object-Oriented Language for Pattern Matching](http://www.vpri.org/pdf/tr2007003_ometa.pdf)
    * [PEG-based transformer provides front-, middle and back-end stages in a simple compiler](http://www.vpri.org/pdf/tr2010003_PEG.pdf)
