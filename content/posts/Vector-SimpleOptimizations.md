---
title: Vector - Simple Optimizations
date: 2016-03-19T15:06:40-0700
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
description: C++ By Example. The Vector Part 4. We look at a couple of other types
  available in the template utility library that allow optimization via SFINAE.
draft: false
cover:
  image: /images/post/post-8.png
  hidden: false
  caption: Photo by Matthew Brodeur
---

Now that we have used `std::is_nothrow_move_constructible,` we can also examine a couple of other types available in the template utility library.

# Optimized Destruction

Since we have to manually call the destructor on all objects in the container (because we are using placement new), we can look to see if we can optimize that. The type `std::is_trivially_destructible` detects if the type is **Trivially** destructible. This basically means that there will be no side effects from the destructor (See: Section 12.4 Paragraph 5 of the standard). For types we don't need to call the destructor of the object. For the `Vector` class, this means we can eliminate the call to the destructor but, more importantly, the loop.

#### Destroying Elements
```cpp
~Vector()
{
    // STUFF..

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
        // STUFF 1 ...
    }
    catch(...)
    {
        // STUFF 2 ...
        // If there was an exception then destroy everything
        // that was created to make it exception safe.
        for(int loop = 0; loop < length; ++loop)
        {
            buffer[length - 1 - loop].~T();
        }
        throw;
    }
}
```

We can use the same SFINAE technique that we used in the previous article to remove the loops when the contained type is trivially destructible.

```cpp
~Vector()
{
    // STUFF..
    clearElements<T>();
}
Vector(Vector const& copy)
    : capacity(copy.length)
    , length(0)
    , buffer(static_cast<T*>(::operator new(sizeof(T) * capacity)))
{
    try
    {
        // STUFF 1 ...
    }
    catch(...)
    {
        // STUFF 2 ...
        clearElements<T>();
        throw;
    }
}

template<typename X>
typename std::enable_if<std::std::is_trivially_destructible<X>::value == false>::type
clearElements()
{
    // Call the destructor on all the members in reverse order
    for(int loop = 0; loop < length; ++loop)
    {
        // Note we destroy the elements in reverse order.
        buffer[length - 1 - loop].~T();
    }
}

template<typename X>
typename std::enable_if<std::std::is_trivially_destructible<X>::value == true>::type
clearElements()
{
    // Trivially destructible objects can be reused without using the destructor.
}
```

# Optimized Assignment Operator
The final optimization is because resource allocation is expensive, if we can avoid the resource allocation altogether and reuse the space we currently have.

#### Copy Assignment
```cpp
Vector& operator=(Vector const& copy)
{
    // Copy and Swap idiom
    Vector<T>  tmp(copy);
    tmp.swap(*this);
    return *this;
}
```

The copy and swap idiom is perfect for providing the strong exception guarantee in the presence of exceptions. **But** if there are no exceptions during destruction or construction, then we can potentially reuse the available memory. So if we rewrote the assignment operator with the assumption that there were no exceptions, it would look like the following (Note: In the real code, use SFINAE to do the optimization only when necessary).

#### Copy the easy way
```cpp
Vector& operator=(Vector const& copy)
{
    // Check for self assignment
    // As we are doing work anyway.
    if (this == &copy)
    {
        return *this;
    }

    // If the length of the `copy` object exceeds
    // the capacity of the current object then
    // we have to do resource management. It costs
    // nothing extra to use the copy and swap idiom
    if (copy.length > capacity)
    {
        // Copy and Swap idiom
        Vector<T>  tmp(copy);
        tmp.swap(*this);
        return *this;
    }

    // The optimization happens here.
    // We can reuse the buffer we already have.
    clearElements<T>();     // use clearElements() as it probably does very little.
    length = 0;

    // Now add the elements to this container as cheaply as possible.
    for(int loop = 0; loop < copy.length; ++loop)
    {
        pushBackInternal(copy[loop]);
    }
    return *this;
}
```

# Final Version <a id="VectorVersion-4"></a>

The final version

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

            clearElements<T>();
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
                clearElements<T>();

                // Make sure the exceptions continue propagating after
                // the cleanup has completed.
                throw;
            }
        }
        Vector& operator=(Vector const& copy)
        {
            copyAssign<T>(copy);
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
        void reserveCapacity(std::size_t newCapacity)
        {
            Vector<T>  tmpBuffer(newCapacity);

            simpleCopy<T>(tmpBuffer);

            tmpBuffer.swap(*this);
        }
        void pushBackInternal(T const& value)
        {
            new (buffer + length) T(value);
            ++length;
        }
        void moveBackInternal(T&& value)
        {
            new (buffer + length) T(std::move(value));
            ++length;
        }

        template<typename X>
        typename std::enable_if<std::is_nothrow_move_constructible<X>::value == false>::type
        simpleCopy(Vector<T>& dst)
        {
            std::for_each(buffer, buffer + length,
                          [&dst](T const& v){dst.pushBackInternal(v);}
                         );
        }

        template<typename X>
        typename std::enable_if<std::is_nothrow_move_constructible<X>::value == true>::type
        simpleCopy(Vector<T>& dst)
        {
            std::for_each(buffer, buffer + length,
                          [&dst](T& v){dst.moveBackInternal(std::move(v));}
                         );
        }

        template<typename X>
        typename std::enable_if<std::is_trivially_destructible<X>::value == false>::type
        clearElements()
        {
            // Call the destructor on all the members in reverse order
            for(int loop = 0; loop < length; ++loop)
            {
                // Note we destroy the elements in reverse order.
                buffer[length - 1 - loop].~T();
            }
        }

        template<typename X>
        typename std::enable_if<std::is_trivially_destructible<X>::value == true>::type
        clearElements()
        {
            // Trivially destructible objects can be reused without using the destructor.
        }

        template<typename X>
        typename std::enable_if<(std::is_nothrow_copy_constructible<X>::value
                             &&  std::is_nothrow_destructible<X>::value) == true>::type
        copyAssign(Vector<X>& copy)
        {
            if (this == &copy)
            {
                return;
            }

            if (capacity <= copy.length)
            {
                clearElements<T>();
                length = 0;
                for(int loop = 0; loop < copy.length; ++loop)
                {
                    pushBackInternal(copy[loop]);
                }
            }
            else
            {
                // Copy and Swap idiom
                Vector<T>  tmp(copy);
                tmp.swap(*this);
            }
        }
        template<typename X>
        typename std::enable_if<(std::is_nothrow_copy_constructible<X>::value
                             &&  std::is_nothrow_destructible<X>::value) == false>::type
        copyAssign(Vector<X>& copy)
        {
            // Copy and Swap idiom
            Vector<T>  tmp(copy);
            tmp.swap(*this);
        }
};
```

# Summary
