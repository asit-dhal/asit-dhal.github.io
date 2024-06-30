---
layout: post
title: Variables in CMake
tags: [CMake, BackToBasics]
---

Like any programming language, CMake provides a mechanism to store data or state during computation. CMake provides two types of variables.

- __Normal Variable__: These are created when CMake runs and are scoped to the current CMake script or function. Normal variables are typically used for temporary storage of data or computation results.
- __Cache Variable__: These are persistent variables that are stored in the file system and can be accessed across multiple CMake runs.

## Normal variable
It provides the command set to create or remove a variable. The syntax of set command is as follows.

```
set(<variable> <value>... [PARENT_SCOPE])
```

This creates a normal variable in the current scope. The variable name can be any alpha numeric string and is case sensitive. However, it also allows `+`, `-`, and other things, don’t use those special things.

In CMake, variables can be referenced using the `${varName}` syntax. This syntax is used to substitute the value of the variable at the point of reference. Additionally, CMake supports variable expansion, which allows you to use the value of one variable as part of the name of another variable. The syntax for this is `${varname${part}}`. In this case, the inner variable `${part}` is expanded first, and its value is used as a part of the name of the outer variable `${varname}`. The resulting variable name is then used for substitution.

```cmake
set(prefix "APP")
set(property_name "PATH")
set(${prefix}_GEN_${property_name} "project_root/gendir") # var name is APP_GEN_PATH
set(${prefix}_BUILD_${property_name} "project_root/bindir") # var name is APP_BUILD_PATH
message("generation path: ${${prefix}_GEN_${property_name}}") # generation path: project_root/gendir
message("build path: ${${prefix}_BUILD_${property_name}}") # build path: project_root/bindir
```

If a single value is provided, it is considered as string. If multiple value is provided, then it is considered as a list. If the string contains space, then the value should be quoted to avoid being interpreted as a list. If the values are separated by `;`, then it will be considered as a list.

```cmake
set(val1 Value1)
message("val1: ${val1}") # val1: Value1
set(val2 "Value 2")
message("val2: ${val2}") # val2: Value 2

set(list_value1 one two three)
message("list_value1: ${list_value1}") # list_value1: one;two;three

set(list_value2 "four;five;six")
message("list_value2: ${list_value2}") # list_value2: four;five;six
```

`if(DEFINED varName)` can be used to check if a variable is defined.

```cmake
# val1 is defined
set(val1 Value)

if(DEFINED val1)
    message("val1: ${val1}") # val1: Value
else()
    message("val1 is not defined")
endif()

if(DEFINED val2)
    message("val2: ${val2}")
else()
    message("val2 is not defined") # val2 is not defined
endif()
```
A variable definition can be removed either with `unset` command or `set(var)` command without a value.

```
unset(<variable> [CACHE | PARENT_SCOPE])
```

`unset` command is more readable, so please avoiud using set without a value.

```cmake
set(val1 Value)
if(DEFINED val1)
    message("val1: ${val1}") # val1: Value
endif()

unset(val1)
if(NOT DEFINED val1)
    message("val1 is not defined") # val1 is not defined
endif()

set(val2 Value)
if(DEFINED val2)
    message("val2: ${val2}") # val2: Value
endif()

set(val2)
if(NOT DEFINED val2)
    message("val2 is not defined") # val2 is not defined
endif()
```

By default, CMake treats undefined variables as empty strings, which can be useful in certain cases, such as when building conditional statements `if(NOT varName)`. However, this behavior can also lead to unexpected results when you expect a variable to have a specific value, but it is actually undefined. To help catch potential errors, CMake provides the `--warn-uninitialized` command-line option, which enables warnings for accessing uninitialized variables. This option can be useful for improving the reliability of your build process and catching potential issues early on.

```cmake
message(STATUS "val1: ${val1}")
```

In the above code, val1 is not declared.

```
Warn about uninitialized values.
CMake Warning (dev) at CMakeLists.txt:11 (message):
  uninitialized variable 'val1'
```

# Setting variable in the parent scope

You can modify a variable using PARENT_SCOPE keyword in set or unsetcommand.

```cmake
function(fun1)
    set(var1 "set inside function" PARENT_SCOPE)
    unset(var2 PARENT_SCOPE)
endfunction()

set(var2 "already set")

message(STATUS "var1 before function call: ${var1}")
message(STATUS "var2 before function call: ${var2}")
fun1()
message(STATUS "var1 after function call: ${var1}")
message(STATUS "var2 after function call: ${var2}")
```

In the above code, inside of `fun1`, `var1` is set and `var2` is set in the `PARENT_SCOPE`.

```
-- var1 before function call: 
-- var2 before function call: already set
-- var1 after function call: set inside function
-- var2 after function call:
```

When the variable is set in the parent scope, it is not defined in the current scope by default. This can be surprising, especially for those who are used to the scoping rules of other programming languages.

# Environment Variables

You can access environment variables using ENV{var}. It can be used in place where the variable can be used.

```cmake
if(DEFINED ENV{HOMEPATH})
    message(STATUS "windows home dir: $ENV{HOMEPATH}") 
elseif(DEFINED ENV{HOME})
    message(STATUS "linux home dir: $ENV{HOME}")
endif()
```

# Cache Variable
Cache variables are a powerful feature that allow you to store persistent configuration information for your project. When you define a cache variable, an entry is made in the `CMakeCache.txt` file generated in the build directory. This means that the value of the cache variable is persisted across multiple invocations of CMake, making it a valuable tool for configuring and customizing your build system.

Unlike normal variables, cache variables store additional information about the variable, such as its type and a brief description of its purpose. This additional information makes it easier for GUI tools and editors to display and interact with cache variables in a user-friendly way.

CMake provides several different ways to manipulate cache variables, depending on your specific use case and requirements.

- The most basic way to manipulate cache variables is using the `set` and `unset` commands. These commands allow you to set the value of a cache variable to a specific value or unset it completely.
- The `option` command is another way to manipulate boolean cache variables. It also has some advanced features which is not available in the `set` command.
- You can set the value of a cache variable using a command-line argument that starts with the `-D` prefix.

```
set(<variable> <value>... CACHE <type> <docstring> [FORCE])
```

`type` is the cache variable type. It can have the following values.

1. `BOOL`: Value can be On/Off. GUI tools(CMake-GUI) uses check box to show this.
2. `FILEPATH`: This is the path of a file. GUI tools use a file dialog to show this.
3. `PATH`: This is the path of a directory. GUI tools use a file dialog to show this.
4. `STRING`: Any string. GUI tools use a line edit to show it.
5. `INTERNAL`: This is an internal type, will not be shown in the GUI. This type is used to store result computationally intensive tasks, e.g. result of a find_packagecommand.
6. `UNINITIALIZED`: The type is not specified. 

`docstring` is optional, if provided, shall be a string which briefly describes the variable.

Let’s create a cache variable.

```cmake
set(var "hello world" CACHE STRING "a variable")
message("var=${var}") #var=hello world
```

After the first run, if you open `CMakeCache.txt`, you will find an entry for `var`.

```
//a variable
var:STRING=hello world
```

When CMake is run for first time, a cache variable will be created with the value “hello world”. In all subsequent runs, the command will be ignored.

If you set the FORCE keyword, the value will be updated.

```cmake
set(var "hello world second time" 
    CACHE STRING 
    "a variable"
    FORCE
)
message("var=${var}") # var=hello world second time
```

If you need to use `FORCE` keyword often, it will defeat the purpose of cache variable. However, there are many valid usecases of `FORCE` keyword.

`INTERNAL` types have `FORCE` keyword by default enabled. It is not recommended to use `INTERNAL` type.

`option()` command is like `set()` command with type `BOOL`.

```cmake
option(<variable> "<help_text>" [value])
```

If `value` is omitted, `OFF` is the default value.

```cmake
option(VERBOSE_MSG "Show all messages" OFF)

if(VERBOSE_MSG)
    message("Verbose message is enabled")
endif()
```

If cmake is invoked using `-DVERBOSE_MSG=ON`, then “Verbose message is enabled” will be printed. For the subsequent run, the value will be retained.

However, in CMake 3.13 version, option command is different. If another variable with the same name is defined, the option command is completely ignored. This is behavior is useful in subdirectory projects. Some CMake variables like `BUILD_SHARED_LIBS` has a project wide effect. If the subdirecory project has explicitly set the `BUILD_SHARED_LIBS`, then the global `BUILD_SHARED_LIBS` is ignored for that project.

The command line argument with `-D` prefix is more flexible than the setand option command. This behaves like set command called with `CACHE` and `FORCE` option.

```cmake
message(STATUS "var_from_d=${var_from_d}")
```

```bash
$ cmake -S <source-dir> -Dvar_from_d="hello world from terminal"
-- var_from_d=hello world from terminal
```

With `-D` option, the type parameter can be omitted. In that case, the type becomes `UNINITIALIZED`.

## Special behavior of type `PATH` and `FILEPATH`

If the type is `PATH` or `FILEPATH` and the value is a relative path, then it is interpreted as being relative to the current working directory, which may vary depending on where CMake was invoked. This can result in inconsistent behavior and errors when the CMake script is run from different locations.

Using an absolute path ensures that the path is always resolved correctly, regardless of where CMake is invoked from. This helps to ensure consistent and reliable behavior across different environments.

## Cache and Normal variable in the same scope with same name

In a scope, if a normal variable and cache variable has the same name, then the normal variable will always shadow the cache variable. In some cases, this might lead to undesired behavior, like you delete the normal variable using unset and after that the cache variable is not shadowed anymore.

So, it is a recommended practice to have some prefix like `MY_PROJECT` for all cached variable.

If this situation can not be avoided, then `$CACHE{varName}` shall be used to explicitly access the cache varible.
