---
layout: post
title: Function and Macros in CMake
tags: [CMake, BackToBasics]
---

CMake supports both functions and macros to provide a named abstraction for some repetitive works. A function or macro always define a new command.

# Functions

```cmake
function(<someName> [<arg1> ...])
  <commands>
endfunction()
```
where `name` is the name of the function, with arguments `arg1`, `arg2`, etc.

Example

```cmake
function(foo)
    message("I am foo!!")
endfunction()

foo()
FOO()
fOO()
Foo()
```

```
I am foo!! 
I am foo!! 
I am foo!!
I am foo!!
```

The function name is case-insensitive. You can call in any case, but it’s always recommended to use the same name declared in the function definition.

### Function Arguments
A cmake function can take two types of arguments.

- named or keyword arguments
- optional arguments

Named arguments are mandatory and will throw error if not provided. You don’t need a comma between argument names.

```cmake
function(fooKeyword country city)
    message("Address: ${country} ${city}")
endfunction()

fooKeyword("germany" "munich")
```

Output

```
Address: germany munich
```

Optional arguments can be accessed using some predefined variables.

- `ARGC` : Total number of arguments(named arguments + optional arguments)
- `ARGV` : list of variables containing both named and optional arguments
- `ARGN` : list of variables containing only optional arguments

```cmake
function(foo name1 name2)
    message("Argument count: ${ARGC}")
    message("all arguments: ${ARGV}")
    message("optional arguments: ${ARGN}")
endfunction()

foo("asit" "dhal" "India" "Bhubaneswar")
message("only named arguments")
foo("harry" "potter")
# foo() #error
```

Output

```cmake
Argument count: 4
all arguments: asit;dhal;India;Bhubaneswar
optional arguments: India;Bhubaneswar
only named arguments
Argument count: 2
all arguments: harry;potter
optional arguments:
```

Other than those three variables, CMake also provides `ARGV0`, `ARGV1`, `ARGV2`, … which will have the actual values of the arguments passed in. Referencing to `ARGV#` arguments beyond `ARGC` will have undefined behavior.

```cmake
function(foo name1 name2)
    if (DEFINED ARGV0)
        message("ARGV0 defined value=${ARGV0}")
    else()
        message("ARGV0 not defined")
    endif()

    math(EXPR lastIndex "${ARGC}-1")
    foreach(index RANGE 0 ${lastIndex})
        message("arguments at index ${index}: ${ARGV${index}}")
    endforeach()
endfunction()

foo("asit" "dhal" "India" "Bhubaneswar")
```

Output

```
ARGV0 defined value=asit
arguments at index 0: asit
arguments at index 1: dhal
arguments at index 2: India
arguments at index 3: Bhubaneswar
```

Usually, named arguments are accessed using the variable and optional arguments are accessed using `ARGN`.

```cmake
function(foo firstName lastName)
    message("first name: ${firstName}")
    message("last name: ${lastName}")

    foreach(arg IN LISTS ARGN)
        message("address parts: ${arg}")
    endforeach()

endfunction()

foo("asit" "dhal" "India" "Bhubaneswar")
```

Output

```
first name: asit
last name: dhal
address parts: India
address parts: Bhubaneswar
```

### Variable Scope

CMake functions introduce new scopes, a variable modified inside the function won’t be accessible outside the function. Another problem in CMake, the functions don’t return any value. This will make the function difficult to use. So, CMake provides the keyword `PARENT_SCOPE` to set a variable in the parent scope.

You can send variable name as a function parameter. The function will set the variable in the parent scope.

```cmake
set(name "Uttam Mohanty")

function(set_name varName )
    set(${varName} "Sriram Panda" PARENT_SCOPE)
endfunction()

message("before call first name: ${name}")
set_name(name)
message("after call first name: ${name}")
```

Output

```
before call first name: Uttam Mohanty
after call first name: Sriram Panda
```

This way of returning value from function is self documenting. But, it becomes repetitive if the function has to return many values. So, many library functions explicitly document well known variables which are set by a function and you don’t need to send any variable name as argument.

In some projects, it’s to common to have functions which set project specific variables in the parent scope. The cmake libraries provide `FindPackageHandleStandardArgs` function which sets a variable called Package_FOUND in the parent scope.

```cmake
find_package(Git)
message("Git found: ${GIT_FOUND}")
if(Git_FOUND)
  message("Git executable: ${GIT_EXECUTABLE}")
  message("Git version: ${GIT_VERSION_STRING}")
endif()
```

Output
```
Git found: TRUE
Git executable: /usr/bin/git
Git version: 2.17.1
```

### return() command

```cmake
function(early_ret)
    message("one")
    return()
    message("two") # won't be printed
endfunction()

early_ret() 
```

You can call `return()` to exit a function early.

# Macros

```cmake
macro(<name> [<arg1> ...])
  <commands>
endmacro()
```

where name is the name of the macro, with arguments `arg1`, `arg2`, etc.

```cmake
macro(foo_macro)
    message("I am a poor little macro!!")
endmacro()

foo_macro()
fOO_macro()
Foo_Macro()
FOO_MACRO()
```

Output
```
I am a poor little macro!!
I am a poor little macro!!
I am a poor little macro!!
I am a poor little macro!!
```

Like functions, macro names are also case insensitive. You can call in any case, but it’s always recommended to use the same name declared in the macro definition.

### Macro Arguments

Like functions, macros also take both named and positional arguments.

```cmake
macro(foo_macro name1 name2)
    message("Argument count: ${ARGC}")
    message("all arguments: ${ARGV}")
    message("optional arguments: ${ARGN}")
endmacro()

foo_macro("asit" "dhal" "India" "Bhubaneswar")
message("only named arguments")
foo_macro("harry" "potter")
#foo_macro() #error
```

Output

```cmake
Argument count: 4
all arguments: asit;dhal;India;Bhubaneswar
optional arguments: India;Bhubaneswar
only named arguments
Argument count: 2
all arguments: harry;potter
optional arguments: 
```

Functions always introduce a new scope when called. In case of macro, the macro call is substituted with the macro body and arguments are replaced using string substitution. No new scope is created. So both functions and macros behave differently in some cases.

Using the DEFINED keyword, you can check if a variable, cache variable or environment variable with given is defined. The value of the variable does not matter.

```cmake
macro(foo_macro name)
    if (DEFINED name)
        message("macro argument name is defined")
    else()
        message("macro argument name is not defined")
    endif()
endmacro()

function(foo_func name)
    if (DEFINED name)
        message("func argument name is defined")
    else()
        message("func argument name is not defined")
    endif()
endfunction()

foo_macro("asit")
foo_func("asit")
```

Output

```
macro argument name is not defined
func argument name is defined
```

Macros show strange behavior while using the three special variables(in some cases).

### Variable Scope

Unlike functions, macro introduce no new scope. The variables declared inside the macro(other than the arguments) will be available after the call.

```cmake
set(name "asit")
macro(foo_macro macro_arg1)
    message("macro arg: ${macro_arg1}")
    message("macro name: ${name}")
    set(middle_name "kumar")
    set(name "ASIT")
    message("micro middle name: ${middle_name}")
endmacro()

foo_macro("bhubaneswar")
message("after macro call name: ${name}")
message("after micro call middle name: ${middle_name}")
```

Output

```
macro arg: bhubaneswar
macro name: asit
micro middle name: kumar
after macro call name: ASIT
after micro call middle name: kumar
```

### return() command

Since, micro doesn’t create any new scope, return() will exit the current scope.

```cmake
macro(early_ret)
    message("one")
    return()
    message("two") # won't be printed
endmacro()

message("before")
early_ret()
message("will never be executed")
```

Output

```
before
one
```

# Redefining Functions and Macros

When `function()` or `macro()` is defined and if a command already exists with that name, the previous command will be overridden. The previous command can be accessed using the name and an underscore prepended.

```cmake
function (good_func)
    message("first definition of good_func")
endfunction()

function (good_func)
    message("second definition of good_func")
endfunction()

good_func()
_good_func()
```

Output

```
second definition of good_func
first definition of good_func
```

If the same function is redefined again, the underscore version will call previously defined function. The original function will never be available.

```cmake
function (good_func)
    message("first definition of good_func")
endfunction()

function (good_func)
    message("second definition of good_func")
endfunction()

function (good_func)
    message("third definition of good_func")
endfunction()

good_func()
_good_func()
```

Output

```
third definition of good_func
second definition of good_func
```

It has two problems.

- If a developer writes a function without knowing that the name is already used for some cmake library functions, then there are chances that the original function will be hidden forever.
- There are some developers who use this feature(or a bug) to add behavior to an existing function. If you do it more than twice, it can cause infinite recursion. It’s difficult to debug when the function definition spans across many modules.

```cmake
function (good_func)
    message("first definition of good_func")
endfunction()

function (good_func) # second redefinition
    message("second definition of good_func")
    _good_func()
endfunction()

function (good_func) 
    message("third definition of good_func")
    _good_func()
endfunction()

good_func()
```

Output
```
second definition of good_func
second definition of good_func
second definition of good_func
second definition of good_func
......
....
```

The desired behavior is that 2nd redefinition will call the original function. But, in this scope, the 2nd redefinition is always the underscore version.

# CMakeParseArguments

CMake has a predefined command to parse function and macro arguments. This command is for use in macros or functions. It processes the arguments given to that macro or function, and defines a set of variables which hold the values of the respective options.

```
cmake_parse_arguments(<prefix> <options> <one_value_keywords> <multi_value_keywords> <args>...)

cmake_parse_arguments(PARSE_ARGV <N> <prefix> <options> <one_value_keywords> <multi_value_keywords>)
```

- The first version can be used both in functions and macros. The 2nd version`(PARSE_ARGV)` can only be used in functions. In this case the arguments that are parsed come from the ARGV# variables of the calling function. The parsing starts with the `<N>`th argument, where `<N>` is an unsigned integer. This allows for the values to have special characters like `;` in them.

- `prefix`: a prefix string which will preceed all variable names.

- `options`: all options for the respective macro/function, i.e. keywords which can be used when calling the macro without any value following. These are the variables if passed will be set as `TRUE`, else `FALSE`.

- `one_value_keywords`: all keywords for this macro or funtion which are followed by one value, e.g. `DESTINATION=/usr/lib`

- `multi_value_keywords`: all keywords for this macro/function which can be followed by more than one value, like e.g. `FILES=test.cpp;main.cpp`

- All remaining arguments are collected in a variable `<prefix>_UNPARSED_ARGUMENTS` that will be undefined if all arguments were recognized. This can be checked afterwards to see whether your macro was called with unrecognized parameters.

```cmake
function(demo_func)

    set(prefix DEMO)
    set(flags IS_ASCII IS_UNICODE)
    set(singleValues TARGET)
    set(multiValues SOURCES RES)

    include(CMakeParseArguments)
    cmake_parse_arguments(${prefix}
                     "${flags}"
                     "${singleValues}"
                     "${multiValues}"
                    ${ARGN})
    message(" DEMO_IS_ASCII: ${DEMO_IS_ASCII}")
    message(" DEMO_IS_UNICODE: ${DEMO_IS_UNICODE}")
    message(" DEMO_TARGET: ${DEMO_TARGET}")
    message(" DEMO_SOURCES: ${DEMO_SOURCES}")
    message(" DEMO_RES: ${DEMO_RES}")
    message(" DEMO_UNPARSED_ARGUMENTS: ${DEMO_UNPARSED_ARGUMENTS}")
endfunction()

message("test 1")
demo_func(SOURCES test.cpp main.cpp TARGET mainApp IS_ASCII config DEBUG)
message("test 2")
demo_func(TARGET mainApp.so RES test.png cat.png bull.png IS_UNICODE config RELEASE)
```
Output

```
test 1
 DEMO_IS_ASCII: TRUE
 DEMO_IS_UNICODE: FALSE
 DEMO_TARGET: mainApp
 DEMO_SOURCES: test.cpp;main.cpp
 DEMO_RES: 
 DEMO_UNPARSED_ARGUMENTS: config;DEBUG
test 2
 DEMO_IS_ASCII: FALSE
 DEMO_IS_UNICODE: TRUE
 DEMO_TARGET: mainApp.so
 DEMO_SOURCES: 
 DEMO_RES: test.png;cat.png;bull.png
 DEMO_UNPARSED_ARGUMENTS: config;RELEASE
```

The above example with `PARSE_ARGV` version

```cmake
function(demo_func project_name project_type)
    set(prefix DEMO)
    set(flags IS_ASCII IS_UNICODE)
    set(singleValues TARGET)
    set(multiValues SOURCES RES)

    include(CMakeParseArguments)

    cmake_parse_arguments(PARSE_ARGV 2 ${prefix}
                     "${flags}"
                     "${singleValues}"
                     "${multiValues}")

    message(" DEMO_IS_ASCII: ${DEMO_IS_ASCII}")
    message(" DEMO_IS_UNICODE: ${DEMO_IS_UNICODE}")
    message(" DEMO_TARGET: ${DEMO_TARGET}")
    message(" DEMO_SOURCES: ${DEMO_SOURCES}")
    message(" DEMO_RES: ${DEMO_RES}")

    message(" named arg project_name: ${project_name}")
    message(" named arg project_type: ${project_type}")

endfunction()

message("test 1")
demo_func("main_gen" "gen" SOURCES test.cpp main.cpp TARGET mainApp IS_ASCII)
message("test 2")
demo_func("main_test" "unit test" TARGET mainApp.so RES test.png cat.png bull.png IS_UNICODE)
```

Output
```
test 1
 DEMO_IS_ASCII: TRUE
 DEMO_IS_UNICODE: FALSE
 DEMO_TARGET: mainApp
 DEMO_SOURCES: test.cpp;main.cpp
 DEMO_RES: 
 named arg project_name: main_gen
 named arg project_type: gen
test 2
 DEMO_IS_ASCII: FALSE
 DEMO_IS_UNICODE: TRUE
 DEMO_TARGET: mainApp.so
 DEMO_SOURCES: 
 DEMO_RES: test.png;cat.png;bull.png
 named arg project_name: main_test
 named arg project_type: unit test
```

