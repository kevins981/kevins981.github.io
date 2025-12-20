---
title:  "C++ Smart Pointer"
---

This post is my notes while going through [1]. see Sources section for all sources used in this poist. I do not own any of 
the materials.

# Background
- Resource acquisition is initialization (RAII): a programming technique where the resource 
required to use an object (holdign the resource is a class invariant) is allocated during object initialization and released during object destruction. 
    - "Obtain the resouce in class constructor and release in destructor."
- Member Initialization List: used in constructors to initialize class variables (there are practical reasons why this would be useful. Skipping for now to focus on smart pointers). Ex.
```c++
class ResourceGuard {
  private:
    const std::string resource;
  public:
    // resource(res) is the initialization list
    ResourceGuard(const std::string& res):resource(res) {
      std::cout << "Acquire the " << resource << "." <<  std::endl;
    }
    // release resource in destructor
    ~ResourceGuard() {
      std::cout << "Release the "<< resource << "." << std::endl;
    }
};
```
- Reference counting: method to automatically track the lifetime of an object by counting
how many pointers point to the object. 

# Smart Pointers
{:refdef: style="text-align: center;"}
![](/assets/images/posts/cpp_smart_ptr/ptr_overview.png){: width="450" }
{: refdef}
- Top two are exclusive ownership, last two are shared.
- `auto_ptr` issue: cannot be copied, only moved. Should always use `unique_ptr`

## unique_ptr
- Two components: a *stored pointer*, which is the ptr to the object that it manages, and a *stored
deleter*, which is a function used to delete the managed object [2].
- Cannot be copied, only be moved. (this seems to be the same as auto_ptr. So why was auto_ptr
deprecated and what are the differences?)
- "has NO space and time overhead compared to raw pointer." How does this work? 
- This means that at a user level, there is no reason to use raw pointers. 
- Can customize the deleter to ex. close a file. How does this work?
- Can be specialized for arrays. Why do we need this? How does this work?

Example: what is the output of the following? 
```c++
struct MyInt {
  MyInt(int i) : i_(i) {}
  ~MyInt() {
    std::cout << "Good bye from " << i_ << std::endl;
  }
  int i_;
};

int main() {

  std::cout << std::endl;

  MyInt* myInt15 = new MyInt(15);
  std::unique_ptr<MyInt> uniquePtr1(myInt15);
  std::unique_ptr<MyInt> uniquePtr15(myInt15);

  std::cout << "uniquePtr1.get(): " << uniquePtr1.get() << std::endl;

  std::unique_ptr<MyInt> uniquePtr2{std::move(uniquePtr1)};
  std::cout << "uniquePtr1.get(): " << uniquePtr1.get() << std::endl;
  std::cout << "uniquePtr2.get(): " << uniquePtr2.get() << std::endl;

  std::cout << std::endl;


  {
    std::unique_ptr<MyInt> localPtr{new MyInt(2003)};
  }

  std::cout << std::endl;

  // destroy what uniquePtr2 was holding and take ownership of new ptr
  // think of new as malloc: returns pointer to heap
  uniquePtr2.reset(new MyInt(2011));
  // uniquePtr2 not responsible for 2011 anymore. Returns value of stored ptr
  MyInt* myInt= uniquePtr2.release();
  delete myInt;

  std::cout << std::endl;

}
```
```
// let the value of myInt15 be XXX
uniquePtr1.get(): XXX
uniquePtr1.get(): 0
uniquePtr2.get(): XXX

Good bye from 2003
Good bye from 15
Good bye from 2011
```
- The above output is correct, except that I missed the following outputs at the end:
```
Good bye from 0
free(): double free detected in tcache 2
Aborted (core dumped)
```
    - The problem is that we have two unique_ptrs on myInt15. This means that we tried to destroy
    myInt15 two times. This results in undefined behavior [3].
- To pass unique pointer into a function, one should use std::move at the call site. Otherwise,
undefined behavior.

## Shared Pointer
- Cpp's answer to garbage collection
- The control interface is thread safe, but the underlying resource is not. This means that
one should avoid bypassing the shared_ptr interface in multi-threading. 

Ex. what is the output?
```c++
class MyInt {
public:
  MyInt(int v) : val(v) {
   std::cout << "  Hello: " << val << std::endl;
  }
  ~MyInt() {
   std::cout << "  Good Bye: " << val << std::endl;
  }
private:
  int val;
};

int main() {

  std::cout << std::endl;

  std::shared_ptr<MyInt> sharPtr(new MyInt(1998));
  
  std::cout << "sharedPtr.use_count(): " << sharPtr.use_count() << std::endl;
  {
    // copy the shared ptr. now two owners
    std::shared_ptr<MyInt> locSharPtr(sharPtr);
    std::cout << "locSharPtr.use_count(): " << locSharPtr.use_count() << std::endl;
  }
  std::cout << "sharPtr.use_count(): "<<  sharPtr.use_count() << std::endl;

  std::shared_ptr<MyInt> globSharPtr = sharPtr; // copy assignment
  std::cout << "sharPtr.use_count(): "<<  sharPtr.use_count() << std::endl;
  
  globSharPtr.reset(); //This only release one owner
  std::cout << "sharPtr.use_count(): "<<  sharPtr.use_count() << std::endl;

  // make sharPtr take care of a different resource.
  sharPtr = std::shared_ptr<MyInt>(new MyInt(2011));

  std::cout << std::endl;
}
```
Solution:
```
Hello: 1998
sharedPtr.use_count(): 1
locSharedPtr.use_count(): 2
sharedPtr.use_count(): 1
sharedPtr.use_count(): 2
sharedPtr.use_count(): 1
Hello: 2011
Good Bye: 1998
Good Bye: 2011
```

Custom deleter example:
```c++
template <typename T>
class Deleter {
public:
  // Key!!! this is called by the shared pointer to release the resource
  void operator()(T *ptr) {
    ++Deleter::count;
    delete ptr;
  }
  void getInfo() {
    std::string typeId{typeid(T).name()};
    size_t sz= Deleter::count * sizeof(T);
    std::cout << "Deleted " << Deleter::count << " objects of type: " << typeId << std::endl;
  }
private:
  static int count;  // note that this is static: shared by all instances of Deleter
};

template <typename T>
int Deleter<T>::count = 0;
typedef Deleter<int> IntDeleter;

int main() {
  int* myint = new int(1998);
  std::shared_ptr<int> sharedPtr1(myint, IntDeleter());
  std::shared_ptr<int> sharedPtr2 = sharedPtr1; // ref count ++ 
  std::cout << "sharPtr1.use_count(): "<<  sharedPtr1.use_count() << std::endl;

  auto intDeleter= std::get_deleter<IntDeleter>(sharedPtr1);
  intDeleter->getInfo();
  sharedPtr2.reset(); // this does not delete stuff since the underlying resource is not destroyed
  intDeleter->getInfo();
  std::cout << "sharPtr1.use_count(): "<<  sharedPtr1.use_count() << std::endl;
  sharedPtr1.reset(); // this should delete stuff since ref count is now 0
  intDeleter->getInfo();
}
```
Output: 
```
sharPtr1.use_count(): 2
Deleted 0 objects of type: i
sharPtr1.use_count(): 1
Deleted 0 objects of type: i
Deleted 1 objects of type: i
```

## Weak Pointer
- key motivation is eliminating cyclic references in shared pointers. Example [4]: 

```c++
struct B;
struct A {
  // each struct contains a shared ptr as a member
  std::shared_ptr<B> b;  
  ~A() { std::cout << "~A()\n"; }
};

struct B {
  std::shared_ptr<A> a;
  ~B() { std::cout << "~B()\n"; }  
};

void useAnB() {
  // make_shared passes the provided arguments (empty here) to type T's constructor (A).
  // since we are using default constructor, empty arguments.
  auto a_shared = std::make_shared<A>();
  auto b_shared = std::make_shared<B>();
  // we have 4 shared_ptr here: struct A's b, struct B's a, a_shared, and b_shared
  a_shared->b = b_shared;
  b_shared->a = a_shared;
  // a_shared and b_shared out of scope. But no destructors are called!
  // Although a_shared and b_shared are destroyed, the structs themselves are not because 
  // two shared pointers still exist in the program!
  // btw, make_shared "uses ::new to allocate storage for the object", so memory leak
}

int main() {
   useAnB();
   std::cout << "Finished using A and B\n";
}

// the solution is to make either one of A->b or B->a a weak pointer to break the cycle.
```

- On a high level, weak_ptr models temporary ownership. I can use it to access the underlying
object only when the object still exists, and the object can be deleted by someone else at anytime.
- A weak_ptr holds reference to an object managed by a shared_ptr
- weak_ptr is weak in the sense that it does NOT own resources, does not provide access to the resource, and does not increase the reference count

{:refdef: style="text-align: center;"}
![](/assets/images/posts/cpp_smart_ptr/weak_ptr.png){: width="450" }
{: refdef}

```c++
  auto sharedPtr = std::make_shared<int>(2011);
  // create weak pointer from shared pointer
  std::weak_ptr<int> weakPtr(sharedPtr);
  
  // this use_count returns the number of shared_ptrs owning the obj. 
  // it really has nothing to do with the weak pointer itself.
  std::cout << "weakPtr.use_count(): " << weakPtr.use_count() << std::endl;
  std::cout << "sharedPtr.use_count(): " << sharedPtr.use_count() << std::endl;
  // expired() checks whether the referenced obj is already deleted
  std::cout << "weakPtr.expired(): " << weakPtr.expired() << std::endl;

  // create a temporary shared ptr to access object
  if( std::shared_ptr<int> sharedPtr1 = weakPtr.lock() ) {
    std::cout << "*sharedPtr: " << *sharedPtr << std::endl;
    // the temporary shared pointer still counts
    std::cout << "sharedPtr.use_count(): " << sharedPtr.use_count() << std::endl;
  } else {
    std::cout << "Don't get the resource!" << std::endl;
  }

  // release the referece. Note that this does not change the object and its shared pointers.
  weakPtr.reset(); 
  if( std::shared_ptr<int> sharedPtr1 = weakPtr.lock() ) {
    std::cout << "*sharedPtr: " << *sharedPtr << std::endl;
  } else {
    std::cout << "Don't get the resource!" << std::endl;
  }
```
Output:
```
  weakPtr.use_count(): 1
  sharedPtr.use_count(): 1
  weakPtr.expired(): 0
  *sharedPtr: 2011
  sharedPtr.use_count(): 2
  Don't get the resource!
```

- Another interesting case if doubly linked list. If all prev and next pointers are 
shared, this creates cyclic reference. The solution is to make one of prev and next weak.
Note that in practice it is often not nessesary to use shared_ptr for linked list, since
there is usually only one owner. 

# Sources
[1] https://www.youtube.com/watch?v=sQCSX7vmmKY&list=PLHTh1InhhwT5o3GwbFYy3sR7HDNRA353e&index=10

[2] https://www.cplusplus.com/reference/memory/unique_ptr/
 
[3] https://stackoverflow.com/questions/49334967/c-multiple-unique-pointers-from-same-raw-pointer

[4] https://stackoverflow.com/questions/27085782/how-to-break-shared-ptr-cyclic-reference-using-weak-ptr
