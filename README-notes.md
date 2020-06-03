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

- [Writing an LLVM Pass](http://llvm.org/docs/WritingAnLLVMPass.html)
- [WRITING AN LLVM PASS (Nice examples)](http://laure.gonnord.org/pro/research/ER03_2015/lab3_intro.pdf)

`MachineFunctionPass` seems promising for this problem. Following seems to work for RISCV:
```diff
diff --git a/llvm/lib/Target/RISCV/CMakeLists.txt b/llvm/lib/Target/RISCV/CMakeLists.txt
index 31a82be1981..521c43a82ea 100644
--- a/llvm/lib/Target/RISCV/CMakeLists.txt
+++ b/llvm/lib/Target/RISCV/CMakeLists.txt
@@ -17,6 +17,7 @@ tablegen(LLVM RISCVGenSystemOperands.inc -gen-searchable-tables)
 add_public_tablegen_target(RISCVCommonTableGen)

 add_llvm_target(RISCVCodeGen
+  VanillaPasses.cpp
   RISCVAsmPrinter.cpp
   RISCVCallLowering.cpp
   RISCVExpandPseudoInsts.cpp
diff --git a/llvm/lib/Target/RISCV/RISCV.h b/llvm/lib/Target/RISCV/RISCV.h
index f23f742a478..2ca2f7f5611 100644
--- a/llvm/lib/Target/RISCV/RISCV.h
+++ b/llvm/lib/Target/RISCV/RISCV.h
@@ -43,6 +43,9 @@ void initializeRISCVMergeBaseOffsetOptPass(PassRegistry &);
 FunctionPass *createRISCVExpandPseudoPass();
 void initializeRISCVExpandPseudoPass(PassRegistry &);

+FunctionPass *createRISCVVanillaPass();
+void initializeRISCVVanillaPass(PassRegistry &);
+
 InstructionSelector *createRISCVInstructionSelector(const RISCVTargetMachine &,
                                                     RISCVSubtarget &,
                                                     RISCVRegisterBankInfo &);
diff --git a/llvm/lib/Target/RISCV/RISCVTargetMachine.cpp b/llvm/lib/Target/RISCV/RISCVTargetMachine.cpp
index de71c01753d..4e171387756 100644
--- a/llvm/lib/Target/RISCV/RISCVTargetMachine.cpp
+++ b/llvm/lib/Target/RISCV/RISCVTargetMachine.cpp
@@ -174,6 +174,7 @@ void RISCVPassConfig::addPreEmitPass2() {
   // possibility for other passes to break the requirements for forward
   // progress in the LR/SC block.
   addPass(createRISCVExpandPseudoPass());
+  addPass(createRISCVVanillaPass());
 }

 void RISCVPassConfig::addPreRegAlloc() {
diff --git a/llvm/lib/Target/RISCV/VanillaPasses.cpp b/llvm/lib/Target/RISCV/VanillaPasses.cpp
new file mode 100644
index 00000000000..596391c236f
--- /dev/null
+++ b/llvm/lib/Target/RISCV/VanillaPasses.cpp
@@ -0,0 +1,30 @@
+#include "RISCV.h"
+#include "llvm/Pass.h"
+#include "llvm/CodeGen/MachineFunctionPass.h"
+#include "llvm/CodeGen/MachineFunction.h"
+#include "llvm/Support/raw_ostream.h"
+
+using namespace llvm;
+
+namespace {
+
+struct VanillaPass : public MachineFunctionPass {
+  static char ID;
+
+  VanillaPass(): MachineFunctionPass(ID) {}
+
+  bool runOnMachineFunction(MachineFunction &MF) override {
+    errs() << "VanillaPass: ";
+    errs() << MF.getName() << "\n";
+    return false;
+  }
+};
+
+}
+
+FunctionPass* llvm::createRISCVVanillaPass() {
+  return new VanillaPass();
+}
+
+char VanillaPass::ID = 0;
+static RegisterPass<VanillaPass> X("vanilla", "Vanilla Pass");
```

Based on [this](https://llvm.org/docs/WritingAnLLVMPass.html#registering-dynamically-loaded-passes), possibly, there is a way to load out-of-source passes to the LLVM backend (`llc`). TODO: check this out. 

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

## Scheduler Pass

An additional scheduler pass might be necessary to schedule slow loads together. This seems to be a nice reference:
https://llvm.org/doxygen/PostRASchedulerList_8cpp_source.html

## Things worth trying with scheduling

`--disable-sched-reg-pressure` (list-ilp)
`--max-sched-reorder` (list-ilp)
```
  --misched=<value>                                               - Machine instruction scheduler to use
    =default                                                      -   Use the target's default scheduler
 choice.
    =converge                                                     -   Standard converging scheduler.
    =ilpmax                                                       -   Schedule bottom-up for max ILP
    =ilpmin                                                       -   Schedule bottom-up for min ILP
    =shuffle                                                      -   Shuffle machine instructions alter
nating directions
  --misched-bottomup                                              - Force bottom-up list scheduling
  --misched-cluster                                               - Enable memop clustering.
  --misched-cutoff=<uint>                                         - Stop scheduling after N instructions
  --misched-cyclicpath                                            - Enable cyclic critical path analysis
.
  --misched-dcpl                                                  - Print critical path length to stdout
  --misched-fusion                                                - Enable scheduling for macro fusion.
  --misched-limit=<uint>                                          - Limit ready list to N instructions
  --misched-only-block=<uint>                                     - Only schedule this MBB#
  --misched-only-func=<string>                                    - Only schedule this function
  --misched-postra                                                - Run MachineScheduler post regalloc (
independent of preRA sched)
  --misched-print-dags                                            - Print schedule DAGs
  --misched-regpressure                                           - Enable register pressure scheduling.
  --misched-topdown                                               - Force top-down list scheduling
  --misfetch-cost=<uint>                                          - Cost that models the probabilistic r
isk of an instruction misfetch due to a jump comparing to falling through, whose cost is zero.
```

## Modifying Non-Blocking Load Latency

Starting with scheduler architecture:

RISCV backend seems to use the MIScheduler which is intending to be modern replacement of legacy ScheduleDAG. [Slide 15 here](https://llvm.org/devmtg/2016-09/slides/Absar-SchedulingInOrder.pdf) provides a great overview of the APIs.

In case latency cannot be modelled in tablegen, this is the function that give the instruction latency: https://llvm.org/doxygen/classllvm_1_1MCSubtargetInfo.html#ab5c894b844ecc7efdb7c932d351a320c

### Test program

```
#define __remote __attribute__((address_space(1)))

int vec_add(__remote int* a, __remote int* b, int n) {
  #pragma unroll 4
  for(int i=0; i<n; ++i) {
    a[i] += b[i];
  }

  return 0;
}
```

#### Attempt 1: Load Latency = 20; top-down scheduling algorithm; no register pressure;

```
.LBB0_4:                                # %for.body
                                        # =>This Inner Loop Header: Depth=1
  lw  a7, -8(a5)
  lw  t0, -8(a2)
  lw  t1, -4(a2)
  add a3, t0, a7
  sw  a3, -8(a2)
  lw  a3, -4(a5)
  lw  a7, 0(a2)
  add a3, t1, a3
  sw  a3, -4(a2)
  lw  a3, 0(a5)
  lw  t0, 4(a2)
  add a3, a7, a3
  sw  a3, 0(a2)
  lw  a3, 4(a5)
  addi  a4, a4, 4
  add a3, t0, a3
  sw  a3, 4(a2)
  addi  a5, a5, 16
  addi  a2, a2, 16
  bne a6, a4, .LBB0_4
```
