---
title: Vector - Resource Management Allocation
date: 2016-02-27T12:00:31-0800
author: Loki Astari, (C)2016
comments: true
categories:
- C++
- Vector
- Resource-Management
- C++-By-Example
- Coding
series:
- Vector
tags:
- Vector
subtitle: C++ By Example
description: C++ By Example. The Vector - Part 1. Many new developers of C++ attempt
  to build a `Vector'- like container as a learning process. Getting a simple version
  of this working for POD types (like int) is not that complicated. The next step
  in getting this working for arbitrary data types takes a significant leap forward
  in thinking in C++, especially when you start looking at efficiency and exception
  safety. This set of five articles looks at building an efficient `Vector` implementation.
  I show some common mistakes and explain why and how to resolve the problems.
draft: false
cover:
  image: /images/post/post-5.png
  hidden: false
  caption: Photo by Oscar Nilsson
---

Many new developers of C++ attempt to build a `Vector` like container as a learning process. Getting a simple version of this working for POD types (like int) is not that complicated. The next step in getting this working for arbitrary data types takes a significant leap forward in thinking in C++, especially when you start looking at efficiency and exception safety. This set of five articles looks at building an efficient `Vector` implementation. I show some of the common mistakes and explain why and how to resolve the problems:

Note: This is not meant to replace `std::vector<>`; this is intended as a teaching process.

# Rule of Zero

You will notice that half the attempts below [Sources](#Sources) are Vector implementations, and the other half are for Matrix implementations. I mention both to emphasize the [Separation of concerns](https://en.wikipedia.org/wiki/Separation_of_concerns). An object should be responsible for either business logic or resource management (not both). Many Matrix implementations try to mix resource management (memory management) with the business logic of how matrices interact. So if you want to write a matrix class, you should delegate resource management to a separate class (In a first pass, `std::vector<int>` would be a good choice).

In C++, the compiler generates a couple of methods for free.

* Destructor
* Copy Constructor
* Copy Assignment Operator
* Move Constructor
* Move Assignment Operator

These methods usually work perfectly well; **unless** your class contains a pointer (or a pointer like resource object). But if your class is doing business logic, it should not contain a pointer. So classes that handle business logic, therefore, should not be defining any of these compiler-generated methods (just let the compiler-generated ones work for you). Occasionally, you want to delete them to prevent copying or movement, but it is very unusual for these to need specialized implementations.

Conversely, resource management classes usually contain a pointer (or pointer-like resource object). These classes should define all the above methods to handle the resource correctly. This is where ownership semantics of the resource are defined. The resource owner is responsible for destroying the resource when its lifespan is over (in terms of pointers, the owner is responsible for calling `delete` on the pointer, usually in the destructor). If you are not the owner of a resource, you should not have access to the resource object directly, as the owner may destroy it without other objects knowing.

# [Rule of three](https://stackoverflow.com/q/4172722/14065)

The rule of three comes from C++03, where we only had copy semantics.

## Version-1: Simple Resource Management
When creating a class to manage resources, the first version created by beginners usually looks like this:

#### Rule of three first pass
```cpp
template<typename T>
class Vector
{
    std::size_t     size;
    T*              buffer;
    Vector(int size = 100)
        : size(size)
        , buffer(new T[size])   // Allocate the resource
    {}
    ~Vector()
    {
        delete [] buffer;       // Clean up the resource
    }
};
```
The trouble here is that this version has a fundamental flaw because of the way the [compiler generated](https://stackoverflow.com/a/4044360/14065) copy constructor and copy assignment operator work with pointers (commonly referred to as the [shallow copy problem](https://stackoverflow.com/q/2344664/14065)).

#### Shallow copy problem.
```cpp
int main()
{
    Vector<int>   x;
    Vector<int>   y(x);     // Compiler generate copy constructor does
                            // an element-wise shallow copy of each element.
                            // This means both `x` and `y` have a buffer
                            // member that points at the same area in memory.
                            //
                            // When the objects go out of scope, both will
                            // try and call delete on the memory resulting
                            // in a double delete of the memory.

    Vector<int>   z;        // Same problem after an assignment.
    z=x;
}
```

## Version-2: Rule of Three
The rule of three simply stated is: If you define any of the methods Destructor/Copy Constructor/Copy Assignment Operator, then you should define all three. When done correctly, this resolves the shallow copy problem. `Vector` defines the destructor, so we also need to define the copy constructor and copy assignment operator.

I often see this as an initial attempt at defining the rule of three for vectors.

#### Rule of three second pass
```cpp
template<typename T>
class Vector
{
    std::size_t     size;
    T*              buffer;
    Vector(int size = 100)
        : size(size)
        , buffer(new T[size])
    {}
    ~Vector()
    {
        delete [] buffer;
    }
    Vector(Vector const& copy)
        : size(copy.size)
        , buffer(new T[size])
    {
        // Copy constructor is simple.
        // We create a new resource area of the required size.
        // Then we copy the data from the old buffer to the new buffer.
        std::copy(copy.buffer, copy.buffer + copy.size, buffer);
    }
    Vector& operator=(Vector const& copy)
    {
        // Copy Object
        // This is relatively easy. But I want to cover this in detail in a subsequent post.
        return *this;
    }
};
```

## Version-3: Lazy Construction of elements.

The problem with the previous version is that it immediately forces the initialization of all elements in the buffer. This forces the requirement that members of the `Vector` (i.e. type `T`) must be default constructable. It also has two efficiency constraints imposed on the Vector:

* You can't pre-allocate space for future members.
    + Resizing (larger or smaller) becomes very expensive, as each resize requires copying all the elements to the newly resized buffer.
    + Alternatively, pre-creating all the elements you need can also be expensive, especially if construction of `T` is expensive.
* The copy constructor is twice as expensive as it should be. Each element must be:
    + Default constructed (when the buffer is created).
    + Then copy constructed with the value from the source vector.


This attempt improves that by allowing efficient pre-allocating of space (`capacity`) for the buffer. New members are added by constructing in-place using [placement new](https://stackoverflow.com/questions/362953/what-are-uses-of-the-c-construct-placement-new).

#### Rule of three third pass
```cpp
template<typename T>
class Vector
{
    std::size_t     capacity;
    std::size_t     length;
    T*              buffer;
    Vector(int capacity)
        : capacity(capacity = 100)
        , length(0)
        // Allocates space but does not call the constructor
        , buffer(static_cast<T*>(::operator new(sizeof(T) * capacity)))
        // Useful if the type T has an expensive constructor
        // We preallocate space without initializing it, giving
        // room to grow and shrink the buffer without re-allocating.
    {}
    ~Vector()
    {
        // Because elements are constructed in-place using
        // placement new. Then we must manually call the destructor
        // on the elements.
        for(int loop = 0; loop < length; ++loop)
        {
            // Note we destroy the elements in reverse order.
            buffer[length - 1 - loop].~T();
        }
        ::operator delete(buffer);
    }
    Vector(Vector const& copy)
        : capacity(copy.length)
        , length(0)
        , buffer(static_cast<T*>(::operator new(sizeof(T) * capacity)))
    {
        // Copy constructor is simple.
        // We create a new resource area of the required length.
        // But these elements are not initialized, so we use push_back to copy them
        // into the new object. This is an improvement because we
        // only construct the members of the vector once.
        for(int loop = 0; loop < copy.length; ++loop)
        {
            push_back(copy.buffer[loop]);
        }
    }
    Vector& operator=(Vector const& copy)
    {
        // Copy Object
        // This is relatively easy. But I want to cover this in detail in a subsequent post.
        return *this;
    }
    void push_back(T const& value)
    {
        // Use placement new to copy buffer into the new buffer
        new (buffer + length) T(value);
        ++length;

        // Note we will handle growing the capacity later.
    }
    void pop_back()
    {
        // When removing elements, you need to manually call the destructor
        // because we created them using placement new.
        --length;
        buffer[length].~T();
    }
};
```

# Rule of Five

In C++11, the language added the concept of "Move Semantics". Rather than having to copy an object (especially on return from a function), we could "move" an object. The concept here is that movement is supposed to be much cheaper than copying because you move the internal data structure of an object rather than all the elements. A good example is a std::vector. Before C++11, a return by value meant copying the object. The constructor of the new object allocates a new internal buffer and then copies all the elements from the original object's buffer to the new object's buffer.

On the other hand, a move simply gives the new object the internal buffer of the old object (we move the pointer to the internal buffer). When an object is moved to another object, the old object should be left in a valid state, but for efficiency, the standard rarely specifies the state of an object after it has been the source of a move. Thus, using an object after a move is a bad idea unless you are setting it to a specific state.

There are two new methods that allow us to specify move semantics on a class.

#### Vector Move Semantics.
```cpp
class Vector
{
    std::size_t     capacity;
    std::size_t     length;
    T*              buffer;
    // STUFF

    // Move Constructor
    Vector(Vector&& move) noexcept;

    // Move Assignment Operator
    Vector& operator=(Vector&& move) noexcept;
};
```

Notice the `&&` operator. This donates an r-value reference and means that your object is the destination of a move operation. The parameter passed is the source object and the state you should use to define your new object's state. After the move, the source object must be left in a valid (but can be undefined state). For a vector, this means it must no longer be the owner of the internal buffer that you are now using in your buffer.

The simplest way to achieve this goal is to set up the object in a valid (but very cheap to achieve state) and then swap the current object with the destination object.

#### Vector Move Semantics Implementation
```cpp
class Vector
{
    std::size_t     capacity;
    std::size_t     length;
    T*              buffer;
    // STUFF

    // Move Constructor
    Vector(Vector&& move) noexcept
        : capacity(0)
        , length(0)
        , buffer(nullptr)
    {
        // The source object now has a nullptr/
        // This object has taken the state of the source object.
        move.swap(*this);
    }

    // Move Assignment Operator
    Vector& operator=(Vector&& move) noexcept
    {
        // In this case simply swap the source object
        // and this object around.
        move.swap(*this);
        return *this;
    }
};
```

Note: I marked both move operators `noexcept`. Assuming the operations are guaranteed not to throw you should mark them as `noexcept`. If we know that certain operations are exception safe, we can optimize resize operations and maintain the strong exception guarantee. This and some other optimizations will be documented in a subsequent post.

# Final Version <a id="VectorVersion-1"></a>

#### Vector Final Version
```cpp
template<typename T>
class Vector
{
    std::size_t     capacity;
    std::size_t     length;
    T*              buffer;

    struct Deleter
    {
        void operator()(T* buffer) const
        {
            ::operator delete(buffer);
        }
    };

    public:
        Vector(int capacity = 10)
            : capacity(capacity)
            , length(0)
            , buffer(static_cast<T*>(::operator new(sizeof(T) * capacity)))
        {}
        ~Vector()
        {
            // Make sure the buffer is deleted even with exceptions
            // This will be called to release the pointer at the end
            // of scope.
            std::unique_ptr<T, Deleter>     deleter(buffer, Deleter());

            // Call the destructor on all the members in reverse order
            for(int loop = 0; loop < length; ++loop)
            {
                // Note we destroy the elements in reverse order.
                buffer[length - 1 - loop].~T();
            }
        }
        Vector(Vector const& copy)
            : capacity(copy.length)
            , length(0)
            , buffer(static_cast<T*>(::operator new(sizeof(T) * capacity)))
        {
            try
            {
                for(int loop = 0; loop < copy.length; ++loop)
                {
                    push_back(copy.buffer[loop]);
                }
            }
            catch(...)
            {
                std::unique_ptr<T, Deleter>     deleter(buffer, Deleter());
                // If there was an exception then destroy everything
                // that was created to make it exception safe.
                for(int loop = 0; loop < length; ++loop)
                {
                    buffer[length - 1 - loop].~T();
                }

                // Make sure the exceptions continue propagating after
                // the cleanup has completed.
                throw;
            }
        }
        Vector& operator=(Vector const& copy)
        {
            // Covered in Part 2
            return *this;
        }
        Vector(Vector&& move) noexcept
            : capacity(0)
            , length(0)
            , buffer(nullptr)
        {
            move.swap(*this);
        }
        Vector& operator=(Vector&& move) noexcept
        {
            move.swap(*this);
            return *this;
        }
        void swap(Vector& other) noexcept
        {
            using std::swap;
            swap(capacity,      other.capacity);
            swap(length,        other.length);
            swap(buffer,        other.buffer);
        }
        void push_back(T const& value)
        {
            resizeIfRequire();
            new (buffer + length) T(value);
            ++length;
        }
        void pop_back()
        {
            --length;
            buffer[length].~T();
        }
    private:
        void resizeIfRequire()
        {
            if (length == capacity)
            {
                // Covered in Part 2
            }
        }
};
```
# Summary
This article shows how to handle the basic resource management required by a vector and covers several important principles for C++ programmers.

* Separation Of Concerns
* Rule of Zero
* Rule of Three
* Rule of Five
* Default compiler generated methods
* Shallow Copy Problem
* Placement New
* Exception Guarantees

# Sources  <a id="Sources"></a>
Looking at [CodeReview.stackexchange.com](https://CodeReview.stackexchange.com); reimplementing the vector class is a common goal for a first project.

* 2011/Nov/07 - [Mathematical Vector2 class implementation](https://codereview.stackexchange.com/q/5856/507)*
* 2012/May/21 - [C++ Vector2 Class Review](https://codereview.stackexchange.com/q/11934/507)*
* 2012/Aug/17 - [Templated Matrix class](https://codereview.stackexchange.com/q/14784/507)
* 2013/Jan/07 - [Vector implementation - simple replacement](https://codereview.stackexchange.com/q/20243/507)
* 2013/May/25 - [Review of 2d Vector class](https://codereview.stackexchange.com/q/26608/507)
* 2013/Jun/19 - [Simple matrix class](https://codereview.stackexchange.com/q/27573/507)
* 2013/Jun/21 - [Matrix and Vector4 classes](https://codereview.stackexchange.com/q/27625/507)*
* 2013/Jun/25 - [Simple matrix class - version 2](https://codereview.stackexchange.com/q/27752/507)*
* 2013/Aug/03 - [Template vector class](https://codereview.stackexchange.com/q/29331/507)*
* 2014/Feb/20 - [C++ vector implementation](https://codereview.stackexchange.com/q/42297/507)
* 2014/Mar/01 - [Reimplementation of C++ vector](https://codereview.stackexchange.com/q/43136/507)
* 2014/Mar/12 - [3D mathematical vector class](https://codereview.stackexchange.com/q/44167/507)
* 2014/May/17 - [Creating a custom Vector class](https://codereview.stackexchange.com/q/50975/507)
* 2014/Aug/19 - [STL vector implementation](https://codereview.stackexchange.com/q/60484/507)
* 2014/Sep/12 - [C++ 3D Vector Implementation](https://codereview.stackexchange.com/a/62774/507)
* 2014/Sep/26 - [Custom mathematical vector class](https://codereview.stackexchange.com/q/63970/507)
* 2014/Oct/19 - [Vector backed by memory pages](https://codereview.stackexchange.com/q/67209/507)
* 2014/Oct/31 - [Custom matrix class](https://codereview.stackexchange.com/q/68486/507)
* 2014/Nov/25 - [Vector/matrix class](https://codereview.stackexchange.com/q/70815/507)
* 2014/Dec/22 - [Vector implementation](https://codereview.stackexchange.com/q/74521/507)
* 2015/Feb/17 - [Mathematical matrices implementation](https://codereview.stackexchange.com/q/81751/507)
* 2015/Mar/01 - [C++ vector implementation errors](https://codereview.stackexchange.com/q/82906/507)
* 2015/Jun/20 - [Implementation of std::vector class](https://codereview.stackexchange.com/q/94211/507)
* 2015/Jul/08 - [Second implementation of std::vector](https://codereview.stackexchange.com/q/96253/507)
* 2015/Oct/17 - [Simple multi-dimensional Array class in C++11](https://codereview.stackexchange.com/q/107877/507)
* 2015/Oct/19 - [Creating n-dimensional mathematical vector classes through inheritance](https://codereview.stackexchange.com/q/108072/507)
* 2015/Oct/20 - [Implementation of Vector in C++](https://codereview.stackexchange.com/q/108140/507)
* 2015/Oct/23 - [Simple multi-dimensional Array class in C++11 - follow-up](https://codereview.stackexchange.com/q/108558/507)
* 2015/Nov/18 - [Custom vector that uses less memory than std::vector](https://codereview.stackexchange.com/q/111114/507)
* 2015/Nov/24 - [Attempt at templates by creating a class for N-dimensional mathematical vectors](https://codereview.stackexchange.com/q/111746/507)
* 2016/Jan/10 - [Vector Implementation C++](https://codereview.stackexchange.com/q/116377/507)
