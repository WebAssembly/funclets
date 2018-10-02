# Funclets: Flexible Intraprocedural Control Flow

### Background

Tail calls can be used to create arbitrary control flow patterns, however tail calls are difficult to implement as efficiently as intraprocedural control flow. The key difference is that in intraprocedural control flow, implementations can easily determine all the entry points into a region of code. Statically determining the callees of a function requires full interprocedural knowledge in the best case, and is undecidable in the worst.

Also, one of WebAssembly's goals is to be easy to target from a wide variety of compilers, and to avoid biasing towards one style of compiler over another. Currently WebAssembly makes one very popular style of compiler awkward, and it sometimes leads to the use of inefficient code.

And, common dynamic-language compilation techniques involve dynamically patching control flow paths within a function, which is something that WebAssembly may want to support in the future. Such functionality isn't included here, but the proposal here would be a natural base on which to design such a feature.

The following is a proposal for a solution, which carefully preserves key properties of wasm: linear-time decoding and validation, fast SSA construction, and efficient baseline compilation. The issue here is the first step in [phase 0 of the CG process](https://github.com/WebAssembly/meetings/blob/master/process/phases.md#0-pre-proposal-community-group).

### Overview

This adds a new control-flow construct called a *funclet region*. A funclet region contains a sequence of *funclets*, which are essentially small functions that can tail call each other, passing arguments on the operand stack.

Funclet regions nest within and can be nested within wasm's existing control-flow operators, so producers can use the existing control-flow constructs when they make sense, and funclet regions when they need extra flexibility.

### New Opcodes

 - `funclet_region`

   | Field | Type | Description |
   | ----- | ---- | ----------- |
   | `signature` | signature | the funclet region's signature, similar to a `block` signature
   | `num_funclets` | varuint32 | the number of funclets belonging to the funclet region (and must be non-zero)

   This starts a funclet region with `num_funclets` funclets, and starts the first funclet.

   Similar to `block`, `funclet_region` adds a control flow nesting level, with branches (`br`, `br_if`, etc.) to this level branching to the end of the funclet region. The `signature` immediate has the same role as `block`'s signature.

 - `funclet_sig`

   This declares an explicit type signature for a funclet, which is needed if the funclet isn't branched to from above. This opcode may only appear at the beginning of a funclet.

   | Field | Type | Description |
   | ----- | ---- | ----------- |
   | `signature` | signature | the signature for the funclet
   | `num_preds` | varuint32 | the number of backward funclet callers of the funclet

   See the SSA construction section below for an explanation of the `num_preds` field.

 - `funclet_call`, `funclet_call_if`

   | Field | Type | Description |
   | ----- | ---- | ----------- |
   | `delta` | varint32 | the relative index of a funclet to tail call

   These are the unconditional and conditional instructions which allow one funclet in to tail-call another in the same funclet region. `funclet_call` is to `funclet_call_if` as `br` is to `br_if`.

   These are only valid within a funclet region.

 - `funclet_call_table`

   | Field | Type | Description |
   | ----- | ---- | ----------- |
   | target_count | varuint32 | number of entries in the target_table
   | target_table | varint32* | target entries that indicate the relative index of a funclet to tail call
   | default_target | varint32 | the relative index of a funclet to tail call in the default case

   `funclet_call_table` is to `funclet_call_if` as `br_table` is to `br_if`.

### Funclets

A funclet is ended by an `end` or by any unconditional control transfer (`br`, `br_table`, `return`, `funclet_call`, `funclet_call_table`, or `unreachable`), after which the next funclet begins. A funclet region ends when the last of its funclets ends.

When a funclet performs a funclet call or an `end` to another funclet, all values on the stack above the point where the funclet region was entered are the arguments in the tail call to the callee. In the case of an `end` in the last funclet in a funclet region, the funclet region exits in the same manner as `block`.

Funclets don't have local variables of their own, but they can access the local variables of the containing function.

### Linear-time Validation

The main requirement for any binary code format to have linear-time type checking is to ensure that when decoding reaches a control-flow merge point, the types of all variables entering the merge point are known.

Every funclet has a signature, which contains just argument types, and no results, because control transfers between funclets are always tail calls.

For now, if the first funclet in a funclet region has no `funclet_sig`, it has no arguments. This may be generalized to match the proposal to make `block`s accept arguments (which is part of the [multi-value proposal](https://github.com/WebAssembly/multi-value/)).

When validation reaches a funclet called only from above, and there's at least one call from above, then all the incoming funclet calls will have been seen, so the type can be inferred.

When validation reaches a funclet that can be called from below, or is not reachable from any call, it must begin with a `funclet_sig` instruction, which provides a signature as an immediate value.

### SSA Construction

SSA construction within funclet regions can be performed using the well-known "Simple and Efficient Construction of Static Single Assignment Form" algorithm by Braun et al.

 - https://pp.info.uni-karlsruhe.de/uploads/publikationen/braun13cc.pdf
 - https://pp.ipd.kit.edu/uploads/publikationen/ullrich13bachelorarbeit.pdf

`funclet_sig`s contain an immediate value of the number of backwards callers, to allow funclets to be "sealed" (using terminology from the paper) as soon as the last caller of a funclet is processed. This allows for greater on-the-fly optimization.

### Implementation Options

While some implementations may implement flexible intraprocedural control flow directly, it would also be feasible to translate funclets as individual functions using regular interprocedural tail calls, as an implementation detail. Funclets need to be able to access the local variables of the enclosing function, however this can be arranged either by passing them as arguments, or by passing a closure pointer as a hidden argument.

### Examples

A very simple loop:

```
    funclet_region 1  # start a funclet region with 1 funclet
      funclet_sig     # the top of the loop can be called from below
      ...
      funclet_call_if 0 # loop backedge, calls to funclet 0 (the same funclet, so the delta is 0)
      end
```

A very simple control flow diamond (if-else):

```
    funclet_region 2    # start a funclet region with 2 funclets
      ...               # define the if condition value
      funclet_call_if 1 # test the condition, call to the false arm or fall through to the true arm
      ...               # the true arm (still in the first funclet)
      br 0              # branch to the `end`. This ends the first funclet, the second follows.
      ...               # the false arm, starting the second funclet
      end
```

A wasm if-else nested inside a funclet region. Note that nested constructs nest inside of funclets, so this only uses a single funclet:

```
    funclet_region 1  # start a funclet region with 1 funclet
      ...
      if
      ...
      else
      ...
      end             # end the if-else
      ...
      end             # end the funclet region
```

An example with an interesting funclet region signature:

```
    funclet_region 2 (result i32) (result f32) # start a funclet region with 2 funclets, and two results
      ...
      funclet_call_if 1 # start a CFG diamond
      ...               # the true arm
      i32.const 2       # arguments to pass to the tail call
      f32.const 3.0
      br 0              # exit the funclet region
      ...               # funclet 1, the false arm
      i32.const 4       # validation can now check the argument types
      f32.const 5.0
      end
```

An example with an interesting funclet signature called from above:

```
    funclet_region 3 (result i32) (result f32) # start a funclet region with 2 funclets, and two results
      ...
      funclet_call_if 1 # start a CFG diamond
      ...               # the true arm
      i32.const 2       # arguments to pass to the tail call
      f32.const 3.0
      funclet_call 1    # call funclet 2
      ...               # funclet 1, the false arm
      i32.const 4       # validation can now check the argument types
      f32.const 5.0
      end               # call funclet 2
      ...               # funclet 2, takes two arguments
      end
```

An example with an interesting funclet signature called from below:

```
    funclet_region 3  # start a funclet region with 3 funclets
      funclet_call 2  # call funclet 2
      funclet_sig (param i32) (param f32)   # funclet signature
      ...
      br 0            # exit the funclet region
      i32.const 2     # arguments to pass to the tail call
      f32.const 3.0
      funclet_call -1  # call funclet 1, passing it two arguments
```
