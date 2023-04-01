---
permalink: /rendering/stratusgfx/coding_stanard
---

For StratusGFX this is the C++ coding standard that I've adopted over time. They're not extremely strict but the goal is to try and keep things consistent and readable while helping avoid bugs and performance issues.

## General Guidelines

-> Classes, structs and functions start with a capital letter

-> Variables start with lower case letters and use camelCase

-> Private functions and variables end with underscore _

{% highlight c++ %}
void PrivateMemberFunction_();
int privateMemberVariable_;
{% endhighlight %}

-> Preprocessor definitions are fully capitalized with words separated by underscore _

{% highlight c++ %}
#define MACRO_NAME expression
{% endhighlight %}

## Pointers

-> Always prefer std::unique_ptr and std::shared_ptr for automatic memory management

-> If `new` is used, adhere to RAII principles as much as possible to avoid memory leaks

## Use of Const

-> When possible mark variables as const to signal their data won't change

-> If a class function is not supposed to modify object state, mark it const

-> Always prefer passing by const reference to avoid unnecessary data copying

## Use of Standard Library

-> Usage of C++ standard containers and algorithms is preferred whenever possible since they are well-tested and have generally good performance

-> `assert` and `static_assert` are great to prevent us from making a change that breaks a fundamental requirement