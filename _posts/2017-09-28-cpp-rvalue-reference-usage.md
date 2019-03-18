---
layout: post
title:  "C++11中的右值引用用法小结"
categories: c++
tags: c++11 
author: ShuxiaoW
---

* content
{:toc}

本文记录使用C++11中的右值引用的一些tips，包括何时该用右值引用。

## 1 关于右值引用

TODO

## 2 关于移动语义

TODO

## 3 关于完美转发

TODO

## 4 如何使用右值引用

以下节选自[Google C++ Style Guild](https://google.github.io/styleguide/cppguide.html#Rvalue_references):

> Use rvalue references only to define move constructors and move assignment operators, or for perfect forwarding.

只在3种情况下使用右值引用：定义移动构造函数；定义移动赋值操作符；实现完美转发

## 5 参考资料

[1] [C++ Rvalue References Explained](http://thbecker.net/articles/rvalue_references/section_01.html)

[2] [Google C++ Style Guild](https://google.github.io/styleguide/cppguide.html#Rvalue_references)

[3] C++ Primer 5th
