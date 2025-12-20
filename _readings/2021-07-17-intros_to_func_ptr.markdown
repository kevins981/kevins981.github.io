---
title:  "Introduction to Function Pointers"
---

This post is my notes while going through LLVM tutorials [1]. I do not own any of these materials.

## Creating and Assigning Function Pointers
Function pointers are pointers to functions. To create a non-const function pointer:
```c
// fcnPtr is a pointer to a function that takes no arguments and returns an integer
int (*fcnPtr)();
```
Here, fcnPtr can point to any function that takes no arguments and returns an int. Note that the parenthesis around `*fcnPtr` is required.

To create a const function pointer:
```c
int (*const fcnPtr)();
```

Similar to normal pointers, function pointers can be initialized or assigned after the definition:
```c
int foo() {
    return 5;
}
 
int goo(){
    return 6;
}
 
int main() {
    int (*fcnPtr)(){ &foo }; // fcnPtr points to function foo
    fcnPtr = &goo; // fcnPtr now points to function goo
    return 0;
}
```
The type of the function pointer and the assigned function must match:
```c
// function prototypes
int foo();
double goo();
int hoo(int x);
 
// function pointer assignments
int (*fcnPtr1)(){ &foo }; // okay
int (*fcnPtr2)(){ &goo }; // not okay. goo returns double
double (*fcnPtr4)(){ &goo }; // okay
fcnPtr1 = &hoo; // not okay. hoo takes one int input
int (*fcnPtr3)(int){ &hoo }; // okay
```

## Calling Functions Using Function Pointers
Two ways to do this: explicit and implicit dereference.

Explicit dererefence:
```c
int foo(int x) {
    return x;
}
 
int main() {
    int (*fcnPtr)(int){ &foo };
    (*fcnPtr)(5); // call foo(5) explicitly
    return 0;
}
```
Implicit dererefence:
```c
int main() {
    int (*fcnPtr)(int){ &foo }; 
    fcnPtr(5); // call foo(5) implicitly
    return 0;
}
```

## Passing Function Pointers as Arguments
Common use case. Functions used as arguments to other functions are sometimes called **callback functionss**.
For example, I can create a function to perform sort but let the caller provide the specifc comparison function as a function pointer:
```c
// the user-defined comparison is the third parameter
void selectionSort(int *array, int size, bool (*comparisonFcn)(int, int)) {
    for (int startIndex{ 0 }; startIndex < (size - 1); ++startIndex) {
        int bestIndex{ startIndex };
        for (int currentIndex{ startIndex + 1 }; currentIndex < size; ++currentIndex) {
        // Use the function pointer implicitly
            if (comparisonFcn(array[bestIndex], array[currentIndex])) { 
                bestIndex = currentIndex;
            }
        }
        std::swap(array[startIndex], array[bestIndex]);
    }
}
 
bool ascending(int x, int y) {
    return x > y; 
}
 
bool descending(int x, int y) {
    return x < y;
}
 
int main() {
    int array[9]{ 3, 7, 9, 5, 6, 1, 8, 2, 4 };
    selectionSort(array, 9, descending); // pass in different comparison functions
    selectionSort(array, 9, ascending);
    return 0;
}
```
Note that as a function parameter, function pointer type is equivalent to function type. Thus, the below two lines are the equivalent:
```c
void selectionSort(int *array, int size, bool (*comparisonFcn)(int, int))
void selectionSort(int *array, int size, bool comparisonFcn(int, int))
```
We can provide default functions:
```c
void selectionSort(int *array, int size, bool (*comparisonFcn)(int, int) = ascending);
```
## Using Type Aliases to Make Function Pointers Prettier
```c
using ValidateFunction = bool(*)(int, int);
bool validate(int x, int y, bool (*fcnPtr)(int, int)); // ugly
bool validate(int x, int y, ValidateFunction pfcn) // clean
```
## Type Inference
We can use the `auto` keyword to infer the type of a function pointer:
```c
int foo(int x) {
	return x;
}
 
int main() {
	auto fcnPtr{ &foo };
	std::cout << fcnPtr(5) << '\n';
	return 0;
}
```

# Sources
[1] https://www.learncpp.com/cpp-tutorial/function-pointers/

