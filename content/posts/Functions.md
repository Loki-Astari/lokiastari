---
title: Functions
date: 2013-11-24T09:22:04-0800
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
description: C++ for beginners. Part 4 Functions. All C++ applications must have at
  least one function called `main()`. Additionally, you can have user-defined functions
  that encapsulate individual tasks, thus making the code cleaner and easier to read.
  Therefore, this is a helpful feature if you repeat the same task repeatedly with
  only slight variations.
draft: false
cover:
  image: /images/post/post-6.png
  hidden: false
  caption: Photo by Ben Griffiths
---

### Usage
All C++ applications must have at least one function called `main()`. Additionally, you can have user-defined functions that encapsulate individual tasks, thus making the code cleaner and easier to read. Therefore, this is a useful feature if you repeat the same task many times with only slight variations:

#### function1.cpp
```cpp
#include <string>
#include <iostream>

int main(int argc, char* argv[])
{
     std::cout << "What is your first name?\n";
     std::string firstName;
     std::cin >> firstName;

     std::cout << "What is your second name?\n";
     std::string secondName;
     std::cin >> secondName;

     std::cout << "What is your Mother's name?\n";
     std::string motherName;
     std::cin >> motherName;

     std::cout << "What is your Father's name?\n";
     std::string fatherName;
     std::cin >> fatherName;
}
```

The obvious repetition here is easy to spot. We can simplify this code using a function that does all the common work. We can pass anything unique as a parameter to the function.

#### function2.cpp
```cpp
#include <string>
#include <iostream>

std::string getNameFor(std::string who)
{
    std::cout << "What is your " << who << " name?\n";
    std::string result;
    std::cin >> result;
    return result;
}

int main(int argc, char* argv[])
{
     std::string firstName  = getNameFor("first");
     std::string secondName = getNameFor("second");
     std::string motherName = getNameFor("Mother's");
     std::string fatherName = getNameFor("Father's");
}
```

### Definition
OK. We have seen an example, but what is the exact format of a function?

#### function3.cpp
```cpp
// A function definition is straightforward
<ReturnType>  <FunctionName>(<OptionalArgumentList>)
{
    <OptionalCode>
}

//  ReturnType:            This is the name of any type (built-in or user-defined)
//                         At the end of the function, you must have a statement
//                         that returns an object of this type.
//
//  FunctionName:          A unique name that identifies the function.
//
//  OptionalArgumentList:  This is either empty.
//                         Or a comma-separated list of parameters.
//                         Because C++ is strongly typed, each parameter is defined
//                         with both a type and a name.
//
//  OptionalCode:          We will be discussing this in more detail throughout
//                         these articles. But the new statement to learn is
//                         `return <Value>`. This is the value returned by the
//                         function to the original caller.
//
//  Value:                 Notice that above, I use the term `Value` and not object.
//                         A `Value` here can be an explicit object or the result
//                         of evaluating an expression (temporary object). Note
//                         One type of expression is a function call.
//
//                         return "An explicit String Object";
//
//                         return theResultOfAFunctionCall("Get A Result");
```

If a function has a `void` return type, you don't need to **Return Statement**. With any other return type, your function must exit using a **Return Statement**. The **Return Statement** determines the value returned to the caller from the function. The one exception to this rule (and there must be an exception to make it a rule) is `int main()`. If you don't explicitly have a **Return Statement** in `int main()`, the compiler will plant `return 0;` for you.


### Forward Declaration

One thing to note about a function is that you cannot use it before a declaration. We rewrite the original example above as follows:

#### function4.cpp
```cpp
#include <string>
#include <iostream>

int main(int argc, char* argv[])
{
     std::string firstName  = getNameFor("first");
     std::string secondName = getNameFor("second");
     std::string motherName = getNameFor("Mother's");
     std::string fatherName = getNameFor("Father's");
}

std::string getNameFor(std::string who)
{
    std::cout << "What is your " << who << " name?\n";
    std::string result;
    std::cin >> result;
    return result;
}
```

The only difference from above is that I have moved the `main()` function before the `getNameFor()` function. This will generate a compilation error as you use the `getNameFor()` function before a declaration. This may seem like a potential problem, but it is a deliberate technique that ensures you spell things correctly before use. In the above situation, the only change you need to make is a forward declaration. This allows you to declare a function before you define it. The utility of this will become apparent when we start defining modules.

#### function5.cpp
```cpp
#include <string>
#include <iostream>

// Add a forward declaration
extern std::string getNameFor(std::string who);

// A forward declaration is a function declaration without a body.
// Add an extern prefix and a semicolon on the end (the rest you should copy
// and paste from the function definition).
//
//
// Note: For the language lawyers who want to complain about the extern.
//       Hold your horses; we will learn the intricacies in due course.
//       This is only lesson 4.

int main(int argc, char* argv[])
{
     std::string firstName  = getNameFor("first");
     std::string secondName = getNameFor("second");
     std::string motherName = getNameFor("Mother's");
     std::string fatherName = getNameFor("Father's");
}

std::string getNameFor(std::string who)
{
    std::cout << "What is your " << who << " name?\n";
    std::string result;
    std::cin >> result;
    return result;
}
```

