# Hacking LLVM

The process of hacking LLVM as an absolute beginnner with no compilers knowledge.

## The problem I'm trying to solve

In a custom RISC-V architecture I'm trying to program, there are two disctinct address spaces. Loads from one address space are significantly slower than loads from the other. The goal is modify the the cost model and instruction scheduler to hide latencies, by schduling all the slow loads together with minimum immediate dependencies.

## Understanding LLVM design and learning to build & use it

Understanding high-level architecture is central for efficiently navigating thorugh the plethora of documentation. Found these resource very helpful to get a conceptual understanding of LLVM and why it stands apart from other similar projects:

- [Intro to LLVM](http://www.aosabook.org/en/llvm.html) by the lead architect of LLVM.
- [LLVM for Grad Students](https://www.cs.cornell.edu/~asampson/blog/llvm.html) by Adrian Sampson, professor in CS at Cornell. This also does a deep dive by leveraging LLVM library based design to make it do cool stuff, along with providing a conceptual overview of LLVM.

Key ideas to understand:
- Frontend --> Optimizer --> Backend.
- IR is the center piece in LLVM compiler optimizations.
- Optimizer can be extended by writing cutom IR->IR transorfmations, also known as "passes". These passes can be out-of-source!
- Tablegen is used to implement abstarct backend optimization alogrithms, while also using target specific data.
- Modular library based design.
- Agile software engineering -- not afraid of breaking backward compatibility!

## LLVM IR

LLVM IR is a [Static Single Assignemnt](https://en.wikipedia.org/wiki/Static_single_assignment_form) based representation of the program compiled by LLVM. LLVM IR is defacto format for optimizations in LLVM.

## Writing LLVM Passes

[Writing an LLVM Pass](http://llvm.org/docs/WritingAnLLVMPass.html)
[WRITING AN LLVM PASS (Nice examples)](http://laure.gonnord.org/pro/research/ER03_2015/lab3_intro.pdf) 

## Dev mailing list

Dev mailing list is active and really helpful. It's often should be the first point of renference to get an informed opinion on the approaches to consider. I got to know that LLVM has the concept of address space which seems to be precisely what I needed. 

## LLVM Address Spaces

Clang supports multiple address spaces. For example, following code annotates the loads to be from specific address spaces:
```c++
#define __remote __attribute__((address_space(1)))

int kernel_vec_add(__remote int *dst, const __remote int *src, const int size) {

  for (int i=0; i < size; ++i) {
    dst[i] += src[i];
  }

  return 0;
}
```
The loop above results in IR like this (notice address space annotations on pointers):
```llvm
for.body:                                         ; preds = %entry, %for.body
  %i.07 = phi i32 [ %inc, %for.body ], [ 0, %entry ]
  call void @llvm.dbg.value(metadata i32 %i.07, metadata !20, metadata !DIExpression()), !dbg !23
  %arrayidx = getelementptr inbounds i32, i32 addrspace(1)* %src, i32 %i.07, !dbg !28
  %0 = load i32, i32 addrspace(1)* %arrayidx, align 4, !dbg !28, !tbaa !30
  %arrayidx1 = getelementptr inbounds i32, i32 addrspace(1)* %dst, i32 %i.07, !dbg !34
  %1 = load i32, i32 addrspace(1)* %arrayidx1, align 4, !dbg !35, !tbaa !30
  %add = add nsw i32 %1, %0, !dbg !35
  store i32 %add, i32 addrspace(1)* %arrayidx1, align 4, !dbg !35, !tbaa !30
  %inc = add nuw nsw i32 %i.07, 1, !dbg !36
  call void @llvm.dbg.value(metadata i32 %inc, metadata !20, metadata !DIExpression()), !dbg !23
  %exitcond = icmp eq i32 %inc, %size, !dbg !24
  br i1 %exitcond, label %for.cond.cleanup, label %for.body, !dbg !26, !llvm.loop !37
}
```

Semantics of different address spaces are intended to be target specific. Pointers in LLVM can have numbered address space attribute. Default address space is address spaces 0. In the program above, pointers annotated with `__remote` have `addrspace(1)` attribute. 
 
If a target supports casting between different address spaces, it must implement [`addrspacecast`](http://llvm.org/docs/LangRef.html#addrspacecast-to-instruction) instruction. AMD GPU's address space cast seems to nice reference for implementing one: https://gitlab.redox-os.org/redox-os/llvm/commit/f9fe65992283ac87b676f51b969de90aee63738e.
 
Pro tip: Searching for "address space" in [LLVM Language Reference](http://llvm.org/docs/LangRef.html) provides wealth of possibilities with address spaces in LLVM.
