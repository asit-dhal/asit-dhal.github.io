---
layout: post
title: RAII in C++
tags: [Cpp, DesignPatten]
---

RAII is one of the patterns in C++ to manage resources.

Sometimes at some part of the program, we need to allocate some memory and later free it.

```cpp
template <typename T, std::size_t N>
T* allocateMemory()
{
    return new T[N];
}
template <typename T>
void freeMemory(T *p)
{
    delete[] p;
}
int main()
{
    int *p = allocateMemory<int, 10>();
    // use p
    freeMemory(p);
    
    return 0;
}
```

Disaster can happen when

- `allocateMemory()` is not called at all. Then the program will crash.
- `freeMemory()` is not called at all. There will be memory leak.
- `freeMemory()` is called twice from two different control paths or threads. It will cause double deletion and might crash.

This is a problem in C++, because resource management is developer’s responsibility. RAII or _Resource Acquisition is Initialization_ is a pattern which fixes this problem.

In RAII

- Resource is a class invariant.
- Resource acquisition(memory allocation, file opening, acquiring mutex , database connection) is done when lifetime of the object begins. In C++11, lifetime of object begins after construction.
- Resource is available throughout the object lifetime.
- Resource is released when lifetime of the object object ends.

So RAII can be summarized as

- Constructor acquires the resource and establishes all class invariant or throws an exception if that cannot be done.
- Destructor releases the resource and never throws an exception.

Resource is always accessed through RAII object and

- has automatic storage duration or temporary lifetime itself
- has lifetime that is bounded by the lifetime of an automatic or temporary object

If something goes wrong, destructor will always be called. So resources are freed.

If resources can be moved(like heap memory), then RAII objects should support move operations to transfer resources. If not(resources like mutex, socket handle, file handle), then move operations should be forbidden.

![RAII]({{ "/assets/img/pexels/RAII.png" | relative_url}})


## Implementation

We want to design an RAII class for our trivial file operation. Our RAII class will do the followings

1. open the file in the constructor
2. close the file in the destructor
3. It has a method which takes a string to write into the file.

```cpp
class FileHandlerRAII
{
public:
    explicit FileHandlerRAII(const char* fileName)
        : m_fp(std::fopen(fileName, "w+"))
    {
        std::cout << "file resource: " << fileName
                  << " acquired" << std::endl;
    }
    
    bool write(const char *str)
    {
        if (std::fputs(str, m_fp) == EOF) {
            return false;
        }
        return true;
    }
    
    ~FileHandlerRAII()
    {
        std::fclose(m_fp);
        std::cout << " file resource released" << std::endl;
    }
    
private:
    FILE *m_fp;
};
```

It can happen that, file opening fails. C++ contructor can’t return anything. The caller needs to know the status. So we will throw an exception.

```cpp
class BadFile: virtual public std::exception
{
public:
    explicit BadFile(const std::string &msg): m_msg(msg) {}
    virtual ~BadFile() throw () {}
    virtual const char* what() const throw()
    {
        return m_msg.c_str();
    }
    
private:
    std::string m_msg;
};
```

We will throw an exception if file opening fails. Similarly, we will close the file only when file is opened.

```cpp
class FileHandlerRAII
{
public:
    explicit FileHandlerRAII(const char* fileName)
        : m_fp(std::fopen(fileName, "w+"))
    {
        if (m_fp == NULL) {
            throw BadFile(std::string(fileName) +
                          " failed to be opened.");
        }
        std::cout << "file resource: " << fileName 
                  << " acquired" << std::endl;
    }
    
    //....    
    ~FileHandlerRAII()
    {
        if (m_fp != NULL) {
            std::fclose(m_fp);
            std::cout << "file resource released" << std::endl;
        }
    }
    
private:
    FILE *m_fp;
};
```

Our RAII class should not be moved or copied like trivial classes. So we will delete all move and copy member functions.

```cpp
FileHandlerRAII(const FileHandlerRAII &) = delete;
FileHandlerRAII(FileHandlerRAII &&) = delete;
    
FileHandlerRAII& operator=(const FileHandlerRAII &) = delete;
FileHandlerRAII& operator=(FileHandlerRAII &&) = delete;
```

Let's try to use our `FileHandlerRAII` class.

```cpp
int main()
{
    FileHandlerRAII raii("test.log");
    if (raii.write("Hello RAII")) {
        std::cout << "file accessed successfully" << std::endl;
    }
    return 0;
}
```

Output

```
file resource: test.log acquired
file accessed successfully
file resource released
```

In the above example we don’t call the destructor at all.

Even if something terribly goes wrong, our resource will be released.

```cpp
int main()
{
    try {
        FileHandlerRAII raii("test.log");
        
        for (auto i=0; i < 3; i++) {
            if (raii.write("Hello RAII")) {
                std::cout << "file accessed successfully"
                          << std::endl;
            }
            
            if (i == 1) {
                throw std::runtime_error("Bye bye cruel world !");
            }
        }
    } catch (...) {
        std::cout << "something went wrong" << std::endl;
    }
    
    return 0;
}
```

```
file accessed successfully
file accessed successfully
file resource released
something went wrong
```cpp

We can also provide multi threading support by using a mutex during construction and destruction.

The code is publicly available here

[RAII.cpp](https://gist.github.com/asit-dhal/ac2f11948ca24e7130685a3fc8ae29aa "RAII.cpp")

Thanks for reading !!
