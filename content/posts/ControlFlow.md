---
title: Control Flow
date: 2013-12-02T12:00:11-0800
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
description: C++ for beginners. Part 5 Control Flow. So far, we have demonstrated
  basic programs that do a single task without making any decisions. Most (all but
  the most trivial) programming languages provide decision-making constructs (Conditional
  Branching).
draft: false
cover:
  image: /images/post/post-8.png
  hidden: false
  caption: Photo by Matthew Brodeur
---

So far, we have created basic programs that perform a single task without making any decisions. Most (all but the most trivial) programming languages provide decision-making constructs (Conditional Branching).

C++ provides two forms of branching. The **"If Statement"** and the **"Switch Statement"** .

Note: Looping is also a form of branching. The looping concept is extensive enough that we will deal with looping separately in its own article.

### If Statement

The **"If Statement"** allows code to be executed when a specific condition is fulfilled and optionally an alternative piece of code otherwise.

#### ifstatement.cpp
```cpp
// First version of "If Statement"
// Execute code if <Condition> is true.
//
if (<Condition>)
{
    <code to execute if Condition is true>
}


// Second version of "If Statement"
// Execute code1 if <Condition> is true or code2 if <Condition> is false
//
if (<Condition>)
{
    <code1: execute if Condition is true>
}
else
{
    <code2: execute if Condition is false>
}
```

The standard comparison operators that you find in most languages can be used. These operators are defined for all the built-in types. User-defined types in the standard library are defined in ways that make their usage obvious. When you define these for your user-defined types, you should also make sure that they behave in the logical manner described below; the language does not enforce this, **BUT** if you don't follow this suggestion, your types will scare people, and they will not be used, so follow the expected behavior.

#### Standard Comparison Operators
```text
/*
| Operator  | Usage   | Result Type | Meaning                                                  |
| ----------|---------|-------------|----------------------------------------------------------|
|    !      |  !A     |  bool       | Not A.                                                   |
|           |         |             | If A is true, then false; if A is false, then true       |
|           |         |             | If A is not a bool type, it is converted (see below)     |
|    ==     |  A == B |  bool       | Equality: true if A and B are logically equivalent;      |
|           |         |             | otherwise, false.                                        |
|    !=     |  A != B |  bool       | Should mean !(A == B)                                    |
|    <      |  A <  B |  bool       | true if A is logically less than B.                      |
|    <=     |  A <= B |  bool       | true if A is logically less than or equal to B.          |
|    >      |  A >  B |  bool       | true if A is logically greater than B.                   |
|    >=     |  a >= B |  bool       | true if A is logically greater than or equal to B.       |
|    &&     |  A && B |  bool       | true if A is true **AND** B is true.                     |
|           |         |             | If the expressions A or B are not bool, then             |
|           |         |             | it is converted (see below).                             |
|           |         |             | Also worth noting is that if A is **false** then the     |
|           |         |             | expression for B is not evaluated.                       |
|           |         |             | This is known as a shortcut operator.                    |
|           |         |             | we will describe this later.                             |
|    ||     |  A || B |  bool       | true if A is true **OR** B is true.                      |
|           |         |             | If the expressions A or B are not actually a bool then   |
|           |         |             | it is converted (see below). Also worth noting is that   |
|           |         |             | if A is **true** then the expression for B is not        |
|           |         |             | evaluated.                                               |
|           |         |             | This is known as a shortcut operator.                    |
|           |         |             | we will describe this later.                             |
|-----------|---------|-------------|----------------------------------------------------------|
*/
```

If the expression you use in &lt;Condition&gt; does not result in a bool value, the compiler will insert a conversion that will result in a bool (true/false) value. If no conversion is possible, it results in a compile-time error.

#### Type conversion
```text
/*
| Type             | false      | true            | Notes                                      |
|------------------|------------|-----------------|--------------------------------------------|
| bool             | false      | true            | No actual conversion used.                 |
| Integers         | 0          | (anything else) | Integer shorthand for (char/short/int/long)|
| Pointers         | NULL       | (anything else) | Will discuss pointers in detail later.     |
| User Define Type | ?          | ?               | If a cast operator to bool/Integer/pointer |
|                  |            |                 | exists this will be used.                  |
|------------------|------------|-----------------|--------------------------------------------|
*/
```

An example of using an **If Statement**:

#### itest.cpp
```cpp
#include <iostream>
#include <string>

int main()
{
    std::string    name;
    std::cout << "Please enter your name\n";
    std::cin >> name;

    if (name == "Loki")
    {
        std::cout << "Hello Admin\n";
    }
    else
    {
        std::cout << "Hello Muggle\n";
    }

    int   value;
    std::cin >> value;
    std::cout << "Please enter a non-zero integer value.\n";
    if (value) // integer value converted to bool
    {
        std::cout << "You got it correct. Must use a non-zero value.\n";
    }
}
```

### Switch Statement

The **"Switch Statement"** is an alternative to the **"If Statement"**. Prefer the switch when you have many options derived from the same expression. Unlike other high-level languages, C++ can only use **Integer** types in a switch statement; thus, in all `case <Value>` the &lt;Value&gt; must be an integer **literal** value.

#### switch.cpp
```cpp
switch(<Test Expression>)
{
    case <value1>:
    {
        <code1>
        break;
    }
    case <value2>:
    {
        <code2>
        break;
    }
    case <value3>:
    {
        <code3>
        break;
    }
    default:
    {
        <code Default>
        break
    }
}
//
//
// Equivalent "If Statement"

int test = <Test Expression>;
if (<value1> == test)
{
    <code1>
}
else
{
    if (<value2> == test)
    {
        <code2>
    }
    else
    {
        if (<value3> == test)
        {
            <code3>
        }
        else
        {
            <code Default>
        }
    }
}
```

If you use a non-integer expression in the switch statement, the compiler will try to convert the value to an integer. If this is not possible, it generates a compile-time error.

#### switch.cpp
```cpp
    #include <iostream>

    int main()
    {
        int  value;
        std::cout << "Input a value between 0 and 5\n";
        std::cin >> value;

        switch(value)
        {
            case 0: {std::cout << "You used zero\n";    break;}
            case 1: {std::cout << "You used one\n";     break;}
            case 2: {std::cout << "You used two\n";     break;}
            case 3: {std::cout << "You used three\n";   break;}
            case 4: {std::cout << "You used four\n";    break;}
            case 5: {std::cout << "You used five\n";    break;}
            default: {std::cout << "You failed to follow instructions\n";break;}
        }
    }
```

Note I: The language does not require you to use a **Break Statement** in each block. **BUT** you should; compilers will warn you when you don't.

Note II: You should always use a **Default Statement** . If the value does not hit a value specified in a **Case Statement**, then the **Default Statement** is used; If the **Default Statement** is not defined in this situation, it results in undefined behavior. To avoid problems, you should always define the **Default Statement**, even if all this does is generate an error. This will avoid maintenance issues down the road.




