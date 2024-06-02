---
title: C++ Fun With Templates
author: Chan Jian Le
date: 2024-06-01 22:26:57 +0800
categories: [Programming]
tags: [programming, cpp]
---

> This article are notes on using template with C++ in my daily life.

## Fold Expression

### Example 1 (Debugging Prints)

Sometimes we may want to print debugging outputs,
but we may not know the size of the inputs. For example, printing the size of
a set of point clouds.

```cpp
#include <string>
#include <iostream>

int index = 0;

template <typename T>
std::string printCloud(const T& msg)
{
  return "cloud " + std::to_string(index--) + std::string{" size "} + std::to_string(msg) + std::string{", "};
}

// Base case
std::string printClouds()
{
  return {};
}

template <typename T, typename... Args>
std::string printClouds(const T& msg, const Args&... args)
{
  // increment index
  index++;
  return printCloud(msg) + printClouds(args...);
}

int main()
{
  std::cout << "result string: " << printClouds(100, 200, 12, 11) << "\n";
  std::cout << "result string: " << printClouds(55, 67, 77) << "\n";

  return EXIT_SUCCESS;
}
```

From the above code, it is more complicated as a base case is needed
for the recursive function to end. However, in c++17, we can make use of
the [fold expression](https://en.cppreference.com/w/cpp/language/fold)
to address this in a more consice manner without the need of a base case.

```cpp
#include <string>
#include <iostream>

template <typename... Args>
std::string printClouds(const Args&... args)
{
  // Get index
  auto index = sizeof...(args);

  return ([&, args]() {
    return "cloud " + std::to_string(index--) + std::string{" size "} +
      std::to_string(args) + std::string{", "};
  }() + ...);
}

int main()
{
  std::cout << "result string: " << printClouds(100, 200, 12, 11) << "\n";
  std::cout << "result string: " << printClouds(55, 67, 77) << "\n";

  return EXIT_SUCCESS;
}
```

### Code Analysis

Let's compare the snippet of code and its corresponding insights when
there are 7 inputs.

*Without Fold Expression Code*
```cpp
#include <string>
#include <iostream>

int index = 0;

template <typename T>
std::string printCloud(const T& msg)
{
  return "cloud " + std::to_string(index--)
  + std::string{" size "} + std::to_string(msg) + std::string{", "};
}

// Base case
std::string printClouds()
{
  return {};
}

template <typename T, typename... Args>
std::string printClouds(const T& msg, const Args&... args)
{
  // increment index
  index++;
  return printCloud(msg) + printClouds(args...);
}

int main()
{
  std::cout << "result string: " << printClouds(100, 200, 12, 11, 5, 6, 7) << "\n";
  return EXIT_SUCCESS;
}
```

*Without Fold Expression Insights*
```cpp
// INSIGHTS
#include <string>
#include <iostream>

int index = 0;

template<typename T>
std::basic_string<char> printCloud(const T & msg)
{
  return (std::operator+(std::operator+("cloud ", std::to_string(index--)), std::basic_string<char>{" size ", std::allocator<char>()}) + std::to_string(msg)) + std::basic_string<char>{", ", std::allocator<char>()};
}

/* First instantiated from: insights.cpp:23 */
#ifdef INSIGHTS_USE_TEMPLATE
template<>
std::basic_string<char> printCloud<int>(const int & msg)
{
  return std::operator+(std::operator+(std::operator+(std::operator+("cloud ", std::to_string(index--)), std::basic_string<char>(std::basic_string<char>(" size ", std::allocator<char>()))), std::to_string(msg)), std::basic_string<char>(std::basic_string<char>(", ", std::allocator<char>())));
}
#endif


std::basic_string<char> printClouds()
{
  return std::basic_string<char>{};
}

template<typename T, typename ... Args>
std::basic_string<char> printClouds(const T & msg, const Args &... args)
{
  index++;
  return printCloud(msg) + printClouds(args... );
}

/* First instantiated from: insights.cpp:28 */
#ifdef INSIGHTS_USE_TEMPLATE
template<>
std::basic_string<char> printClouds<int, int, int, int, int, int, int>(const int & msg, const int & __args1, const int & __args2, const int & __args3, const int & __args4, const int & __args5, const int & __args6)
{
  index++;
  return std::operator+(printCloud(msg), printClouds(__args1, __args2, __args3, __args4, __args5, __args6));
}
#endif


/* First instantiated from: insights.cpp:23 */
#ifdef INSIGHTS_USE_TEMPLATE
template<>
std::basic_string<char> printClouds<int, int, int, int, int, int>(const int & msg, const int & __args1, const int & __args2, const int & __args3, const int & __args4, const int & __args5)
{
  index++;
  return std::operator+(printCloud(msg), printClouds(__args1, __args2, __args3, __args4, __args5));
}
#endif


/* First instantiated from: insights.cpp:23 */
#ifdef INSIGHTS_USE_TEMPLATE
template<>
std::basic_string<char> printClouds<int, int, int, int, int>(const int & msg, const int & __args1, const int & __args2, const int & __args3, const int & __args4)
{
  index++;
  return std::operator+(printCloud(msg), printClouds(__args1, __args2, __args3, __args4));
}
#endif


/* First instantiated from: insights.cpp:23 */
#ifdef INSIGHTS_USE_TEMPLATE
template<>
std::basic_string<char> printClouds<int, int, int, int>(const int & msg, const int & __args1, const int & __args2, const int & __args3)
{
  index++;
  return std::operator+(printCloud(msg), printClouds(__args1, __args2, __args3));
}
#endif


/* First instantiated from: insights.cpp:23 */
#ifdef INSIGHTS_USE_TEMPLATE
template<>
std::basic_string<char> printClouds<int, int, int>(const int & msg, const int & __args1, const int & __args2)
{
  index++;
  return std::operator+(printCloud(msg), printClouds(__args1, __args2));
}
#endif


/* First instantiated from: insights.cpp:23 */
#ifdef INSIGHTS_USE_TEMPLATE
template<>
std::basic_string<char> printClouds<int, int>(const int & msg, const int & __args1)
{
  index++;
  return std::operator+(printCloud(msg), printClouds(__args1));
}
#endif


/* First instantiated from: insights.cpp:23 */
#ifdef INSIGHTS_USE_TEMPLATE
template<>
std::basic_string<char> printClouds<int>(const int & msg)
{
  index++;
  return std::operator+(printCloud(msg), printClouds());
}
#endif


int main()
{
  std::operator<<(std::operator<<(std::operator<<(std::cout, "result string: "), printClouds(100, 200, 12, 11, 5, 6, 7)), "\n");
  return 0;
}
```

*With Fold Expression Code*
```cpp
// RAW CODE
#include <string>
#include <iostream>

template <typename... Args>
std::string printClouds(const Args&... args)
{
  // increment index
  auto index = sizeof...(args);

  auto printCloud = [&](auto& msg) {
    return "cloud " + std::to_string(index--) + std::string{" size "} +
      std::to_string(msg) + std::string{", "};
  };
  return (... + printCloud(args));
}

int main()
{
  std::cout << "result string: " << printClouds(100, 200, 12, 11, 5, 6, 7) << "\n";

  return EXIT_SUCCESS;
}
```

*With Fold Expression Insights*
```cpp
// INSIGHTS
#include <string>
#include <iostream>

template<typename ... Args>
std::basic_string<char> printClouds(const Args &... args)
{
  unsigned long index = sizeof...(args);
    
  class __lambda_10_21
  {
    public: 
    template<class type_parameter_1_0>
    inline auto operator()(type_parameter_1_0 & msg) const
    {
      return (std::operator+(std::operator+("cloud ", std::to_string(index--)), std::basic_string<char>{" size ", std::allocator<char>()}) + std::to_string(msg)) + std::basic_string<char>{", ", std::allocator<char>()};
    }
    
  };
  
  auto printCloud = __lambda_10_21{};
  return (... + printCloud(args));
}

/* First instantiated from: insights.cpp:19 */
#ifdef INSIGHTS_USE_TEMPLATE
template<>
std::basic_string<char> printClouds<int, int, int, int, int, int, int>(const int & __args0, const int & __args1, const int & __args2, const int & __args3, const int & __args4, const int & __args5, const int & __args6)
{
  unsigned long index = 7;
    
  class __lambda_10_21
  {
    public: 
    template<class type_parameter_0_0>
    inline /*constexpr */ auto operator()(type_parameter_0_0 & msg) const
    {
      return (std::operator+(std::operator+("cloud ", std::to_string(index--)), std::basic_string<char>(std::basic_string<char>(" size ", std::allocator<char>()))) + std::to_string(msg)) + std::basic_string<char>(std::basic_string<char>(", ", std::allocator<char>()));
    }
    
    #ifdef INSIGHTS_USE_TEMPLATE
    template<>
    inline /*constexpr */ std::basic_string<char> operator()<const int>(const int & msg) const
    {
      return std::operator+(std::operator+(std::operator+(std::operator+("cloud ", std::to_string(index--)), std::basic_string<char>(std::basic_string<char>(" size ", std::allocator<char>()))), std::to_string(msg)), std::basic_string<char>(std::basic_string<char>(", ", std::allocator<char>())));
    }
    #endif
    
    private: 
    unsigned long & index;
    
    public:
    __lambda_10_21(unsigned long & _index)
    : index{_index}
    {}
    
  };
  
  __lambda_10_21 printCloud = __lambda_10_21{index};
  return std::operator+(std::operator+(std::operator+(std::operator+(std::operator+(std::operator+(printCloud.operator()(__args0), printCloud.operator()(__args1)), printCloud.operator()(__args2)), printCloud.operator()(__args3)), printCloud.operator()(__args4)), printCloud.operator()(__args5)), printCloud.operator()(__args6));
}
#endif


int main()
{
  std::operator<<(std::operator<<(std::operator<<(std::cout, "result string: "), printClouds(100, 200, 12, 11, 5, 6, 7)), "\n");
  return 0;
}
```

### Conclusion

From the code insights, fold expressions will only generate the relavant code
for the printing. On the other hand, template without fold expression will
generate all templates for the recursive used. 7 -> 6 -> 5 -> 4 -> 3 -> 2 -> 1 -> 0.

Using `x86-64 gcc 14.1` to analysis the assembly code, only
[423](https://godbolt.org/z/rs69WdTPM) lines of
instructions where generated for the fold expersion version while the one without
has [620](https://godbolt.org/z/9E8GEqn71).

We can conclude that fold expression method is easier to read and
much more consice.

## References

- https://www.youtube.com/watch?v=v5tLFRfktWA
