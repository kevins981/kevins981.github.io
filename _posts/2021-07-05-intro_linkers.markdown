---
title:  "Introduction to Linkers "
---
(Definitely should have taken that undergrad compiler elective... Welp better late than never.)

This post is my own notes and summary of [1]. I did not come up with any of these materials.

# Parts of a C file
Declaration vs Definition
- Definition: associates a name with an implementation of that name. The implementation could be either data or code
    - What the heck is a name, an implementation? What is data and code? 
    - Ok Googling "names in C" returns "Charlie, Chloe, Charlotte, Connor..."
    - I am guessing names refer to things like variable and function names. Ah, I think code refers to code that we write, such as functions. Data is, well, data, such as variables.
    - A variable definition makes the compiler reserve some space for that variable and potentially also filling in a value.
    - A function definition makes the compiler generate code for that function (not sure what this means. I guess generating the binary for that function?)
- Declaration: tells the compiler that the definition of this name exists somewhere else, probably in another C file. (A definition is also a declaration, where the "somewhere else" is here).

Variable definitions have two types:
- Global variables: exist for the whole program lifetime ("static extent") and accessible in different functions
- Local variables: only exists while a function is executing ("local extent") and only accessible in that function
But wait, there are some special cases:
- Static local variables are actually global variables. They exist for the program lifetime but are only visible inside a single function. Ex. we can use a static local variable to count 
how many times a function is called.
- Static global variables are global variables, but they can only be accessed by functions within the file where the variable is defined.

Ok easy. Here is a summary. But wait! Why is there no such thing as a local variable declaration? If I do `int x;` within a function, is this not a declaration? No! This is a definition of an 
uninitialized local variable. So I am mixing up **initialization vs declaration**. Doesn't the lack of `=` mean declaration? Nope.

For the table below, we see that a data without a `=` is definitely uninitialized, but says nothing about whether it is a declaration or definition. For variables, the rule seems simple: a piece 
of code is declaration iff `extern`. For functions, a declaration ends with `;`, and a definition ends with `}`. A variable is initialized iff there is a `=` sign.

{:refdef: style="text-align: center;"}
![](/assets/images/posts/intro_linker/c_parts.jpg){: width="850" } 
{: refdef}

Here is an example given in the tutorial. Makes sense.
```c
/* This is the definition of a uninitialized global variable */
int x_global_uninit;

/* This is the definition of a initialized global variable */
int x_global_init = 1;

/* This is the definition of a uninitialized global variable, albeit
 * one that can only be accessed by name in this C file */
static int y_global_uninit;

/* This is the definition of a initialized global variable, albeit
 * one that can only be accessed by name in this C file */
static int y_global_init = 2;

/* This is a declaration of a global variable that exists somewhere
 * else in the program */
extern int z_global;

/* This is a declaration of a function that exists somewhere else in
 * the program (you can add "extern" beforehand if you like, but it's
 * not needed) */
int fn_a(int x, int y);

/* This is a definition of a function, but because it is marked as
 * static, it can only be referred to by name in this C file alone */
static int fn_b(int x)
{
  return x+1;
}

/* This is a definition of a function. */
/* The function parameter counts as a local variable */
int fn_c(int x_local)
{
  /* This is the definition of an uninitialized local variable */
  int y_local_uninit;
  /* This is the definition of an initialized local variable */
  int y_local_init = 3;

  /* Code that refers to local and global variables and other
   * functions by name */
  x_global_uninit = fn_a(x_local, x_global_init);
  y_local_uninit = fn_a(x_local, y_local_init);
  y_local_uninit += fn_b(z_global);
  return (y_global_uninit + y_local_uninit);
}
```

# The C Compiler
The compiler converts C source files to an object file, which have the .o suffix. The object file contains two types of contents:
- code: definitions of functions. This is basically the machine instructions. 
- data: definitions of **global variables**

The compiler only allows the code to refer to a variable or a function if the compiler has seen its declaration. Thus, we can depict the previous program's object file using the below diagram.
Note that since `z_global` and `fn_a` are both declared without definitions, the compiler "leaves a blank".

{:refdef: style="text-align: center;"}
![](/assets/images/posts/intro_linker/obj_ex.jpg){: width="850" } 
{: refdef}

We can use the `nm` command to list the symbols of the object file. Here is what we will see:
```
Symbols from c_parts.o:

Name                  Value   Class        Type         Size     Line  Section

fn_a                |        |   U  |            NOTYPE|        |     |*UND*
z_global            |        |   U  |            NOTYPE|        |     |*UND*
fn_b                |00000000|   t  |              FUNC|00000009|     |.text
x_global_init       |00000000|   D  |            OBJECT|00000004|     |.data
y_global_uninit     |00000000|   b  |            OBJECT|00000004|     |.bss
x_global_uninit     |00000004|   C  |            OBJECT|00000004|     |*COM*
y_global_init       |00000004|   d  |            OBJECT|00000004|     |.data
fn_c                |00000009|   T  |              FUNC|00000055|     |.text
```
Under the Class column, U indicates that the symbol is undefined. t or T indicates code and whether the function is static (t) or not (T). d or D indicates initialized global variables and whether
it is static or not. b and B/C are uninitialized global variables and whether it is static or not.

# The Linker
Let's first give a companion C file to the previous one:
```c
/* Initialized global variable */
int z_global = 11;
/* Second global named y_global_init, but they are both static */
static int y_global_init = 2;
/* Declaration of another global variable */
extern int x_global_init;

int fn_a(int x, int y)
{
  return(x+y);
}

int main(int argc, char *argv[])
{
  const char *message = "Hello, world";

  return fn_a(11,12);
}
```

Now, we can fill in the blanks left previous in the first diagram. Note that now `fn_c` can point to `z_global` and `fn_a`.

{:refdef: style="text-align: center;"}
![](/assets/images/posts/intro_linker/obj_ex2.jpg){: width="850" } 
{: refdef}

## Duplicated Symbols
If the linker cannot find the definition of a symbol, it will give an error message. If there are *two* definitions for a symbol:
- in C++, this is not allow, as per the *one definition rule*.
- in C, this is allowed for uninitialized global variables. Their definitions are called *tentative definition*, and different source files can have tentative definitions for the same object.

# What the OS Does When Executing a Program
First, the OS must transfer the machine instructions from hard disk to memory. This chunk of memory is called *code* or *text segment*. Similarly, global variables need to be present in the memory
as well. *Initialized global variables' values are stored in the object file and the executable*. When the program starts, the OS copies these values into the memory in the *data segment*.

Uninitialized global variables are assumed to be 0 at the start, so there is no need to copy values into the memory. This chunk of memory that is initialized to 0 is the *bss segment*, where bss stands
for block starting symbol. This also means that we can save disk space by *compressing the uninitialized global variables*:

{:refdef: style="text-align: center;"}
![](/assets/images/posts/intro_linker/bss.png){: width="550" } 
{: refdef}

The linker **does not interact with local and dynamically allocated variables** because their lifetime only starts when the program is running, at the time which the linker already finished its job.
These two variables are stored in the stack and the heap, which grow in opposite directions.

{:refdef: style="text-align: center;"}
![](/assets/images/posts/intro_linker/os_map.png){: width="750" } 
{: refdef}

# The Linker and Libraries
## Static Libraries
Static libraries are libraries, aka. common code that is used often, that are linked at compile time. Static library binaries are a part of the program executable's code segment. 
This is disadvantageous since every program executable needs to have a copy of the same code (ex. all programs that uses printf will have the definition of printf in their executable). 
Shared libraries aims to address this.

We can create a static library as follows [2]:
```c
/* Filename: lib_mylib.c */
#include <stdio.h>
void fun(void)
{
  printf("fun() called from a static library");
}
```
```c
/* Filename: lib_mylib.h */
void fun(void);
```
```bash
 gcc -c lib_mylib.c -o lib_mylib.o  #compiler library source into object file
 ar rcs lib_mylib.a lib_mylib.o     #use the ar command to create static library file
```
```c
/* filename: driver.c  */
#include "lib_mylib.h"
void main() 
{
  fun();
}
```
```bash
gcc -c driver.c -o driver.o           #compiler driver code
gcc -o driver driver.o -L. -l_mylib   #link the driver code to static library
./driver 
fun() called from a static library
```

Note that static library files are usually prefixed with "lib" and have the `.a` extention.

During the linking process, the linker now have another place to look for unresolved symbols: the static libraries. Note that pulling in objects from static libraries can both resolve and **introduce
undefined references**. Thus, we need to make sure we include all the libraries we need.

## Shared Libraries
As discussed in the previous section, static libraries are disadvantageous in terms of **disk space** for popular libraries (ex. libc) that are used by many programs. Another disadvantage is that
once a program is statically linked, the **executable code is fixed**. That means if someone updates ex. the printf function, every program will need to recompile if printf was statically linked.
Shared libraries (ending with .`.so`) were introduced to resolve both of these problems. If the linker finds a symbol that is referencing a shared library, it **does not include that symbol's definition
in the final program executable**. When the program runs and before main is executed, **a smaller version of the linker, `ld.so` (dynamic linker), goes through the previously omitted symbols 
and pulls in code from shared libraries.** This means that to update printf, we just need to change `libc.so`. 

# Sources
[1] https://www.lurklurk.org/linkers/linkers.html

[2] https://www.geeksforgeeks.org/static-vs-dynamic-libraries/
