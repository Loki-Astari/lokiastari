---
title: Vector - Resize
date: 2016-03-12T05:53:07-0700
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
description: C++ By Example. The Vector Part 3. Because resizing a vector is expensive,
  the standard vector class uses exponential growth to minimize the number of times
  that the vector is resized, a technique we replicate in this version. But every
  now and then, you still need to resize the internal buffer.
draft: false
cover:
  image: /images/post/post-7.png
  hidden: false
  caption: Photo by Alex Knight
---
Because resizing a vector is expensive, the `std::vector` class uses exponential growth to minimize the number of times that the vector is resized: a technique we replicate in this version. But every now and then, you still need to resize the internal buffer.

In the [current version](#VectorVersion-1), resizing the vector requires allocating a new buffer and copying all the members into it. We use the copy and swap idiom to provide the strong exception guarantee (If an exception is thrown, all resources are cleaned up, and the object remains unchanged).

#### Vector Resize with Copy
```cpp
    void pushBackInternal(T const& value)
    {
        new (buffer + length) T(value);
        ++length;
    }

    void reserveCapacity(std::size_t newCapacity)
    {
        Vector<T>  tmpBuffer(newCapacity);
        std::for_each(buffer, buffer + length,
                      [&tmpBuffer](T const& v){tmpBuffer.pushBackInternal(v);}
                     );

        tmpBuffer.swap(*this);
    }
```


# Resize With Move Construction
Thus, resizing a Vector can be a very expensive operation because of all the copying that can occur.

Using the move constructor rather than the copy constructor during a resize operation could potentially be much more efficient. However, the move constructor mutates the original object, and if there is a problem, we need to undo the mutations to maintain the strong exception guarantee.

The first attempt at this is:

#### Vector Resize with Move With Exceptions
```cpp
void moveBackInternal(T&& value)
{
    new (buffer + length) T(std::move(value));
    ++length;
}

void reserveCapacity(std::size_t newCapacity)
{
    Vector<T>  tmpBuffer(newCapacity);
    try
    {
        std::for_each(buffer, buffer + length,
                      [&tmpBuffer](T& v){tmpBuffer.moveBackInternal(std::move(v));}
                     );
    }
    catch(...)
    {
        // If an exception is thrown you need to move the objects back
        // from the temporary buffer back to this object.
        for(int loop=0; loop < tmpBuffer.length; ++loop)
        {
            // The problem is here:
            // If the initial move can throw,
            // then trying to move any of the objects back can also throw.
            // which would leave the object in an inconsistent state.
            buffer[loop] = std::move(tmpBuffer[loop]);
        }

        // Then remember to rethrow the exception after we have fixed the state.
        throw;
    }

    tmpBuffer.swap(*this);
}
```
# Resize With NoThrow Move Construction
As the above code shows, if the type `T` can throw during its move constructor, you can't guarantee that the object will be returned to its original state (moving the elements back may cause another exception). So, we cannot use the move constructor to resize the vector if the type `T` can throw during move construction.

However, not all types throw when being moved. In fact, it is recommended that move constructors never throw. If we can guarantee that the move constructor does not throw, we can simplify the above code considerably and still provide a strong exception guarantee.

#### Vector Resize with Move
```cpp
void reserveCapacity(std::size_t newCapacity)
{
    Vector<T>  tmpBuffer(newCapacity);
    std::for_each(buffer, buffer + length,
                  [&tmpBuffer](T& v){tmpBuffer.moveBackInternal(std::move(v));}
                 );

    tmpBuffer.swap(*this);
}
void moveBackInternal(T&& value)
{
    new (buffer + length) T(std::move(value));
    ++length;
}
```
# Resize Template Specialization
So now we have to write the code that decides which version we should use at compile time. The simplest way to do this is to use template specialization of a class using the standard class `std::is_nothrow_move_constructible<T>` to help differentiate types with a non-throwing move constructor. This is simple enough:

#### Template class Specialization
```cpp
template<typename T, bool = std::is_nothrow_move_constructible<T>::value>
struct SimpleCopy
{
    // Define two different versions of this class.
    // The object is to copy all the elements from src to dst Vector
    // using pushBackInternal or moveBackInternal
    //
    // SimpleCopy<T, false>:        Copy Constructor
    //                              Defines a version that uses pushBackInternal 
    //                              This is always safe to use.
    // SimpleCopy<T, true>:         Move Constructor
    //                              Defines a version that uses moveBackInternal
    //                              Safe when move construction does not throw.
    //
    void operator()(Vector<T>& src, Vector<T>& dst) const;
};
template<typename T>
class Vector
{
    public:
        .....
    private:
        // We are using private methods for efficiency.
        // So these classes need to be friends.
        friend struct SimpleCopy<T, true>;
        friend struct SimpleCopy<T, false>;

        void reserveCapacity(std::size_t newCapacity)
        {
            Vector<T>  tmpBuffer(newCapacity);

            // Create the copier object based on the type T.
            // Note: The second parameter is automatically generated based
            //       on if the type T is move constructable with no exception.
            SimpleCopy<T>   copier;
            copier(*this, tmpBuffer);

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
}
// Define the two different types of copier
template<typename T>
struct SimpleCopy<T, false> // false: does not have nothrow move constructor
{
    void operator()(Vector<T>& src, Vector<T>& dst) const
    {
        std::for_each(buffer, buffer + length,
                      [&dst](T const& v){dst.pushBackInternal(v);}
                     );
    }
};
template<typename T>
struct SimpleCopy<T, true> // true: has a nothrow move constructor
{
    void operator()(Vector<T>& src, Vector<T>& dst) const
    {
        std::for_each(buffer, buffer + length,
                      [&dst](T& v){dst.moveBackInternal(std::move(v));}
                     );
    }
};
```
# Resize With NoThrow SFINAE
The above technique has a couple of issues.

The type `SimpleClass` is publicly available and is a friend of `Vector<T>`. This makes it susceptible to accidentally being used (even if not explicitly documented). Unfortunately, it can't be included as a member class and also be specialized.

Additionally, it looks awful!!

But we can also use [SFINAE](https://en.wikipedia.org/wiki/Substitution_failure_is_not_an_error) and method overloading.

SFINAE allows us to define several versions of a method with exactly the same arguments, as long as only one is valid at compile time. So in the example below, we define two versions of the method `SimpleCopy(Vector<T>& src, Vector<T>& dst)` but then use `std::enable_if` to make sure only one version of the function is valid at compile time.

#### SFINAE method overload
```cpp
template<typename T>
class Vector
{
    public:
        .....
    private:
        void reserveCapacity(std::size_t newCapacity)
        {
            Vector<T>  tmpBuffer(newCapacity);

            SimpleCopy<T>(*this, tmpBuffer);

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
        // Note: this defines the return type of the function.
        //       But only one has a valid member `type` thus only
        //       one of the following functions is actually valid.
        typename std::enable_if<std::is_nothrow_move_constructible<X>::value == false>::type
        simpleCopy(Vector<T>& src, Vector<T>& dst)
        {
            std::for_each(buffer, buffer + length,
                          [&dst](T const& v){dst.pushBackInternal(v);}
                         );
        }

        template<typename X>
        typename std::enable_if<std::is_nothrow_move_constructible<X>::value == true>::type
        simpleCopy()(Vector<T>& src, Vector<T>& dst)
        {
            std::for_each(buffer, buffer + length,
                          [&dst](T& v){dst.moveBackInternal(std::move(v));}
                         );
        }
}
```

# Final Version <a id="VectorVersion-3"></a>

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
};
```

# Summary
This article has gone over the design of resizing the internal buffer. We have covered a couple of techniques on the way:

* Move Constructor Concepts
* Template Class Specialization
* SFINAE
* std::is_nothrow_move_constructible&lt;&gt;
* std::enable_if&lt;&gt;
