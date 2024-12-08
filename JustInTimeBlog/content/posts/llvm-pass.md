---
title: 'Be an LLVM contributor: Writing an optimization pattern for LLVM'
date:  2024-12-08
draft: false
# cover: 
#     image: images/post1/nested-parantheses.jpg
#     caption: 'Are we writing in Racket again?'
tags: ["llvm", "optimization"]
categories: ["compilers"]
---

## LLVM's Optimizations Are Truly Impressive

LLVM performs aggressive optimizations. One of my favorites is the Data Structure Elimination. For example, we want to 1) compute the sum of two integers (`get_sum` function), and 2) select one of the two characters based on a condition argument (`get_char` function):
```c++
#include <vector>
#include <string>

using namespace std;

int get_sum(int a, int b) {
    vector<int> vec1; // Create the first vector vec1
    vector<int> vec2; // Create the second vector vec2
    vec1.push_back(a); // Push the first argument to vec1
    vec2.push_back(b); // Push the second argument to vec2
    return vec1[0] + vec2[0]; // Return the sum of them
}
// Create a similar logic for string values
char get_char(char a, char b, bool cond) {
    string str1{a};
    string str2{b};
    return cond ? str1[0] : str2[0];
}
```
```asm
get_sum(int, int):
        lea     eax, [rdi + rsi]
        ret

get_char(char, char, bool):
        mov     eax, edi
        test    edx, edx
        cmove   eax, esi
        ret
```
For the `get_sum` function, the compiler takes advantage of the `lea` instruction for the calculation, and for `get_char`, it uses `cmove` which it usually does to avoid potential misprediction penalties. Despite the vector being created on the heap, LLVM is able to [optimize both of them away](https://gcc.godbolt.org/z/z74cThYr9), whereas GCC fails to do so for the vector but succeeds for string values. This is because small strings are stored on the stack (as explained in Raymond Chen's [excellent post](https://devblogs.microsoft.com/oldnewthing/20230803-00/?p=108532)), while the vector’s data is immediately allocated on the heap (For strings with a length of 15 characters (16 - 1 for the null character), the string is stored on the stack; beyond 15 characters, it is moved to the heap.)

## Missed optimizations

Despite the many existing optimizations, there are still several optimization patterns missing. These optimizations either do not trigger frequently enough to be incorporated into the LLVM production pipeline, or they have been genuinely overlooked. During the testing of LLVM's optimization techniques, I came across one such missing pattern, which I discussed in [my previous post](https://khagan.net/posts/missed-opt/). The pattern in question is: `X - Y + Y * 2 = X + Y`. In this post, we will be adding this pattern to LLVM.

## Write an optimization pattern
First, we need to install LLVM. After cloning the repository, I typically use the following command:
```zsh
cmake .. -GNinja -DLLVM_ENABLE_RTTI=ON -DLLVM_ENABLE_EH=ON -DBUILD_SHARED_LIBS=ON -DCMAKE_BUILD_TYPE=Release -DLLVM_ENABLE_ASSERTIONS=ON -DLLVM_ENABLE_PROJECTS="llvm;clang" /path/to/llvm -DBUILTINS_CMAKE_ARGS=-DCOMPILER_RT_ENABLE_IOS=OFF
```
Next, we need to find the appropriate place to add the pattern. For this optimization, the ideal location appears to be `llvm/lib/Transforms/InstCombine/InstCombineAddSub.cpp`. Before proceeding, we want to ensure that our optimization fires as frequently as possible. To achieve this, we generalize the pattern `X - Y + Y * 2 = X + Y` to `X - Y + Y * N = X + Y * (N - 1)`.

LLVM provides APIs for implementing optimization patterns. These APIs are so well-developed that they can almost be considered a DSL for adding such patterns. 

Since adding a missing optimization was part of a course assignment (see the Acknowledgement section), I will use the same technique to add and test this optimization. To begin, we add the following:
```c++
// I keep the name of the function used for the assignment
Instruction* cs6475_optimizer(Instruction *I, InstCombinerImpl &IC) { 
    // Create a constant integer and placeholder values for variables and subexpressions.
    ConstantInt *C = nullptr; // Represents a compile-time constant integer value.
    Value *X = nullptr;       // Represents a generic LLVM value (could be an instruction, constant, etc.).
    Value *Y = nullptr;       // Another generic LLVM value.
    Value *LHS = nullptr;     // Left-hand side of an addition operation.
    Value *RHS = nullptr;     // Right-hand side of an addition operation.
    // Match a pattern to see if I is an addition of two values.
    if (match(I, m_c_Add(m_Value(LHS), m_Value(RHS)))) {
    // Check if the left-hand side (LHS) matches the pattern: X - Y.
    if (match(LHS, m_Sub(m_Value(X), m_Value(Y))) &&
        // Check if the right-hand side (RHS) matches the pattern: Y * C (Y multiplied by a constant integer C).
        match(RHS, m_c_Mul(m_Value(Y), m_ConstantInt(C)))) {
            // Create a new multiplication instruction: Y * (C - 1).
            log_optzn("Khagan Karimov"); // Log the frequency
            Value *NewMul = IC.Builder.CreateMul(
                Y, IC.Builder.CreateSub(C, ConstantInt::get(C->getType(), 1)));
            // Create a new addition instruction: X + (Y * (C - 1)).
            Instruction *NewAdd = BinaryOperator::CreateAdd(X, NewMul);
            // Return the newly created instruction.
            return NewAdd;
        }
    }
}
```
- **`ConstantInt`**: Represents an immutable integer constant at compile-time.

- **`Value`**: Base class for all values in LLVM IR, including constants, instructions, and function arguments.

- **`match()`**: Matches an LLVM IR pattern against a value or instruction.
  - **`m_c_Add`**: Matches a commutative addition operation.
  - **`m_Sub`**: Matches a subtraction operation.
  - **`m_c_Mul`**: Matches a commutative multiplication operation.
  - **`m_Value`**: Matches any LLVM `Value`.
  - **`m_ConstantInt`**: Matches an LLVM `ConstantInt`.

- **`IC.Builder.CreateMul()`**: Creates a multiplication instruction in the current IR context using the provided values.

- **`IC.Builder.CreateSub()`**: Creates a subtraction instruction with the provided operands.

- **`ConstantInt::get()`**: Creates an LLVM `ConstantInt` of the specified type and value.

- **`BinaryOperator::CreateAdd()`**: Creates a binary addition instruction.

- **`Instruction`**: Represents a single operation in LLVM IR, such as addition, subtraction, or multiplication.

- **`log_optzn()`**: A custom logging function added to log optimization steps. In this case, it logs the message `"Khagan Karimov"` to track the applied optimization for debugging or analysis purposes.

When a new instruction is returned, LLVM automatically replaces the previous one with the new instruction.

I understand that this explanation might seem too abstract. For more details, please refer to the [LLVM documentation and the complete list of APIs](https://llvm.org/doxygen/).

## Testing Frequency of the New Optimization Pattern
To test the optimization, we need a large project to compile using the version of LLVM with the new optimization pattern we just added. LLVM itself is a great candidate for this, as it consists of approximately 10 million lines of code. We will compile LLVM using our modified version of LLVM (stay tuned for my upcoming post on "Reflections on Trusting Trust"). After compiling LLVM, we see the following output:
```txt
process 62594: CS6475 optimization by Khagan Karimov
process 62594: CS6475 optimization by Khagan Karimov
process 62594: CS6475 optimization by Khagan Karimov
process 62594: CS6475 optimization by Khagan Karimov
process 62644: CS6475 optimization by Khagan Karimov
process 62644: CS6475 optimization by Khagan Karimov
process 62623: CS6475 optimization by Khagan Karimov
process 62623: CS6475 optimization by Khagan Karimov
process 62623: CS6475 optimization by Khagan Karimov
process 62623: CS6475 optimization by Khagan Karimov
process 62623: CS6475 optimization by Khagan Karimov
process 62623: CS6475 optimization by Khagan Karimov
process 63423: CS6475 optimization by Khagan Karimov
process 63430: CS6475 optimization by Khagan Karimov
process 63709: CS6475 optimization by Khagan Karimov
process 64353: CS6475 optimization by Khagan Karimov
process 66501: CS6475 optimization by Khagan Karimov
process 69388: CS6475 optimization by Khagan Karimov
process 69388: CS6475 optimization by Khagan Karimov
```
This is our custom testing. LLVM developers have their own benchmark to measure how many times an optimization fires before deciding to work on it. For our example, you can see the results [here](https://github.com/dtcxzyw/llvm-opt-benchmark/issues/1299).

## Writing tests for our pattern
LLVM also has its unique way of testing. To understand it better, it is better to look at a test for our pattern:
```llvm
;; ALive2 verified: https://alive2.llvm.org/ce/z/ZogiKZ
define i32 @test1(i32 %A, i32 %B) {
  %C = sub i32 %A, %B     
  %D = mul i32 %B, 10        
  %E = add i32 %C, %D       
  ret i32 %E                 
; CHECK-LABEL: @test1(
; CHECK-NEXT:    [[F:%.*]] = mul i32 [[B:%.*]], 9
; CHECK-NEXT:    [[G:%.*]] = add i32 [[A:%.*]], [[F:%.*]]
; CHECK-NEXT:    ret i32 [[G:%.*]]
;
}
```
Initially, we write LLVM IR that performs the operation `A - B + B * 10` based on our pattern. The operations in the original code are as follows:

1. **`%C = sub i32 %A, %B`**:  
   This subtracts `B` from `A`, resulting in `C = A - B`.

2. **`%D = mul i32 %B, 10`**:  
   This multiplies `B` by 10, resulting in `D = B * 10`.

3. **`%E = add i32 %C, %D`**:  
   This adds the result of `C` and `D`, resulting in `E = (A - B) + (B * 10)`.

However, based on the optimization, the code should simplify to `A + 9 * B`. This transformation involves recognizing that: `(A - B) + (B * 10) = A + (B * 9)`

```llvm
define i32 @test1(i32 %A, i32 %B) {
  %F = mul i32 %B, 9       ; F = B * 9
  %G = add i32 %A, %F      ; G = A + (B * 9)
  ret i32 %G               ; Return G
}
```
In this optimized version, we perform the multiplication `B * 9` and then add it to `A`, resulting in `A + 9 * B`. We also provide `Alive2` verification of our test in the comments. 

This was a positive test, meaning that our optimization was successfully triggered. We also need to perform a negative test, where the optimization pattern should not fire because it may result in worse code generation. For example, if a `mul` operation is canonicalized to a `shl` (shift left), the optimization should not occur.

```llvm
;; When mul B, C where C can be shl no optimization happens
;; Alive2 verified: https://alive2.llvm.org/ce/z/w23md4
define i32 @test5(i32 %A, i32 %B) {
  %C = sub i32 %A, %B     
  %D = mul i32 %B, 8      
  %E = add i32 %C, %D       
  ret i32 %E                 
; CHECK-LABEL: @test5(
; CHECK-NOT:    [[F:%.*]] = mul i32 [[B:%.*]], 7
}
```
In fact, this suggests that there is still room to further generalize our optimization pattern, particularly for shifting instructions. However, this is beyond the scope of this blog post.

## Conclusion
In conclusion, although LLVM is a massive project, identifying and adding a missing optimization is a valuable way to contribute to LLVM.

## Acknowledgement
This blog post is based on my lecture notes from the CS-6475 Advanced Compilers course, taught by Prof. Dr. John Regehr at the University of Utah. The work was done as part of the second assignment for that course. Although I initially intended to submit a pull request (PR), another contributor made the PR while preserving my tests and the logic of the code, making me a co-contributor. You can find more details in the LLVM issue [here](https://github.com/llvm/llvm-project/issues/108451).
