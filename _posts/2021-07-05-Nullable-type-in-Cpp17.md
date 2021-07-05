---
layout: post
title: Nullable Type in C++17
---
Sometimes we create an object which contains no valid data. We often a associate a flag or a special value, which decides the validity of the data. For example,

1. a pointer with value nullptr is a pointer that points to no location.
2. an id type with an allowed value of 0, which says the id is invalid if 0 or it’s a valid if more than 0.

```cpp
unsigned int getId(const std::string &name)
{
    if (name == "test") {
        return 1;
    }
    
    if (name == "foo") {
        return 2;
    }
    
    return 0;
}
int main() 
{
    std::vector<std::string> names {"test", "foo", "bar" };
    
    for (auto const &e: names) {
        if (auto id = getId(e); id != 0) {
            std::cout << e << ": " << id << std::endl;
        } else {
            std::cout << e << " is not registered in the system"
                      << std::endl;
        }
    }
    
    return 0;
}
```

The problem is we need to know what is that special value that represents absence of value. For a type representing ID of type int, 0 is the special value. For pointers, nullptr is a special value. Special value depends on the underlying type. This is cumbersome to maintain when the application has 100s of types.

A nullable value type can be represented as a wrapper of an existing type and a method which says whether the instance has a valid value. In C++17, std::optional was introduced to represent nullable value type. std::optional objects usually have the contained object and a boolean flag.

1. If it contains std::nullopt, it means it has no value. When it has no value, the value is in uninitialized state.
2. It supports both copy semantics and move semantics. The semantics depends on the underlying type.
3. It has a small memory overhead for the boolean flag plus alignment.
4. A few functions to access the state of the object and contained object.

The above functions can be written as follows using std::optional.

```cpp
std::optional<unsigned int> getId(const std::string &name)
{
    if (name == "test") {
        return 1;
    }
    
    if (name == "foo") {
        return 2;
    }
    
    return std::nullopt;
}
int main() 
{
    std::vector<std::string> names {"test", "foo", "bar" };
    
    for (auto const &e: names) {
        if (auto id = getId(e); id) {
            std::cout << e << ": " << id.value() << std::endl;
        } else {
            std::cout << e << " is not registered in the system"
                      << std::endl;
        }
    }
    
    return 0;
}
```

Instead of returning 0 as a special value, we return std::nullopt. In the calling site, we don’t have to worry about what is the special value.

### Construction

We can create std::optional object which has no value at all. The contained object won’t be constructed at all.

```cpp
class Val
{
public:
    Val()
    {
        std::cout << "default constructor" << std::endl;
    }
    
    ~Val()
    {
        std::cout << "destructor" << std::endl;
    }
};
int main() 
{
    std::optional<Val> opt1;
    std::optional<Val> opt2(std::nullopt);
    
    return 0;
}
```

Here, the constructor of contained object is not called. When the std::optional instance is created, a byte array of size and alignment enough to hold the object is allocated. When it needs to construct the contained object, then placement new is used to create the object at the same memory location. For one std::optional instance, there can be many contained objects in a lifetime, but memory is always fixed.

We can initialize with some value. Because of C++17’s type deduction rules, type does not need to be specified.

```cpp
std::optional opt1{ 10 }; // std::optional<int>
std::optional opt2{ 'a' }; // std::optional<char>
std::optional opt3{ std::make_tuple(10, "Value") };
```

Here, an unnecessary temporary will be created. We can use `std::in_place` to construct the object in place and avoid temporary. In place construction will need the type to be explicitly specified.

```cpp
std::optional<std::tuple<int, std::string>> opt3 { std::in_place, 10, "Value" };
```

std::optional has a constructor, which takes universal reference. This allows in place construction avoiding creation of unnecessary temporaries.
There is an utility function, `std::make_optional`, can also be used to do the in place construction with `std::in_place` tag.

```cpp
std::optional<std::tuple<int, std::string>> optInPlace{ std::in_place, 10, "Value" };
std::optional optUtility = std::make_optional<std::tuple<int, std::string>>{ 10, "Value" };
```

In place construction is useful, if a type is neither copy-able nor movable.

```cpp
class Val
{
public:
    Val(int a, const std::string &b): m_a(a), m_b(b) {}
    Val(const Val &) = delete;
    Val(Val &&) = delete;
    Val& operator=(const Val &) = delete;
    Val& operator=(Val &&) = delete;
    ~Val() = default;
    
private:
    int m_a;
    std::string m_b;
};
int main() 
{
    // std::optional<Val> opt{ Val(5, "Value") }; // error if uncommented
    std::optional<Val> opt{ std::in_place, 5, "Value" };
    return 0;
}
```

std::optional can also be copy or move constructed from other std::optional objects.

### Accessing the contained value

std::optional provides conversion to bool operator and `has_value()` method to check if it has a value.

```cpp
int main() 
{
    std::optional<int> opt1{ 10 };
    std::optional<int> opt2;
    
    if (opt1) {
        std::cout << "opt1 has a value" << std::endl;
    } else {
        std::cout << "opt1 has no value" << std::endl;
    }
    
    if (opt2.has_value()) {
        std::cout << "opt2 has a value" << std::endl;
    } else {
        std::cout << "opt2 has no value" << std::endl;
    }
    
    return 0;
}
```

std::optional provides * and -> overloaded operator to access the contained object. If it has no value, it will have an undefined behavior. It also provides value() method which throws `std::bad_optional_access` exception in case it has no value.

```cpp
int main()
{
    std::optional<int> opt1{ 10 };
    if (opt1) {
        std::cout << "value from (*): " << *opt1 << std::endl;
        std::cout << "value from value(): "
                  << opt1.value() << std::endl;
    }
    
    std::optional<std::pair<int, std::string>> opt2
                           { std::in_place, 10, "test" }; 
    if (opt2) {
        std::cout << "value from (->): ("
                  << opt2->first << ", " 
                  << opt2->second << ")" << std::endl;
    }
    
    std::optional<int> opt3{ std::nullopt };
    try {
        std::cout << opt3.value();
    } catch (const std::bad_optional_access &e) {
        std::cout << e.what() << std::endl;
    }
    
    return 0;
}
```

output

```
value from (*): 10
value from value()): 10
value from (->): (10, test)
bad optional access
```

There is a method `value_or(param..)`, which has a fallback mechanism if the optional object contains no value.

```cpp
int main()
{
    std::optional<int> opt1{ 10 };
    std::cout << "value_or: " << opt1.value_or(0) << std::endl;
    
    std::optional<int> opt2{ std::nullopt };
    std::cout << "value_or: " << opt2.value_or(0) << std::endl;
    
    return 0;
}
```

output
```
value_or: 10
value_or: 0
```

operator * and value() returns object by reference. So, you should avoid it using on temporaries.
`value_or()` returns by value. So, you should avoid it using on objects where copy is an expensive operation.


### Modifying the value
std::optional provides emplace() method assign a new value to the contained type. emplace() takes a universal reference. So, if a temporary is given, it will move construct the contained object.
It also provides reset() to destroy the underlying object.

```cpp
class Value
{
public:
    Value(int a, int b): m_a(a), m_b(b)
    {
        std::cout << "constructed" << std::endl;
    }
    
    Value(const Value &others) : m_a(others.m_a), m_b(others.m_b)
    {
        std::cout << "copy constructed" << std::endl;
    }
    
    Value(Value &&v) : m_a(std::move(v.m_a)), m_b(std::move(v.m_b))
    {
        std::cout << "move constructed" << std::endl;
    }
    
    Value& operator=(const Value &other)
    {
        if (this == &other) {
            return *this;
        }
        
        m_a = other.m_a;
        m_b = other.m_b;
        std::cout << "copy assigned" << std::endl;
        
        return *this;
    }
    
    Value& operator=(Value &&other)
    {
        if (this == &other) {
            return *this;
        }
        
        m_a = std::move(other.m_a);
        m_b = std::move(other.m_b);
        std::cout << "move assigned" << std::endl;
        
        return *this;
    }
    
    ~Value()
    {
        std::cout << "destructed" << std::endl;
    }
    
    int getA() const
    {
        return m_a;
    }
private:
    int m_a;
    int m_b;
};
int main() 
{
    std::optional<Value> opt{ std::in_place, 10, 20 };
    if (opt.has_value()) {
        std::cout << "value: " << opt->getA()<< std::endl;
        std::cout << "changing value: " << std::endl;
        opt.emplace(Value{ 100, 200 });
        std::cout << "value: " << opt->getA() << std::endl;
    }
    
    opt.reset();
    if (!opt.has_value()) {
        std::cout << "has no value" << std::endl;
    }
    
    return 0;
}
```

Output

```
constructed
value: 10
changing value
constructed
destructed
move constructed
destructed
value: 100
destructed
has no value
```

### Comparison

* If both optional objects have value, then the contained objects are compared. If the contained objects can’t be compared, the compiler will give error.
* If both optional objects have no value, then they are equal.
* A comparison between two objects where one has value and other has no value, the object with a value will always be greater.

```cpp
int main() 
{
    std::optional<int> int1{10};
    std::optional<int> int2{20};
    std::cout << std::boolalpha 
              << (int1 < int2) << std::endl; // true
    
    std::optional<int> nullInt1;
    std::optional<int> nullInt2;
    std::cout << std::boolalpha 
              << (nullInt1 == nullInt2) << std::endl; // true
    
    std::cout << std::boolalpha
              << (nullInt1 < int1) << std::endl; // true
    
    return 0;
}
```
