<!---
{
  "id": "a29aa0d7-e54c-4927-a4cc-0cd84f3b1032",
  "depends_on": ["6e50613d-60e2-40a8-91ae-609e0721f613"],
  "author": "Stephan Bökelmann",
  "first_used": "2025-06-05",
  "keywords": ["C Local Variables", "Inline Assembly", "Functions", "Header-Only Library", "Standard Library"]
}
--->

# From Inline Assembly to Libraries: Using Only Local Integers

> In this exercise you will learn how to work exclusively with local integer variables in C. You will also learn how to transition from inline assembly to reusable functions, libraries, and finally to using standardized library functions like `printf`. This mirrors real-world software development, where low-level hardware-specific code is gradually abstracted into portable, reusable components.

## Introduction

In modern programming, we rarely need to directly control the hardware to print text to the screen. Instead, we use platform-independent libraries that hide these details from us. However, these libraries were not always there. Historically, programmers had to write platform-specific code for basic operations such as outputting text.

In this exercise, we will start with inline assembly that directly invokes a system call. Then, we will refactor this code step-by-step into reusable functions, and eventually into a standard library function call (`printf`). Each stage will show you why abstraction layers exist, and how C supports both low-level and high-level development paradigms.

We will calculate a simple formula using local integers:

$result = (a + b) * c$

where `a`, `b`, and `c` are local integers.

> ⚠ This exercise assumes you are on x86\_64 Linux.

### 1.1) Further Readings and Other Sources

* [GCC Inline Assembly HOWTO](https://www.ibiblio.org/gferg/ldp/GCC-Inline-Assembly-HOWTO.html)
* [Linux System Calls Table](https://filippo.io/linux-syscall-table/)
* [Beej's Guide to C Programming](https://beej.us/guide/bgc/)
* [System V ABI PDF](https://gitlab.com/x86-psABIs/x86-64-ABI/-/raw/master/x86-64-ABI.pdf)

## Tasks

### Task 1: Calculate and Print Using Inline Assembly

Create the following file:

```c
// file: inline_asm.c

char msg[] = "The result is:\n";

int main() {
    int a = 5;
    int b = 7;
    int c = 3;
    int result = (a + b) * c;

    long ret;
    asm volatile (
        "movq $1, %%rax\n"
        "movq %1, %%rdi\n"
        "movq %2, %%rsi\n"
        "movq %3, %%rdx\n"
        "syscall\n"
        : "=a"(ret)
        : "r"((long)1), "r"(msg), "r"((long)14)
        : "rcx", "r11", "memory"
    );

    asm volatile (
        "movq $1, %%rax\n"
        "movq $1, %%rdi\n"
        "lea %[res], %%rsi\n"
        "movq $4, %%rdx\n"
        "syscall\n"
        :
        : [res] "m" (result)
        : "rax", "rdi", "rsi", "rdx", "rcx", "r11", "memory"
    );

    return 0;
}
```

#### a) Compile:

* `gcc -Wall -o inline_asm inline_asm.c`

#### b) Inspect assembly:

* `objdump -d inline_asm | less`

#### c) Reflect:

* How is the calculation done? The calculation is done in C
* How is the result printed using the syscall? Doesn't work

---

### Task 2: Move Inline Assembly into a Function

Now refactor your code into two files.

**print\_syscall.c:**

```c
long print_syscall(char* buf, long len) {
    long ret;
    asm volatile (
        "movq $1, %%rax\n"
        "movq %1, %%rdi\n"
        "movq %2, %%rsi\n"
        "movq %3, %%rdx\n"
        "syscall\n"
        : "=a"(ret)
        : "r"((long)1), "r"(buf), "r"(len)
        : "rcx", "r11", "memory"
    );
    return ret;
}
```

**function\_call.c:**

```c
// file: function_call.c

extern long print_syscall(char* buf, long len);

char msg[] = "The result is:\n";

int main() {
    int a = 5;
    int b = 7;
    int c = 3;
    int result = (a + b) * c;

    print_syscall(msg, 14);

    char num_msg[] = "XXX\n"; // placeholder for now
    print_syscall(num_msg, 4);

    return 0;
}
```

#### a) Compile:

* `gcc -Wall -c print_syscall.c`
* `gcc -Wall -o function_call function_call.c print_syscall.o`

#### b) Reflect:

* How does the external function encapsulate the syscall? *The external function has to be declared and could than be used in main.*
* Why is this more maintainable? It's better for maintenence, because you are able find errors faster. Also practical for using the same functions for different problems.

---

### Task 3: Move Function into a Header-Only Library

Now create a header-only version.

**print\_syscall.h:**

```c
#ifndef PRINT_SYSCALL_H
#define PRINT_SYSCALL_H

static inline long print_syscall(char* buf, long len) {
    long ret;
    asm volatile (
        "movq $1, %%rax\n"
        "movq %1, %%rdi\n"
        "movq %2, %%rsi\n"
        "movq %3, %%rdx\n"
        "syscall\n"
        : "=a"(ret)
        : "r"((long)1), "r"(buf), "r"(len)
        : "rcx", "r11", "memory"
    );
    return ret;
}

#endif
```

**header\_only.c:**

```c
#include "print_syscall.h"

char msg[] = "The result is:\n";

int main() {
    int a = 5;
    int b = 7;
    int c = 3;
    int result = (a + b) * c;

    print_syscall(msg, 14);

    char num_msg[] = "XXX\n"; // still placeholder
    print_syscall(num_msg, 4);

    return 0;
}
```

#### a) Compile:

* `gcc -Wall -o header_only header_only.c`

#### b) Reflect:

* Why is header-only convenient for small reusable functionality?
* How does `static inline` help avoid multiple definition errors?

---

### Task 4: Replace Everything with `printf`

Finally, switch to using the standard library.

```c
// file: final_version.c

#include <stdio.h>

int main() {
    int a = 5;
    int b = 7;
    int c = 3;
    int result = (a + b) * c;

    printf("The result is: %d\n", result);
    return 0;
}
```

#### a) Compile:

* `gcc -Wall -o final_version final_version.c`

#### b) Reflect:

* Why is `printf` easier to use?
* What hardware-specific details does `printf` hide from you?
* Why is this more portable?

---

## Questions

1. How does inline assembly interact with local variables?
2. Why is it useful to encapsulate low-level code into functions?
3. What is the benefit of a header-only implementation for small utilities?
4. Why does `printf` not require us to write any assembly or syscalls?
5. What role does the C standard library play in platform independence?
6. What problems would occur if everyone wrote direct syscall code for every platform?

## Advice

In professional software, we rarely write system calls directly. Instead, we rely on standard libraries like `libc`, which abstract away platform-specific details. Functions like `printf` allow the same source code to compile and run on many different operating systems and architectures. This portability is the reason why the C standard library exists and is universally adopted. Start small, learn the hardware, but always appreciate the value of well-designed libraries.

2. How do exercises improve long-term retention?
3. Can you think of a subject where learning through exercises might be less effective? Why?
4. What role does feedback play in learning through exercises?
5. How can self-designed exercises improve understanding?
6. Why is it beneficial to review past mistakes in exercises?
7. How does explaining a concept to someone else reinforce your own understanding?
8. What strategies can you use to stay motivated when practicing with exercises?
9. How can timed challenges contribute to learning efficiency?
10. How do exercises help bridge the gap between theory and practical application?

## Advice
Practice consistently and seek out diverse exercises that challenge different aspects of a topic. Combine exercises with reflection and feedback to maximize your learning efficiency. Don't hesitate to adapt exercises to fit your own needs and ensure that you're actively engaging with the material, rather than just going through the motions.

