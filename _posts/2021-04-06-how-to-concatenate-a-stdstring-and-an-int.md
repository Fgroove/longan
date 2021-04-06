---
title: Question list
author: Mista
date: 2021-04-06 23:32:00 +0800
categories: [Question]
tags: [String]
---


对于Java和Python来说，操作符`+`就足够了，但是C++不支持直接拼接字符串和数字。

## 字符串拼接数字

按字母顺序，

```c++
std::string name = "John";
int age = 21;
std::string result;

// 1. with Boost
result = name + boost::lexical_cast<std::string>(age);

// 2. with C++11
result = name + std::to_string(age);

// 3. with FastFormat.Format
fastformat::fmt(result, "{0}{1}", name, age);

// 4. with FastFormat.Write
fastformat::write(result, name, age);

// 5. with the {fmt} library
result = fmt::format("{}{}", name, age);

// 6. with IOStreams
std::stringstream sstm;
sstm << name << age;
result = sstm.str();

// 7. with itoa
char numstr[21]; // enough to hold all numbers up to 64-bits
result = name + itoa(age, numstr, 10);

// 8. with sprintf
char numstr[21]; // enough to hold all numbers up to 64-bits
sprintf(numstr, "%d", age);
result = name + numstr;

// 9. with STLSoft's integer_to_string
char numstr[21]; // enough to hold all numbers up to 64-bits
result = name + stlsoft::integer_to_string(numstr, 21, age);

// 10. with STLSoft's winstl::int_to_string()
result = name + winstl::int_to_string(age);

// 11. With Poco NumberFormatter
result = name + Poco::NumberFormatter().format(age);
```



1. is safe, but slow; requires [Boost](http://www.boost.org/) (header-only); most/all platforms
2. **is safe, requires C++11 ([to_string()](http://www.cplusplus.com/reference/string/to_string/) is already included in `#include <string>`)**
3. is safe, and fast; requires [FastFormat](http://fastformat.sourceforge.net/), which must be compiled; most/all platforms
4. (*ditto*)
5. is safe, and fast; requires [the {fmt} library](https://github.com/fmtlib/fmt), which can either be compiled or used in a header-only mode; most/all platforms
6. safe, slow, and verbose; requires `#include <sstream>` (from standard C++)
7. is brittle (you must supply a large enough buffer), fast, and verbose; itoa() is a non-standard extension, and not guaranteed to be available for all platforms
8. is brittle (you must supply a large enough buffer), fast, and verbose; requires nothing (is standard C++); all platforms
9. is brittle (you must supply a large enough buffer), [probably the fastest-possible conversion](http://www.ddj.com/cpp/184401596), verbose; requires [STLSoft](http://www.stlsoft.org/) (header-only); most/all platforms
10. safe-ish (you don't use more than one [int_to_string()](http://www.stlsoft.org/doc-1.9/int__to__string_8hpp.html) call in a single statement), fast; requires [STLSoft](http://www.stlsoft.org/) (header-only); **Windows-only**
11. is safe, but slow; requires [Poco C++](https://pocoproject.org/) ; most/all platforms



## Reference

[^Stack Overflow]: [How to concatenate a std::string and an int?](https://stackoverflow.com/questions/191757/how-to-concatenate-a-stdstring-and-an-int)

[chirpy-homepage]: https://github.com/cotes2020/jekyll-theme-chirpy/

