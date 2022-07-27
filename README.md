An informal space to discuss and design Venom IR, an LLVM-like project but for blockchain VMs, starting with EVM

# Overall Design Goals

- Generic IR for smart contract languages
- Target EVM initially but be general enough to target other blockchain VMs in the future
- Good optimization properties. Allowing for tracking of VM side effects, optimal stack scheduling, static memory allocation, aliasing, analysis
- Amenable to security analysis. Should be able to reconcile source code with IR, analysis of IR should yield same or better results as analysis of source code.

## Problem: stack scheduling

Stack scheduling is difficult using a phi-style SSA representation (that is, an SSA representation which uses phi functions to represent coalescing of variables after join points). The reason is that register layout (the mapping between variables and registers) is static between instructions; stack layout can vary every instruction.

A simple, annotated example to demonstrate that allocation for phi functions is more straightforward with registers than with stack:

```c
%x0 = 1  // stack: x0; register: rax=>1
if (cond) {
    %x1 = 2 // stack: x1   ; register: rax=>2
} else {
    %x2 = 3 // stack: x2   ; register: rax=>3
    %y0 = 4 // stack: x2,y0; register: rax=>3,rbx=>4
}
%x0 = phi(%x1, %x2) // stack: ?, register: rax=>3,rbx=>4
```

### Solution: functional-style SSA

Instead of using phi-instructions to represent layout coalescing at join points, use a functional representation. This requires functions to describe their inputs at entry, and their outputs at exit:

```c
x = 1
y = 2
z = 3
if (cond) [x, y, z] {
    // stack: x, y, z
    x1 = y // stack: 2
    JOIN(x1) // jump to the join point, with x on the stack
} else {
    x1 = z // stack: 3
    t = 1 // can be eliminated because it dies before JOIN

    JOIN(x1) // jump to the join point, with x on the stack
}
```

This shifts responsibility for correct stack layout to the bodies of the branch.

TODO: Compare global scoping for LLVM variables, vs this local-style scoping for functional style SSA.

## Problem: memory allocation

We need to be able to analyse memory allocation, aliasing and memory writes at compile time.

### Solution: alloca builtin

Borrow LLVM's alloca instruction. Particularly if the call stack can be bounded, we can allocate all stack frames statically as well (as in vyper).


## Problem: reconciling source code with IR

For analysis by third party tools, we need to be able to map PCs not just to location in source code but also with locations in IR.

### Solution: publish mapping from opcodes to IR

Self explanatory

## Problem: performing type analysis on IR

For correctness analysis (and debuggers), we need to know the source code types corresponding to IR operations.

This is not entirely trivial because of pointer disambiguation, for instance if the start of an array `xs` is represented by a pointer `123` and we see `123` in the code: does `123` correspond to `xs` or `xs[0]`?

### Solution: propagate source code type information into the IR

Either by explicit annotations, or by using a `getelementptr` (gep) instruction which provides implicit tagging.

## Problem: analysis of VM side effects

For instance, being able to cache SLOADs, can never eliminate a CALL but maybe can eliminate a STATICCALL.

### Solution: tagging system for VM intrinsics

Maybe a syscall like system. TBD.

## References

- [Treegraph-based instruction scheduling for stack-based virtual machines](https://www.researchgate.net/publication/220369290_Treegraph-based_Instruction_Scheduling_for_Stack-based_Virtual_Machines). A stack scheduler for TinyVM
- [Lecture notes on Static Single Assignment form](https://www.cs.cmu.edu/~fp/courses/15411-f13/lectures/06-ssa.pdf). Some notes on a non-phi-style SSA.
