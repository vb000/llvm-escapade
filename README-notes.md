# Hacking LLVM

The process of hacking LLVM as an absolute beginnner with no compilers knowledge.

## The problem

In a custom RISC-V architecture I'm trying to program, there are two disctinct address spaces. Loads from one address space are significantly slower than loads from the other. The goal is modify the the cost model and instruction scheduler to hide latencies by schduling all the slow loads together with minimum immediate dependencies.

## Get source, compile and run some programs.

Learned the high-level architecture of LLVM compiler and how custom passes work.

## Dev mailing list

Dev mailing list is active and really helpful. It's often should be the first point of renference to get an informed opinion on the approaches to consider. I got to know that LLVM has the concept of address space which seems to be precisely what I need. Clang supports multiple address spaces. For example, this annotates the loads to be from specific address spaces:
```c++
#define __remote __attribute__((address_space(1)))

int kernel_vec_add(__remote int *dst, const __remote int *src, const int size) {

  for (int i=0; i < size; ++i) {
    dst[i] += src[i];
  }

  return 0;
}
```
The loop above results in IR like this (notice address space annotations in the IR):
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

## LLVM IR

LLVM IR is a [Static Single Assignemnt](https://en.wikipedia.org/wiki/Static_single_assignment_form) based representation of the program compiled by LLVM. LLVM IR is defacto format for optimization in LLVM.
