---
layout: post
title: Structured Binding Declaration
tags: [Cpp, BackToBasics]
---
Structured binding allows you to initialize multiple variables with individual elements of a structure, tuple, or array.

Often, a function will return multiple values packed in a structure. In good old C++ you need to assign it to a struct variable and access the individual members from there. In pre-C++17, however, you need to assign the return value to a struct variable and access the individual elements (as shown below). This article aims to demonstrate how structured binding allows you to initialize multiple variables with individual elements of a structure, tuple, or array.

```cpp
struct RetVal 
{
    int a;
    std::string b;
};

RetVal doSomething()
{
    //do something useful
    return {10, "Value"};
}

int main()
{
    auto retVal = doSomething();
    assert(retVal.a == 10);
    assert(retVal.b == "Value");
    
    return 0;
}
```

In C++17, you can use structured binding to access the individual elements, as shown below:


```cpp
struct RetVal 
{
    int a;
    std::string b;
};

RetVal doSomething()
{
    //do something useful
    return {10, "Value"};
}

int main()
{
    auto [x, y] = doSomething();
    assert(x == 10);
    assert(y == "Value");
    
    return 0;
}
```

Under the hood, an anonymous variable with two members (x and y) is created. The type is deduced from the element of the structure. As long as the anonymous variable exists, we can access x and y(current scope). We can’t declare another variable with x or y.

The general syntax is:

```cpp
attr(optional) cv-auto ref-operator(optional) [ identifie-list ] = expression; [1]

attr(optional) cv-auto ref-operator(optional) [ identifie-list ] { expression } [2]

attr(optional) cv-auto ref-operator(optional) [ identifie-list ] ( expression ) [1]

```

`attr` can be any attribute.
`cv-auto` is cv- qualified type specifier auto.
`ref-operator` is `&` or `&&`.

identifier-list is a comma-separated list of variables.

It works with:

- struct
- tuple and pair
- array

### Struct: Data Member Binding

For a struct, all non-static members should be directly accessible. It doesn’t work if any of the members are private or protected.

```cpp
struct A {
    A(int w, int x, int y, int z) : w(w), x(x), y(y), z(z){}
    int w;
    int x;
    int y;
private:
    int z;
};

int main()
{
    A a(1, 2, 3, 4);
    auto [m1, m2, m3] = a;
    return 0;
}
```

In this case, the compiler will give an error:

```
main.cpp: In function ‘int main()’:
main.cpp:20:10: error: cannot decompose inaccessible member ‘A::z’ of ‘A’
     auto [m1, m2, m3] = a;
          ^~~~~~~~~~~~
main.cpp:13:9: note: declared private here
     int z;
```

If the struct has a base class, then all non-static members should either be in the base class or in the child class.

```cpp
struct A {
    A(int w, int x, int y) : w(w), x(x), y(y){}
    int w;
    int x;
    int y;
};

struct B : public A {
    B(int w, int x, int y, int z):A(w, x, y),z(z) {}
    int z;
};

int main()
{
    B b(1, 2, 3, 4);
    auto [m1, m2, m3, m4] = b;
    return 0;
}
```

In this case, the compiler will give an error.

```
main.cpp: In function ‘int main()’:
main.cpp:23:10: error: cannot decompose class type ‘B’: both it and its base class ‘A’ have non-static data members
     auto [m1, m2, m3, m4] = b;
          ^~~~~~~~~~~~~~~~
```

To move forward, empty either the base class or the child class. There are better ways to do this (such as by providing a tuple API) but I will write about this another day.

```cpp
struct A1 {
    A1(int w, int x, int y) : w(w), x(x), y(y){}
    int w;
    int x;
    int y;
};

struct B1 : public A1 {
    B1(int w, int x, int y) : A1(w, x, y) {}
};

// or
struct A2 {
    void emptyClass() {}
};

struct B2 : public A2 {
    B2(int w, int x, int y) : w(w), x(x), y(y){}
    int w;
    int x;
    int y;
};

int main()
{
    B1 b1(1, 2, 3);
    auto [m1, m2, m3] = b1;
    
    assert(b1.w == m1);
    assert(b1.x == m2);
    assert(b1.y == m3);
    
    B2 b2(10, 20, 30);
    auto [e1, e2, e3] = b2;
    
    assert(b2.w == e1);
    assert(b2.x == e2);
    assert(b2.y == e3);
    
    return 0;
}
```

## Tuple like Type

In order to unpack a tuple or pair, in pre C++17, either you need to use `std::tie()` or `std::get<N>`.

```cpp
int main()
{
    auto t = std::make_tuple(10, 'a', "Value");
    
    // method-1
    int t1;
    char t2;
    std::string t3;
    std::tie(t1, t2, t3) = t;
    assert(t1 == 10);
    assert(t2 == 'a');
    assert(t3 == "Value");
    
    // method-2
    auto e1 = std::get<0>(t);
    auto e2 = std::get<1>(t);
    auto e3 = std::get<2>(t);
    assert(e1 == 10);
    assert(e2 == 'a');
    assert(e3 == "Value");
    
    return 0;
}
```

You can use structured binding to simplify it.

```cpp
int main()
{
    auto t = std::make_tuple(10, 'a', "Value");
    
    auto [t1, t2, t3] = t;
    assert(t1 == 10);
    assert(t2 == 'a');
    assert(t3 == "Value");
    
    return 0;
}
```

`std::tie` can be used to ignore certain values, which is not possible in structured binding. You always have to declare the variable, don’t use it if you don’t need to! In this case, the compiler will emit an unused variable warning.

```cpp
int main ()
{
    auto t = std::make_tuple (10, 'a', "Value");

    int t1;
    std::string t3;
    std::tie(t1, std::ignore, t3) = t;
    assert (t1 == 10);
    assert (t3 == "Value");

    auto [e1, unsed, e3] = t;
    (void)unsed;
    assert (e1 == 10);
    assert (e3 == "Value");
  
    return 0;
}
```

You can const qualify the bindings. If you want to avoid making a copy, you can make a reference.

```cpp
int main ()
{
    auto t = std::make_tuple (10, 'a', "Value");

    auto& [t1, t2, t3] = t;
    (void)t3;
    t1 = 20;
    t2 = 'z';
    assert (t1 == std::get<0>(t));
    assert (t2 == std::get<1>(t));
  
    return 0;
}
```

## Array-like Type

In case of an array, structured binding also works pretty well. The number of identifiers in structured binding should be the same as the number of array elements.

```cpp
int main ()
{
    std::array<int, 3> cppArr{1, 2,3};
    
    auto [e1, e2, e3] = cppArr;
    assert(e1 == 1);
    assert(e2 == 2);
    assert(e3 == 3);
    
    int cArr[] = {10, 20, 30};
    auto [v1, v2, v3] = cArr;
    assert(v1 == 10);
    assert(v2 == 20);
    assert(v3 == 30);
    
    return 0;
}
```

## cv-auto and Ref-operator
You can qualify the underlying anonymous variable with `const` or `volatile`. You can use `&` or `&&` to make it an lvalue or an rvalue reference. This creates an advantage when a function returns by value. You also want to avoid any unnecessary copy. You can either bind `const auto&` or `auto&&`. The temporary will be available because of lifetime extension. Remember:

> “Const” or “reference” applies to the underlying anonymous variable, so individual elements can’t have explicit qualifiers.The bindings are available in the current scope.

```cpp
struct RetVal
{
    int a;
    std::string b;
};

RetVal doSomething()
{
    // do something
    return {10, "Test"};
}


int main ()
{
    auto&& [x, y] = doSomething();
    assert(x == 10);
    assert(y == "Test");
    
    const auto& [m, n] = doSomething();
    assert(m == 10);
    assert(n == "Test");
  
    return 0;
}
```

The type of individual identifier depends upon the type of individual members of struct or tuple.

```cpp
struct RetVal
{
    int x;
    int& y;
    const int& z;
};

RetVal doSomething(int& a, const int &b) {
    return {10, a, b};
}

int main()
{
    int a = 100;
    auto [m, n, o] = doSomething(a, 200); 

    assert(m == 10);
    assert(n == 100);
    assert(o == 200);
    
    assert((std::is_same_v<decltype(m), int>));
    assert((std::is_same_v<decltype(n), int&>));
    assert((std::is_same_v<decltype(o), const int&>));
    
    return 0;
}
```

In the above example, `m` is deduced as `int`, `n` as `int&` and `o` as `const int&`.

## Expressive Code

Since pair also supports structured binding, we can make the range-based for loop more expressive.

```cpp
int main ()
{
    std::map<std::string, std::string> countryCapitals {
        {"India", "New Delhi"},
        {"Germany", "Berlin"},
        {"USA", "Newyork"}
    };
    
    for (auto const& [country, capital] : countryCapitals) {
        std::cout << country << " => " << capital << std::endl;
    }
    
    return 0;
}
```

If any function returns a status and some useful value, structural binding can be written to write more expressive code.

```cpp
int main ()
{
    std::map<std::string, std::string> countryCapitals {
        {"India", "New Delhi"},
        {"Germany", "Berlin"},
        {"USA", "Newyork"}
    };
    
    if (auto const [itr, success] = countryCapitals.insert({"USA", "Moscow"}); success) {
        std::cout << "insert successful for country: " << itr->first << " capital: " << itr->second << std::endl;
    } else {
        std::cout << "country: " << itr->first << " already has a capital: " << itr->second << std::endl;
    }
    
    return 0;
}
```

## Conclusion

This article aims to demonstrate how structured binding allows you to initialize multiple variables with individual elements of a structure, tuple, or array. I hope you found it helpful. Thanks for reading.

