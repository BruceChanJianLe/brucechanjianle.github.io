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

Let's compare the snippet of code and its corresponding insights.

*Without Fold Expression*
<table>
  <tr>
    <th>Code</th>
    <th>Insight</th>
  </tr>
  <tr>
    <td>
      <pre><code class="language-cpp">
#include &lt;string&gt;
#include &lt;iostream&gt;

int index = 0;

template &lt;typename T&gt;
std::string printCloud(const T& msg)
{
  return "cloud " + std::to_string(index--) + std::string{" size "} + std::to_string(msg) + std::string{", "};
}

// Base case
std::string printClouds()
{
  return {};
}

template &lt;typename T, typename... Args&gt;
std::string printClouds(const T& msg, const Args&... args)
{
  // increment index
  index++;
  return printCloud(msg) + printClouds(args...);
}

int main()
{
  std::cout &lt;&lt; "result string: " &lt;&lt; printClouds(100, 200, 12, 11, 5, 6, 7) &lt;&lt; "\n";
  return EXIT_SUCCESS;
}</code></pre>
    </td>
    <td>
      <pre><code class="language-cpp">
      int main
</code></pre>
    </td>
  </tr>
</table>


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
