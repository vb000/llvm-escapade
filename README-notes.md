# Hacking LLVM

The process of hacking LLVM as an absolute beginnner with no compilers knowledge.

## The problem

In a custom RISC-V architecture I'm trying to program, there are two disctinct address spaces. Loads from one address space are significantly slower than loads from the other. The goal is modify the the cost model and instruction scheduler to hide latencies by schduling all the slow loads together with minimum immediate dependencies.

## Get source, compile and run some programs.

## Dev mailing list

Dev mailing list is active and really helpful. It's often should be the first point of renference to get an informed opinion on the approaches to consider. I got to know that LLVM has the concept of address space which seems to be precisely what I need. Clang supports multiple address spaces. For example, this annotates the loads to be from specific address spaces:
  ```c++
  #define __remote __attribute__((address_space(1)))
  for (...) {
    ((__remote int*) dst)[iter_x] = ((__remote int*) src)[iter_x];
  }
  ```
Which results in IR like this (notice address space annotations in the IR):
  ```llvm
  %arrayidx = addrspacecast i32* %arrayidx18 to i32 addrspace(1)*
  %5 = load i32, i32 addrspace(1)* %arrayidx, align 4, !tbaa !3
  %arrayidx519 = getelementptr inbounds i32, i32* %dst, i32 %iter_x.021
  %arrayidx5 = addrspacecast i32* %arrayidx519 to i32 addrspace(1)*
  store i32 %5, i32 addrspace(1)* %arrayidx5, align 4, !tbaa !3
  ```
