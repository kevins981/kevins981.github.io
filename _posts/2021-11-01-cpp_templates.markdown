---
title:  "Cpp Template"
---

This post is my notes while going through Internet sources (see Sources section). I do not own any of 
the materials.

# Function Templates
Pretty simple idea: function templates are functions that can operate on *generic types*.
The function caller specifies the type via the *template parameter*.

In the below example, myType is the template parameter. Pretty easy right?

```cpp
template <class myType>
myType GetMax (myType a, myType b) {
    return (a>b?a:b);
}

int main() {
    int x,y;
    GetMax <int> (x,y);
}
```

When the compiler encounters the call to GetMax, it generates separate functions for each template
parameter.

We could also not specify the template parameter and let the compiler figure out the type.


# Class Templates
Same idea. We want to use a class (ex. Stack) for various data types. Here is an example class template
to store a pair of objects: 

```cpp
// class templates
template <class T>
class mypair {
    T a, b;
  public:
    mypair (T first, T second) {
        a=first; b=second;
    }
    T getmax ();
};

// bunch of Ts here
template <class T>
T mypair<T>::getmax () {
    T retval;
    retval = a>b? a : b;
    return retval;
}

int main () {
    mypair <int> myobject (100, 75);
    myobject.getmax();
    return 0;
}
```

Note the syntax for defining a function member outside of the class template, mypair. That's a lot of Ts.

```cpp
template <class T>
T mypair<T>::getmax () {
```

Note that this is a function template. 
The first T is the template paramter. I think this is just boiler plate code that must be there.
The second T is the return type. The third T means that the function template's template parameter is the same
as the class template parameter. 

(Seems like "x template's template parameter == x template parameter")

## Using typedef
A good programming practice is to use typedef while instantiating template classes. A couple of advantages:
- Less letters to type and more clean, especially for templates of templates. 
Ex. `typedef vector<int, allocator<int> > INTVECTOR ;` Note that the std::vector is a template class with the 
below definition. Note that the Allocation template parameter is a template itself.

    ```cpp
template<
    class T,
    class Allocator = std::allocator<T>
> class vector;
```
- Easier to modify if the template definition changes. Ex. std::vector now does not require the allocation 
template parameter anymore. Only need to modify the typedef.

## Template Specialization
What if we want to specify a special implementation when a specific template parameter is passed in? 

In the below example, when we instantiate a mycontainer object with template parameter \<char\>, cpp will use
the specialized implementation instead. In this case, we do not support increase() for char type, but provide
uppercase() instead.

```cpp
// class template:
template <class T>
class mycontainer {
    T element;
  public:
    mycontainer (T arg) {element=arg;}
    T increase () {return ++element;}
};

// class template specialization:
template <>
class mycontainer <char> {
    char element;
  public:
    mycontainer (char arg) {element=arg;}
    char uppercase ()
    {
      if ((element>='a')&&(element<='z'))
      element+='A'-'a';
      return element;
    }
};
```

## Non-type Parameters
We can also pass in regular typed parameters (ex. int) to a template. Note the int N template parameter:

```cpp
template <class T, int N>
class mysequence {
    T memblock [N];
  public:
    void setmember (int x, T value);
    T getmember (int x);
};

template <class T, int N>
void mysequence<T,N>::setmember (int x, T value) {
  memblock[x]=value;
}

template <class T, int N>
T mysequence<T,N>::getmember (int x) {
  return memblock[x];
}

int main () {
  mysequence <int,5> myints;
  mysequence <double,5> myfloats;
  // note here we do not need to specify the type to call the member function
  myints.setmember (0,100);
  myfloats.setmember (3,3.1416);
}
```

# Template Instantiations


{:refdef: style="text-align: center;"}
![](/assets/images/posts/bigo/kev.png){: width="450" }
{: refdef}

# Further Readings
- Cpp std::allocator
- namspace

# Sources
[1] http://users.cis.fiu.edu/~weiss/Deltoid/vcstl/templates

[2] https://www.cplusplus.com/doc/oldtutorial/templates/


