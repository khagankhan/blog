---
title: 'Do parentheses matter for better performance?'
date:  2024-08-29
draft: false
# cover: 
#     image: images/post1/nested-parantheses.jpg
#     caption: 'Are we writing in Racket again?'
tags: ["llvm", "optimization"]
categories: ["compilers"]
---

# Missed optimizations in LLVM

The book "Computer Systems: A Programmer's Perspective" warns the readers: "When in doubt, put in parentheses!". Despite the authors saying this when talking about precedence issues, it would be applicable to some other cases. If you have ever written in a Lisp-like language you must love them!

But what would happen if we put some _undue_ parentheses in our code.

Let us consider the following C function (`sum.c`):
```c
unsigned int /* volatile */ src(unsigned int a, unsigned int b) {
    return a - b + b * 2;
}
```
This function, when compiled with certain flags to disable optimizations (`clang -S -emit-llvm sum.c -O0 -Xclang -disable-llvm-optzns -o sum.ll`), produces an unoptimized LLVM IR. (We need to disable optimizations when compiling this code; otherwise, the compiler might optimize away our function since it is not called. We will use Compiler Explorer to examine the optimized version of the code.)

## LLVM IR Analysis
The generated LLVM IR looks complex due to the lack of optimization:
```llvm
define i32 @src(i32 %0, i32 %1) #0 {
  %3 = alloca i32, align 4 ; a 
  %4 = alloca i32, align 4 ; b
  store i32 %0, i32* %3, align 4
  store i32 %1, i32* %4, align 4
  %5 = load i32, i32* %3, align 4 ; load a
  %6 = load i32, i32* %4, align 4 ; load b
  %7 = sub i32 %5, %6 ; a - b
  %8 = load i32, i32* %4, align 4 ; load b -> We load b again because this is SSA.
  %9 = mul i32 %8, 2 ; b * 2
  %10 = add i32 %7, %9 ; a - b + b * 2
  ret i32 %10
}
```
Normally, we would not see this level of detail unless optimizations were turned off, as the function would be simplified significantly. Using Compiler Explorer with optimization flags (with `opt 18.1.0` and `-O3` flag), we can observe a more [optimized version](https://godbolt.org/z/79Mh51vne) of the LLVM IR:
```llvm
define i32 @src(i32 %0, i32 %1) local_unnamed_addr #0 {
  %3 = sub i32 %0, %1 ; a - b
  %4 = shl i32 %1, 1 ; b * 2
  %5 = add i32 %3, %4 ; a - b + b * 2 (which is a + b)
  ret i32 %5 ; simply return a + b
}
```
However, even in this optimized state, the function can be further simplified (That is why I will continue referring to this version as the "non-fully-optimized" version). The transformation should ideally yield a function that simply adds `a` and `b`, as the complex expressions cancel out mathematically. In other words, LLVM [fails to optimize it further](https://godbolt.org/z/qcxMPr57c). Nonetheless, it is possible to optimize this code manually since it ultimately just returns the sum of its inputs: 
```llvm
define i32 @tgt(i32 %0, i32 %1) local_unnamed_addr #0 {
  %3 = add i32 %1, %0 ; a + b
  ret i32 %3 ; return a + b
}
```
[Alive2](https://dl.acm.org/doi/10.1145/3453483.3454030), a formal verification tool for LLVM optimizations, confirms that this simplified [transformation is correct](https://alive2.llvm.org/ce/z/9EYddW)! 
*We found a missed optimization in LLVM!!!*

## Change the order of the expressions
Let us make a small adjustment to our initial C function:
```c
unsigned int /* volatile */ src(unsigned int a, unsigned int b) {
    return a + (b * 2 - b);
}
```
Modifying the initial C function to add parentheses changes how LLVM processes and optimizes the code. Surprisingly, this slight change allows LLVM to achieve the optimal simplification without further manual intervention. We added parentheses to the right expression of the plus sign.
After compiling the modified C code with the same way, it will provide:
```llvm
define i32 @src(i32 %0, i32 %1) #0 {
  %3 = alloca i32, align 4 ; a
  %4 = alloca i32, align 4 ; b
  store i32 %0, i32* %3, align 4
  store i32 %1, i32* %4, align 4
  %5 = load i32, i32* %3, align 4 ; load a
  %6 = load i32, i32* %4, align 4 ; load b
  %7 = mul i32 %6, 2 ; b * 2
  %8 = load i32, i32* %4, align 4 ; load b
  %9 = sub i32 %7, %8 ; b * 2 - b (This apparently evaluates to b)
  %10 = add i32 %5, %9 ; a + b (b * 2 - b => b)
  ret i32 %10
}
```
We can observe the difference in the generated codes [here](https://www.diffchecker.com/q2koNRZU/).
The [LLVM-optimized version](https://godbolt.org/z/Y391xafGM) of the above code is:
```llvm
define i32 @src(i32 %0, i32 %1) local_unnamed_addr #0 {
  %3 = add i32 %1, %0 ; a + b
  ret i32 %3 ; return a + b
}
```
When compiled, LLVM now successfully optimizes the function to return just the sum of `a` and `b`, eliminating unnecessary multiplication and subtraction operations(That is why I will continue referring to this version as the "fully-optimized" version).

## Why does this happen?
To the user, two expressions might look identical, but to the compiler, they can be quite different. This difference is due to the distinct Abstract Syntax Trees (ASTs) that represent each expression, leading to different evaluations. The images below illustrate this difference:
<figure>
    <img src="/images/post1/non_optimized_ast.png" alt="Non-Fully-Optimized Abstract Syntax Tree" style="width: 100%;">
    <figcaption>(a) AST of the "without-parentheses" version of the code (Original)</figcaption>
</figure>

<figure>
    <img src="/images/post1/optimized_ast.png" alt="Fully-Optimized Abstract Syntax Tree" style="width: 100%;">
    <figcaption>(b) AST of the "with-parentheses" version of the code (Adjusted for optimization)</figcaption>
</figure>

<figcaption style="text-align: center;"><em>Figure 1: Abstract Syntax Trees for both cases</em></figcaption>

&nbsp;

Now, let us manually write out the expression in the order the AST dictates, ignoring any optimizations or compiler-generated intermediate representations (IR). For the non-fully-optimized version (the "without-parentheses" one), the compiler might produce something like this:
```llvm
define i32 @src(i32 %0, i32 %1) local_unnamed_addr #0 {
  %3 = sub i32 %0, %1 ; a - b
  %4 = shl i32 %1, 1 ; b * 2
  %5 = add i32 %3, %4 ; a - b + b * 2 (which is a + b)
  ret i32 %5 ; simply return a + b
}
```
For the fully-optimized version, where parentheses dictate the order of operations, we would get a different evaluation:
```llvm
define i32 @src(i32 %0, i32 %1) local_unnamed_addr #0 {
  %3 = shl i32 %1, 1 ; b * 2
  %4 = sub i32 %3, %1 ; b * 2 - b
  %5 = add i32 %0, %4 ; a + b * 2 - b
  ret i32 %5 ; return a + b
}
```
It seems that LLVM has an optimization pattern for the second representation, likely found in one of the files in its [InstCombine directory.](https://github.com/llvm/llvm-project/tree/ee0d70633872a30175cf29f81de7b2dbf771d708/llvm/lib/Transforms/InstCombine) Even without the line `%5 = add i32 %0, %4 ; a + b * 2 - b`, [LLVM would optimize](https://godbolt.org/z/PG8514x7T) `b * 2 - b` to `b`. 

However, it misses the optimization pattern for the first one. If we are interested, we could add an optimization pattern ourselves for this case, though it is uncertain when/if it would be merged. Yes, alas, this is exactly how it works! This is how we improve a compiler's refinement capabilities: by identifying missing optimizations and raising an issue. Then, either users or developers add optimization patterns based on their significance. As of this writing, there are 853 open and 380 closed issues tagged as "missed optimization" in the
[LLVM GitHub repository.](https://github.com/llvm/llvm-project/issues?q=is%3Aissue+is%3Aopen+label%3Amissed-optimization+) In my next post, I will delve into identifying the missed optimization pattern in LLVM and exploring ways to how to add it. 

## Conclusion
Well, we have discovered a missed optimization in LLVM that can achieve optimal form if the order of execution is altered by parentheses. However, we cannot definitively say that parentheses themselves are crucial, as the issue is not the absence of parentheses but rather a missed optimization within LLVM. Interestingly, if we write `b * 2` as `b + b` without using any parentheses, it will be optimized to the maximum extent:
```c
unsigned int /* volatile */ src(unsigned int a, unsigned int b) {
    return a - b + b + b;
}
```
It will provide the following LLVM IR:
```llvm
; This will do maximum optimization
define i32 @src(i32 %0, i32 %1) #0 {
  %3 = alloca i32, align 4 ; a
  %4 = alloca i32, align 4 ; b
  store i32 %0, i32* %3, align 4 
  store i32 %1, i32* %4, align 4
  %5 = load i32, i32* %3, align 4 ; load a
  %6 = load i32, i32* %4, align 4 ; load b
  %7 = sub i32 %5, %6 ; a - b
  %8 = load i32, i32* %4, align 4 ; load b
  %9 = add i32 %7, %8 ; a - b + b -> Evaluates to a
  %10 = load i32, i32* %4, align 4 ; load b
  %11 = add i32 %9, %10 ; ((a - b + b) -> Evaulated to a in %9 assignment) + b
  ret i32 %11 ; returns a + b
}
```
This will be [optimized to the maximum extent possible](https://godbolt.org/z/YKr3jPqTE):
```llvm
define i32 @src(i32 %0, i32 %1) local_unnamed_addr #0 {
  %3 = add i32 %1, %0
  ret i32 %3
}
```
However, if performance is critical and the existing optimization patterns are not applied automatically, users can change the order of evaluation by using parentheses to trigger these optimizations and achieve the best possible performance.