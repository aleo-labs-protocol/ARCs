---
arc: 4
title: Flagged Operations for Conditional Code
authors: bendyarm, d0cd, acoglio, mikebenfield
discussion: [https://github.com/ProvableHQ/ARCs/discussions/89](89)
topic: Protocol
status: Draft
created: 2025-03-09
---

## Abstract

Some Aleo instructions can halt when passed certain arguments.
This halting behavior makes those instructions unusable
by a program that wants to try something else if a halt is detected.

We propose to add nonhalting variants of each such instruction.  
The nonhalting variants additionally return an error flag.  
This change would be strictly additive; none of the existing instructions
or Aleo Instructions programs would change.

### Why does Aleo need this?

The need for these instructions is to enable proper compilation of `if-then-else` in a circuit.  In a circuit, both the `then` statements and the `else` statements are executed (both **must** be executed), and then the `if` condition is used to select which results will be used for further computation.  If a non-taken branch halts, it still halts the entire circuit, which is not the desired behavior.

Especially for code compiled from Leo, this could lead to unexpected program behavior, including possible inaccessibility of funds.  The reason Leo code is especially vulnerable is that it has an `if-then-else` statement that currently has a different semantics than what a Leo programmer would expect (the non-taken branch being executed).  However, the problem also exists in Aleo Instructions of not being able to write the equivalent of an `if-then-else` statement where there are halting instructions in the `then` statements or `else` statements.

However, even for Aleo Instructions programs not compiled from Leo, these operations will allow developers to roll their own `if-then-else` code, and will enable checking if an instruction would have halted without actually triggering a halt.

### Why does adding nonhalting variants solve this problem?

By using nonhalting instruction variants in `then` and `else` branches, a non-taken branch will not cause a halt, even though it is still executed.  When the `if` condition selects which results to use, at that time the code will trigger a halt if the taken branch would have halted if it had used a halting instruction (i.e., if the error flag was set by the nonhalting instruction).

### Simple example in Leo code.

```leo
transition main(public x: field, y: field) -> field {
    if y == 0field {
        return x;
    } else {
        return x / y;
    }
}
```
If called with `y` equal to `0field`, this program will halt because the division is always executed, whereas the expected result would be `x`.  There is currently no way to compile this Leo program to Aleo Instructions that work as expected unless there is a nonhalting `div` operation.  (We are calling this proposed nonhalting Aleo Instructions opcode `div.flagged`.)

After this ARC is implemented, the Leo compiler can be improved so that this program runs as expected.

## Specification

Each flagged operation is identical to its corresponding current operation
when called with arguments that would be nonhalting, except it returns
a second return value of type boolean that is **false** (meaning that the
current operation would not have halted).  When called with arguments
that would have been halting, it generally returns 0 (of the appropriate
output type) and **true** ("would have halted") for the second return value.

The flagged operations are different from wrapped (e.g. `abs.w`) or
lossy (e.g. `cast.lossy`) operations.  It is important that the flagged
operation have the same semantics as the current halting instruction
except for the halting behavior and extra return value, for ease of
use by compilers.

Note that most opcodes compile into different operations based on the types
of the operands, and those operations sometimes have different halting
behavior.  For example, `add` cannot halt when given field, group or scalar
arguments, but it can halt when given integer arguments.  So `add.flagged`
will not be implemented for field, group or scalar arguments. We add notes
for these cases in the table below.

| Current Halting Opcode | New Flagged Opcode | notes |
|:-------------------:|:-----------------------:|:--------------------------:|
| abs | abs.flagged |
| add | add.flagged | add.flagged implemented only for integer types.
| assert.eq | is.eq | We probably don't need a separate assert.(n)eq.flagged;
| assert.neq | is.neq | instead, just use the existing is.(n)eq.
| cast | cast.flagged | As of now, docs don't say what causes cast to halt. TBD.
| div | div.flagged | div can halt on underflow and divide by zero.
| div.w | div.w.flagged | div.w can halt on divide by zero.
| inv | inv.flagged |
| mod | mod.flagged |
| mul | mul.flagged | mul.flagged implemented only for integer types.
| neg | neg.flagged | neg.flagged implemented only for signed integer types.
| pow | pow.flagged | pow.flagged implemented only for integer types.
| rem | rem.flagged | rem can halt on underflow and divide by zero.
| rem.w | rem.w.flagged | rem.w can halt on divide by zero.
| shl | shl.flagged | As of now, Aleo docs don't say what causes shl or shr to halt.
| shr | shr.flagged | But the Leo docs state that shl and shr halt if the shift distance
||                            | exceeds the bit size of the first argument, and also shl halts if the
||                            | shifted result does not fit within the type of the first argument.
| sqrt | sqrt.flagged |
| sub | sub.flagged | sub.flagged implemented only for integer types.

### Test Cases

The following note applies to all tests of Aleo Instructions.  For all tests, both literal constant arguments
and variable arguments should be tested.  For example, if there are two arguments, we need to test all four
combinations.  This is because the Aleo Instructions compiler generates more optimized code for literal
constant arguments, and the circuits can differ substantially.

One or more arguments that causes halting for a current halting opcode should be used as an input
for the equivalent new flagged opcode to make sure it doesn't halt.

An assortment of arguments that do not cause halting for a current halting opcode should be
used as input to both halting and flagged operations to make sure they return the same value
(other than the halting flag, of course).

### Audit and Formal Verification

We intend to have an audit of the implementation of the new opcodes, as well as to formally
verify the implementation.

## Reference Implementations

This ARC will not be fully implemented unless approved.  However, there is a proof-of-concept (PoC)
implementation with the following components:

* In a snarkVM branch, Aleo Instructions `div.flagged` and `inv.flagged` have been implemented, both for
only field operands.

* In a Leo branch, there is a PoC change to the Leo compiler that compiles uses of `/` and `inv()` (that 
are applied to field operands and that are in conditionals) to the new Aleo Instructions `div.flagged` and
`inv.flagged`, along with the appropriate logic to make sure halts happen only when appropriate.

This To try out this PoC implementation, first make sure you have already installed the
preliminaries [for snarkVM](https://developer.aleo.org/guides/introduction/getting_started)
and [for Leo](https://docs.leo-lang.org/getting_started/installation).

```
// cd to the parent of where you want this copy of snarkVM to go.
git clone --branch poc-flagged https://github.com/bendyarm/snarkvm
cd snarkvm
cargo install --path .

// cd to the parent of where you want this copy of Leo to go.
git clone --branch div-flagged https://github.com/ProvableHQ/leo
cd leo
cargo install --path .
```
At this point you can use the [snarkVM CLI](https://developer.aleo.org/guides/aleo/commands) and the [Leo CLI](https://docs.leo-lang.org/cli/overview) to make programs and run them.

This PoC fixes [this Leo issue](https://github.com/ProvableHQ/leo/issues/27482).
Similarly, implementing the full ARC will fix similar issues for all the haltable operations
when they are in conditional code.

## Dependencies

The main change is to the snarkVM repository.

Any tool that needs to know what all the opcodes are needs to be changed.  For example, the Aleo
Explorer and the Provable SDK.

After the ARC is implemented, the Leo compiler can use the new opcodes to improve the quality of compiled code.
However, if the Leo compiler is not changed, the resulting code will continue to work the same way it did before the ARC was implemented.

### Backwards Compatibility

For snarkVM, this change is strictly additive, and does not require any changes to existing Aleo Instructions programs. Also, use of the new opcodes is optional; the change does not require any changes to how you are writing new Aleo Instructions programs.

If there are any Leo programs that were actually depending on the behavior of halting operations in non-taken branches causing halts, then those Leo programs would behave differently after being recompiled.  We believe this case is very unlikely; it is more likely that the recompilation would fix latent bugs that just had not yet been encountered.


## Security & Compliance

This change does not affect security or regulatory issues.


## References

Example Leo bug:
https://github.com/AleoHQ/leo/issues/27482
