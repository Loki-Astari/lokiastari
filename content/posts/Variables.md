---
title: Variables
date: 2013-11-19T09:15:16-0800
author: Loki Astari, (C)2013
comments: true
categories:
- C++
- So-You-Want-To-Learn-C++
- Coding
series:
- So-You-Want-To-Learn-C++
tags:
- So-You-Want-To-Learn-C++
subtitle: So you want to learn C++
description: C++ for beginners. Part 3 Variables. In most programming languages, you
  have the concept of variables. These are named objects that hold a value (more formerly
  referred to as state). By manipulating a variable, you manipulate the state of the
  object the variable refers to.
draft: false
cover:
  image: /images/post/post-5.png
  hidden: false
  caption: Photo by Oscar Nilsson
---

In most programming languages, you have the concept of variables. These are named objects that hold a value (more formerly referred to as state). By manipulating a variable, you manipulate the state of the object the variable refers to.

#### add.cpp
```cpp
void addFunction()
{
    int   x = 0;      // Declare (and initialize) a variable called "x"

    x = x + 5;        // Manipulate the variable "x".
                      // The variable "x" now holds the value "5"

    int   y = x + 3;  // Declare (and initialize) a variable called "y"
                      // This will take the value "8" by adding "+" 3
                      // to the value of "x"
}
```

C++ is a strongly typed language. This means each variable has a specific type that does not change (above that type is **int**). The operations that can be performed on an object depend on the object's type, and the operation's result can depend on the types involved. C++ has several built-in types (listed below) but allows the definition of new user-defined types (which will be described in a later article). The standard library provides a set of commonly used standard types (listed below).

#### Built-in Types
```cpp
char                    // Represents a character.
bool                    // Represents a boolean true/false value.
short                   // Represents an integer of at least 16 bits
int                     // Represents an integer of at least 32 bits
long                    // Represents an integer of at least 32 bits
long long               // Represents an integer of at least 64 bits
float                   // Represents a floating point number
double                  // Represents a double precision floating point number
```

#### Standard Types
```cpp
// This is a list of the most commonly used types (there are many more)
std::string             // Represents a string of characters.
std::vector<T>          // Represents a dynamically sizable array
                        //     of objects with the type 'T'
std::array<T, size>     // Represents a fixed 'size'  array
                        //     of objects with the type 'T'
std::list<T>            // Represents a list of objects with the type 'T'
std::map<Key, Value>    // Represents a dictionary of key, value pairs (index by key).
                        //     The key   has type 'Key'
                        //     The value has type 'Value'
std::set<Key>           // Represents a set of keys of type 'Key'
```

The list may seem a bit daunting at first, but while you are learning if you restrict yourselves to three built-in types (**bool**, **int** and **double**) and two standard types (**std::string** and **std::vector&lt;T&gt;**) you will be able to solve most beginner/training problems.

The other built-in types are usually used when you need a wider range of values or need to save space. The additional standard types (shown above) are different types of containers that provide different access characteristics (which will be explained later). We will eventually cover all these types.

So, an example of usage of the most common types is:

#### variables.cpp
```cpp
#include <string>
#include <vector>
#include <iostream>

int main()
{
    int                       age   = 28;
    std::string               name  = "Loki";
    double                    grade = 12.45;
    std::vector<std::string>  courseNames = { "C++", "Teaching", "Maths", "Art", "Music"};

    std::cout << "Name: " << name  << "\n";
    std::cout << "Age:  " << age   << "\n";
    std::cout << "Grade:" << grade << "\n";
    std::cout << "Course 1: " << courseNames[1] << "\n";
}
```
