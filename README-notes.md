# Hacking LLVM

The process of hacking LLVM as an absolute beginnner with no compilers knowledge.

## The problem

In a custom RISC-V architecture I'm trying to program, there are two disctinct address spaces. Loads from one address space are significantly slower than loads from the other. The goal is modify the the cost model and instruction scheduler to hide latencies by schduling all the slow loads together with minimum immediate dependencies.
