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
  std::cout << "result string: " << printClouds(100, 200, 12, 11) << "\n";
  std::cout << "result string: " << printClouds(55, 67, 77) << "\n";

  return EXIT_SUCCESS;
}
```

Comparisons

<div style="display: flex; justify-content: space-around;">
    <div style="flex-basis: 45%;">
        <pre><code class="language-cpp">
auto printCloud = [&](auto& msg)
{
  return "cloud " + std::to_string(index--) + std::string{" size "}
    + std::to_string(msg) + std::string{", "};
};
        </code></pre>
    </div>
    <div style="flex-basis: 45%;">
        <pre><code class="language-cpp">
auto printCloud = [&](auto& msg) {
    return "cloud " + std::to_string(index--) + std::string{" size "} +
           std::to_string(msg) + std::string{", "};
};
        </code></pre>
    </div>
</div>


## References

- https://www.youtube.com/watch?v=v5tLFRfktWA
