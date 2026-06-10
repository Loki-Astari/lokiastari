---
title: Vector - Resource Management Copy Swap
date: 2016-02-29T12:29:20-0800
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
description: C++ By Example. The Vector Part 2. In the previous article, I went over
  basic allocation for a `Vector` like class. In this article, I want to put some
  detail around the copy assignment operator and resizing the underlying `Vector`.
  Unlike the other methods previously discussed, these methods have to deal with both
  construction and destruction of elements and the potential of exceptions interrupting
  the processes. The goal is to provide exception safe methods that provide the strong
  exception guarantee for the object and do not leak resources.
draft: false
cover:
  image: /images/post/post-6.png
  hidden: false
  caption: Photo by Ben Griffiths
---

In the previous article, I went over basic allocation for a `Vector` like class. In this article, I want to put some detail around the copy assignment operator and resizing the underlying `Vector`. Unlike the other methods previously discussed, these methods have to deal with both construction and destruction of elements and the potential of exceptions interrupting the processes. The goal is to provide exception safe methods that provide the strong exception guarantee for the object and do not leak resources.

# Copy Assignment
## First Try
This is a very common first attempt at a copy constructor.
It simply calls the destructor on all elements currently in the object. Then uses the existing `push_back()` method to copy member elements from the source object, thus allowing the object to automatically resize as required.

#### Copy Assignment (Try 1)
```cpp
class Vector
{
    std::size_t     capacity;
    std::size_t     length;
    T*              buffer;
    // STUFF
    Vector& operator=(Vector const& copy)
    {
        if (&copy == this)
        {
            // Early exit for self assignment
            return *this;
        }
        // First, we have to destroy all the current elements.
        for(int loop = 0; loop < length; ++loop)
        {
            // Destroy in reverse order
            buffer[length - 1 - loop].~T();
        }
        // Now the buffer is empty so reset size to zero.
        length = 0;

        // Now copy all the elements from the source into this object
        for(int loop = 0; loop < copy.length; ++loop)
        {
            push_back(copy.buffer[loop]);
        }
        return *this;
    }
};
```

## Strong Exception Guarantee
The problem here is that this does not provide the strong exception guarantee. If any of the constructors/destructor throw, then the object will be left in an inconsistent state with no way to restore the original state. The strong exception guarantee basically means that the operation works or does not change the state of the object. The most straightforward technique to achieve this is to create a copy in a new temporary buffer that can be thrown away if things go wrong (leaving the current object untouched). If the copy succeeds, then we use it and throw away the original data.

#### Copy Assignment (Try 2)
```cpp
class Vector
{
    std::size_t     capacity;
    std::size_t     length;
    T*              buffer;
    // STUFF
    Vector& operator=(Vector const& copy)
    {
        if (&copy == this)
        {
            // Early exit for self assignment
            return *this;
        }
        // Part-1 Create a copy of the src object.
        std::size_t tmpCap    = copy.length;
        std::size_t tmpSize   = 0;
        T*          tmpBuffer = static_cast<T*>(::operator new(sizeof(T) * tmpCap));

        // Now copy all the elements from the source into the temporary object
        for(int loop = 0; loop < copy.length; ++loop)
        {
            new (tmpBuffer + tmpSize) T(copy.buffer[loop]);
            ++tmpSize;
        }

        // Part-2 Swap the state
        // We have successfully created the new version of this object
        // So swap the temporary and object states.
        std::swap(tmpCap,    capacity);
        std::swap(tmpSize,   length);
        std::swap(tmpBuffer, buffer);

        // Part-3 Destroy the old state.
        // Now we have to delete the old state.
        // If this fails it does not matter the state of the object is consistent
        for(int loop = 0; loop < tmpSize; ++loop)
        {
            tmpBuffer[tmpSize - 1 - loop].~T();
        }
        ::operator delete(tmpBuffer);
        return *this;
    }
};
```

## Copy and Swap
This second attempt is better. But it still leaks if an exception is thrown. But before we add exception handling, let us take a closer look at the three sections of the assignment operator.

Part-1 looks exactly like the copy constructor of Vector.

#### Copy Assignment Part-1
```cpp
        std::size_t tmpCap    = copy.length;
        std::size_t tmpSize   = 0;
        T*          tmpBuffer = static_cast<T*>(::operator new(sizeof(T) * tmpCap));

        // Now copy all the elements from the source into the temporary object
        for(int loop = 0; loop < copy.length; ++loop)
        {
            // This looks exactly like push_back()
            new (tmpBuffer + tmpSize) T(copy.buffer[loop]);
            ++tmpSize;
        }
```

Part-3 looks exactly like the destructor of Vector.

#### Copy Assignment Part-3
```cpp
        // Now we have to delete the old state.
        for(int loop = 0; loop < tmpSize; ++loop)
        {
            tmpBuffer[tmpSize - 1 - loop].~T();
        }
        ::operator delete(tmpBuffer);
```

Using these two observations, we have a rewrite of the copy assignment operator.

#### Copy Assignment (Try 3)
```cpp
class Vector
{
    std::size_t     capacity;
    std::size_t     length;
    T*              buffer;
    // STUFF
    Vector& operator=(Vector const& copy)
    {
        if (&copy == this)
        {
            // Early exit for self assignment
            return *this;
        }
        // Part-1 Create a copy
        Vector  tmp(copy);

        // Part-2 Swap the state
        std::swap(tmp.capacity, capacity);
        std::swap(tmp.length,   length);
        std::swap(tmp.buffer,   buffer);

        return *this;
        // Part-3 Destructor called at end of scope.
        // No actual code required here.
    }
};
```
## [Copy And Swap Idiom](https://stackoverflow.com/q/3279543/14065)

The copy and swap idiom is about dealing with replacing an object state from another object. It is very commonly used in the copy assignment operator but is usefull whenever state is being changed and the [strong exception guarantee](https://en.wikipedia.org/wiki/Exception_safety) is required.

The above code works perfectly. But in Part-2, the swap looks like a regular swap operation, so let's use that rather than doing it manually. Also, self-assignment now works without the need for a test (because we are copying into a temporary). So we can remove the check for self-assessment. Yes, this does make the performance for self-assignment worse, but it makes the normal operation more efficient. Since self assignment is extremely rare I optimize for the normal case. So one final re-factor of the copy constructor leaves us with this.

#### Copy Assignment (Try 4)
```cpp
class Vector
{
    std::size_t     capacity;
    std::size_t     length;
    T*              buffer;
    // STUFF
    Vector& operator=(Vector const& copy)
    {
        Vector  tmp(copy);
        tmp.swap(*this);
        return *this;
    }
    void swap(Vector& other) noexcept
    {
        std::swap(other.capacity, capacity);
        std::swap(other.length,   length);
        std::swap(other.buffer,   buffer);
    }
};
```

# Resizing Underlying buffer

When pushing data into the array, we need to verify that capacity has not been exceeded. If it has, then we need to allocate more capacity, then copy the current content into the new buffer and destroy the old buffer after calling the destructor on all elements.

## Using Copy and Swap
This operation is exceedingly similar to the description of the copy assignment operator. As a result, the best solution looks very similar and uses the Copy and Swap idiom.

#### Vector Reallocating Buffer
```cpp
class Vector
{
    std::size_t     capacity;
    std::size_t     length;
    T*              buffer;
    // STUFF
    void resizeIfRequire()
    {
        if (length == capacity)
        {
            // Create a temporary object with a larger capacity.
            std::size_t     newCapacity  = std::max(2.0, capacity * 1.5);
            Vector<T>  tmpBuffer(newCapacity);

            // Copy the state of this object into the new object.
            std::for_each(buffer, buffer + length, [&tmpBuffer](T const& item){
                tmpBuffer.push_back(item);
            });

            // All the work has been successfully done. So swap
            // the state of the temporary and the current object.
            tmpBuffer.swap(*this);

            // The temporary object goes out of scope here and
            // tidies up the state that is no longer needed.
        }
    }
};
```

# Final Version <a id="VectorVersion-2"></a>

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
            // Copy and Swap idiom
            Vector<T>  tmp(copy);
            tmp.swap(*this);
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
            pushBackInternal(value);
        }
        void pop_back()
        {
            --length;
            buffer[length].~T();
        }
        void reserve(std::size_t capacityUpperBound)
        {
            if (capacityUpperBound > capacity)
            {
                reserveCapacity(capacityUpperBound);
            }
        }
    private:
        void resizeIfRequire()
        {
            if (length == capacity)
            {
                std::size_t     newCapacity  = std::max(2.0, capacity * 1.5);
                reserveCapacity(newCapacity);
            }
        }
        void pushBackInternal(T const& value)
        {
            new (buffer + length) T(value);
            ++length;
        }
        void reserveCapacity(std::size_t newCapacity)
        {
            Vector<T>  tmpBuffer(newCapacity);
            std::for_each(buffer, buffer + length, [&tmpBuffer](T const& v){
                tmpBuffer.pushBackInternal(v);
            });

            tmpBuffer.swap(*this);
        }
};
```

# Summary
This article has gone over the design of the Copy and Swap idiom and shown how it is used in the Copy Assignment Operator and the resize operation.

* Separation Of Concerns
* Copy and Swap Idiom
* Exception Guarantees
