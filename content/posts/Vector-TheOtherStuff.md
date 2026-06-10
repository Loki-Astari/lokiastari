---
title: Vector - The Other Stuff
date: 2016-03-20T22:26:43-0700
author: Loki Astari, (C)2016
comments: true
categories:
- C++
- Vector
- C++-By-Example
- Coding
series:
- Vector
tags:
- Vector
subtitle: C++ By Example
description: C++ By Example. The Vector Part 5. So, the C++ standard specifies a set
  of requirements for containers. Very few requirements are specified in terms of
  containers, so adhering to these exactly is not required (unless you want to be
  considered for the standard). But they provide an insight into what can be done
  with them, and if you support them, it will allow your container to be more easily
  used with some features of the language and standard library. I am not going to
  go over all of them here (that is left as an exercise for the reader), but I will
  go over the ones I would expect to see in a simple implementation (the kind you
  would see in a university project).
draft: false
cover:
  image: /images/post/post-1.png
  hidden: false
  caption: Photo by Possessed Photography
---

So, the C++ standard specifies a set of requirements for containers. Very few requirements are specified in terms of containers, so adhering to these strictly is not required (unless you want to be considered for the standard). But they provide an insight into what can be done with them, and if you support them, it will allow your container to be more easily used with some features of the language and standard library. I am not going to go over all of them here (that is left as an exercise for the reader), but I will go over the ones I would expect to see in a simple implementation (the kind you would see in a university project).

For details, see the [latest copy of the C++ standard](https://stackoverflow.com/a/4653479/14065).

* 23.2.1  General container requirements [container.requirements.general]
* 23.2.3  Sequence containers [sequence.reqmts]


#### Internal Types
* value&#95;type
* reference
* const&#95;reference
* iterator
* const&#95;iterator
* difference&#95;type
* size&#95;type

It is worth specifying the internal types defined here, as this allows you to abstract the implementation details of the container. This will allow you to change the implementation details without users having to change their implementation, as long as the changes still provide the same interface, but the interface to reference/pointers/iterators is relatively trivial and well defined.

#### Constructors

In C++11, the `std::initializer_list<T>` was introduced. This allows a better list initialization syntax to be used with user-defined types. Since this is usually defined in terms of the range-based construction, we should probably add both of these constructors.

* Vector(std::initializer&#95;list&lt;T&gt; const& list)
* Vector(I begin, I end)

#### Iterators
* begin()
* rbegin()
* begin() const
* rbegin() const
* cbegin() const
* crbegin() const
* end()
* rend()
* end() const
* cend() const
* rend() const
* crend() const

The iterators are relatively easy to write. They also allow the container to be used with the new range-based for that was added in C++14. So this becomes another easy ad.

#### Member Access
* at(&lt;index&gt;)
* at(&lt;index&gt;) const
* operator&#91;&#93;(&lt;index&gt;)
* operator&#91;&#93;(&lt;index&gt;) const
* front()
* back()
* front() const
* back() const

Member access to a vector should be very efficient. As a result, normally range checks are not performed on member access, i.e., the user is expected to make sure that the method preconditions have been met before calling the method. This results in very efficient access to the members of a `Vector`. This is not usually a problem because index ranges are normally checked as part of a loop range; as long as these are validated against the size of the array, it does not need to be validated again.

#### For Loop Vector Access
```cpp
Vector<T>   d = getData();
for(int loop = 0; loop < d.size(); ++loop)
{
    std::cout << d[loop];   // No need for antoher range
                            // check here as we know that loop is inside the
                            // bounds of the vector d.
}
```

There is also the `at()` method, which validates the index provided before accessing the element (throwing an exception if the index is out of range).


#### Non-Mutating Member Functions
* size() const
* bool() const

To allow us to check the preconditions on the element access methods, we need some functions that check the state of the object. These are provided here.

#### Mutating Member Functions
* push&#95;back(&lt;object-ref&gt;)
* push&#95;back(&lt;object-rvalue-ref&gt;)
* emplace&#95;back(&lt;args...&gt;)
* pop&#95;back()

The following members are standard easy to implement methods of `std::vector` (O(1)) that I would expect to see in every implementation.

The other mutating member functions are less trivial as they require elements to be moved around. They are not that hard, but you must put some thought into the most efficient techniques to move elements (i.e., move or copy) and make sure that multiple inserts do not exceed capacity. As a result, I would expect to see these methods only on an as-needed basis.

#### Comparators
* operator== const
* operator!= const

Easy comparison operators.
Optionally, you can provide the other comparison operators.


# Final

```cpp
#include <type_traits>
#include <memory>
#include <algorithm>
#include <stdexcept>
#include <iterator>

template<typename T>
class Vector
{
    public:
        using value_type        = T;
        using reference         = T&;
        using const_reference   = T const&;
        using pointer           = T*;
        using const_pointer     = T const*;
        using iterator          = T*;
        using const_iterator    = T const*;
        using riterator         = std::reverse_iterator<iterator>;
        using const_riterator   = std::reverse_iterator<const_iterator>;
        using difference_type   = std::ptrdiff_t;
        using size_type         = std::size_t;

    private:
        size_type       capacity;
        size_type       length;
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
        template<typename I>
        Vector(I begin, I end)
            : capacity(std::distance(begin, end))
            , length(0)
            , buffer(static_cast<T*>(::operator new(sizeof(T) * capacity)))
        {
            for(auto loop = begin;loop != end; ++loop)
            {
                pushBackInternal(*loop);
            }
        }
        Vector(std::initializer_list<T> const& list)
            : Vector(std::begin(list), std::end(list))
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

        // Non-Mutating functions
        size_type       size() const                      {return length;}
        bool            empty() const                     {return length == 0;}

        // Validated element access
        reference       at(size_type index)               {validateIndex(index);
                                                           return buffer[index];}
        const_reference at(size_type index) const         {validateIndex(index);
                                                           return buffer[index];}

        // Non-Validated element access
        reference       operator[](size_type index)       {return buffer[index];}
        const_reference operator[](size_type index) const {return buffer[index];}
        reference       front()                           {return buffer[0];}
        const_reference front() const                     {return buffer[0];}
        reference       back()                            {return buffer[length - 1];}
        const_reference back() const                      {return buffer[length - 1];}

        // Iterators
        iterator        begin()                           {return buffer;}
        riterator       rbegin()                          {return riterator(end());}
        const_iterator  begin() const                     {return buffer;}
        const_riterator rbegin() const                    {return const_riterator(end());}

        iterator        end()                             {return buffer + length;}
        riterator       rend()                            {return riterator(begin());}
        const_iterator  end() const                       {return buffer + length;}
        const_riterator rend() const                      {return const_riterator(begin());}

        const_iterator  cbegin() const                    {return begin();}
        const_riterator crbegin() const                   {return rbegin();}
        const_iterator  cend() const                      {return end();}
        const_riterator crend() const                     {return rend();}

        // Comparison
        bool operator!=(Vector const& rhs) const {return !(*this == rhs);}
        bool operator==(Vector const& rhs) const
        {
            return  (size() == rhs.size())
                &&  std::equal(begin(), end(), rhs.begin());
        }

        // Mutating functions
        void push_back(value_type const& value)
        {
            resizeIfRequire();
            pushBackInternal(value);
        }
        void push_back(value_type&& value)
        {
            resizeIfRequire();
            moveBackInternal(std::move(value));
        }
        template<typename... Args>
        void emplace_back(Args&&... args)
        {
            resizeIfRequire();
            emplaceBackInternal(std::move(args)...);
        }
        void pop_back()
        {
            --length;
            buffer[length].~T();
        }
        void reserve(size_type capacityUpperBound)
        {
            if (capacityUpperBound > capacity)
            {
                reserveCapacity(capacityUpperBound);
            }
        }
    private:
        void validateIndex(size_type index) const
        {
            if (index >= length)
            {
                throw std::out_of_range("Out of Range");
            }
        }

        void resizeIfRequire()
        {
            if (length == capacity)
            {
                size_type     newCapacity  = std::max(2.0, capacity * 1.5);
                reserveCapacity(newCapacity);
            }
        }
        void reserveCapacity(size_type newCapacity)
        {
            Vector<T>  tmpBuffer(newCapacity);

            simpleCopy<T>(tmpBuffer);

            tmpBuffer.swap(*this);
        }

        // Add new element to the end using placement new
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
        template<typename... Args>
        void emplaceBackInternal(Args&&... args)
        {
            new (buffer + length) T(std::move(args)...);
            ++length;
        }

        // Optimizations that use SFINAE to only instantiate one
        // of two versions of a function.
        //      simpleCopy()        Moves when no exceptions are guaranteed;
        //                          otherwise, copies.
        //      clearElements()     When no destructor remove loop.
        //      copyAssign()        Avoid resource allocation when no exceptions guaranteed.
        //                          ie. When copying integers, reuse the buffer if we can
        //                          to avoid expensive resource allocation.

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
            // This function is only used if there is no chance of an exception being
            // thrown during destruction or copy construction of the type T.


            // Quick return for self assignment.
            if (this == &copy)
            {
                return;
            }

            if (capacity <= copy.length)
            {
                // If we have enough space to copy, then reuse the space we currently
                // have to avoid the need to perform an expensive resource allocation.

                clearElements<T>();     // Potentially does nothing (see above)
                                        // But if required will call the destructor of
                                        // all elements.

                // buffer now ready to get a copy of the data.
                length = 0;
                for(int loop = 0; loop < copy.length; ++loop)
                {
                    pushBackInternal(copy[loop]);
                }
            }
            else
            {
                // Fallback to copy and swap if we need more space anyway
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
