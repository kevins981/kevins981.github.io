---
title:  "Introduction(s) to LLVM"
---
This post is my notes while going through LLVM tutorials [1], [2], and [3]. I do not own any of these materials.

# LLVM for Grad Students [1]
- One of LLVM's novelty is its intermediate representation (IR). LLVM operates on program IRs that are **human readable**. This might not sound like a big deal, but 
traditional compiler IRs are usually complex and difficult to understand.
- A compiler infrastructure is useful when ever we need to *do stuff with the program*. Ex. analyze how often the program does something, transform the program to work
better with your system.
- The below diagram shows LLVM's major components. This structure is also applicable to other modern compilers.
    - The *front end* turns your program into an IR.
    - The *passes* transforms an IR into another IR. The passes usually optimize the program. This is usually where the hacking happens.
    - The *backend* generates the machine code.
{:refdef: style="text-align: center;"}
![](/assets/images/posts/intro_llvm/llvm_arch.png){: width="550" } 
{: refdef}
- Another LLVM's novelty is that it uses the same IR format throughout the process, as opposed to other compilers that uses different formats for each pass.

## Writing a Pass
The author provided a template LLVM pass. The meat of this pass is:
```c
virtual bool runOnFunction(Function &F) {
  errs() << "I saw a function called " << F.getName() << "!\n";
  return false;
}
```
This is a *function pass*, where the above method is invoked with every function call in the program. The method returns false to indicate that it did not modify
`F`.

Building this will produce a shared library (`build/skeleton/libSkeletonPass.so` in this case). We will then need to load this library and run some real code.
To run the new pass, invoke clang and add the shared library. (This actually did not print for me. Moving on for now...)
```bash
$ clang -Xclang -load -Xclang build/skeleton/libSkeletonPass.* something.c
I saw a function called main!
```

## Understanding LLVM IR
The below figure shows the important components of an LLVM program.
- A *module* roughly represents a source file. More accurately it represents a *translation unit*.
- Functions are just functions.
- A *basic block* is a contiguous set of instructions.
- An instruction is a single code operation, ex. add, div, store.
- Function, BasicBlock, and Instruction all inherit from a base class called *Value*. A Value is any data that can be used in a computation.

{:refdef: style="text-align: center;"}
![](/assets/images/posts/intro_llvm/llvm_blks.png){: width="250" } 
{: refdef}

## IR Instruction
This example instruction `%5 = add i32 %4, 2` adds two 32 bit integers. `%` means register, and `2` is the number 2. See, human readable.
Note that LLVM has infinitely many registers. In the compiler, this instruction is represented as a `Instruction` object. It has an opcode (addition), a type, and a list
of operands which are pointers to other `Value` objects. In this case, the list of operands points to a `Constant` object 2 and **another instruction representing the 
register &4**. Hmm, why is the register an instruction? This is because LLVM IR uses **static single assignment (SSA)** form, meaning that registers and instructions are actually 
the same. See question 2.

To generate the LLVM IR, use 
```bash
$ clang -emit-llvm -S -o - something.c
```

## Example IR
If we change our previous `runOnFunction` IR to the following, we can (or at least should. I still can't get this running) see more information about the functions:
```c
virtual bool runOnFunction(Function &F) {
  errs() << "In a function called " << F.getName() << "!\n";

  errs() << "Function body:\n";
  //F.dump(); Out-of-Date: no more dump support in modern llvm unless you enable it at compile time.
  F.print(llvm::errs());

  for (auto &B : F) {
    errs() << "Basic block:\n";
    B.print(llvm::errs(), true);

    for (auto &I : B) {
      errs() << "Instruction: \n";
      I.print(llvm::errs(), true);
      errs() << "\n";
    }
  }

  return false;
}
```

## Another (and perhaps more interesting) Example IR
The below example pass look for patterns in the program and change the code when we find them. Let us say we want to replace the first binary operator (ex. +, -)
in every function with a multiplication. Very useful indeed.

The meat of the pass is here:
```c
for (auto& B : F) {
  for (auto& I : B) {
    // Go through each instruction in the function. Check if the instruction is of the type BinaryOperator.
    if (auto* op = dyn_cast<BinaryOperator>(&I)) {
      // Insert at the point where the instruction `op` appears.
      IRBuilder<> builder(op);

      // Make a multiply with the same operands as `op`.
      Value* lhs = op->getOperand(0);
      Value* rhs = op->getOperand(1);
      Value* mul = builder.CreateMul(lhs, rhs);

      // Everywhere the old instruction was used as an operand, use our
      // new multiply instruction instead.
      for (auto& U : op->uses()) {
        User* user = U.getUser();  // A User is anything with operands.
        user->setOperand(U.getOperandNo(), mul);
      }

      // We modified the code.
      return true;
    }
  }
}
```
Some notes:
- The documentation of `dyn_cast<>()` is here: https://llvm.org/docs/ProgrammersManual.html#the-isa-cast-and-dyn-cast-templates. 
> The `dyn_cast<>` operator is a "checking cast" operation. It checks to see if the operand is of the specified type, 
and if so, returns a pointer to it (this operator does not work with references).
- A somewhat unintuitive part of the pass is that we not only need to replace that particular instruction, but also **find all places
the old instruction is used and replace them with the new instruction.** Recall that an Instruction is a Value, meaning that other Instructions
might use it as an operand.
- The author notes that we should also remove the old instruction, but that is not included in the pass.

Now if we compile the below program:
```c
#include <stdio.h>
int main(int argc, const char** argv) {
    int num;
    scanf("%i", &num);  // read input from stdin
    printf("%i\n", num + 2);
    return 0;
}
```
We see our pass taking an effect vs. an ordinary compiler:
```bash
$ cc example.c
$ ./a.out
10
12
$ clang -Xclang -load -Xclang build/skeleton/libSkeletonPass.so example.c
$ ./a.out
10
20
```

## Linking with Runtime Library
In the previous example, all of our pass logic are written in the LLVM cpp file. We could also write our pass in C and link it with the application.
For example, we could have a separate C file `rtlib.c`:
```c
#include <stdio.h>
void logop(int i) {
  printf("computed: %i\n", i);
}
```
Then in our LLVM pass code:
```cpp
// Get the function to call from our runtime library.
LLVMContext& Ctx = F.getContext();
FunctionCallee logFunc = F.getParent()->getOrInsertFunction(
  "logop", Type::getVoidTy(Ctx), Type::getInt32Ty(Ctx)   // add a declaration of the function logop. The definition is in rtlib.c
);

for (auto& B : F) {
  for (auto& I : B) {
    if (auto* op = dyn_cast<BinaryOperator>(&I)) {
      // Insert *after* `op`.
      IRBuilder<> builder(op);
      builder.SetInsertPoint(&B, ++builder.GetInsertPoint());

      // Insert a call to our function.
      Value* args[] = {op};
      builder.CreateCall(logFunc, args);

      return true;
    }
  }
}
```
To run the application, we need to compile the runtime library `rtlib.c` and link it:
```bash
$ cc -c rtlib.c
$ clang -Xclang -load -Xclang build/skeleton/libSkeletonPass.so -c example.c
$ cc example.o rtlib.o
$ ./a.out
12
computed: 14
14
```





# LLVM Design Decisions [2]

# My First Language Frontend with LLVM Tutorial [3]

# Questions
1. "A Value is any data that can be used in a computation". How do I do computation with a function or an instruction?
2. What exactly is static single assignment?

# Sources
[1] https://www.cs.cornell.edu/~asampson/blog/llvm.html

[2] http://www.aosabook.org/en/llvm.html

[3] https://llvm.org/docs/tutorial/MyFirstLanguageFrontend/index.html

[4] https://llvm.org/docs/WritingAnLLVMPass.html
