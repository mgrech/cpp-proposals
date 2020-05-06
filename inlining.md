# Really `inline`
Author: Markus Grech &lt;me at mgrech.dev&gt;
Target: C++23
Number: P?????

## Abstract
Today's compilers do a reasonably good job of deciding which functions or function invocations should be inlined. In order to improve inlining decisions many implementations support attributes that let the programmer ensure that a function does or does not get inlined.

This proposal argues that not only does inlining make sense for optimization purposes, there are also cases where a particular inlining decision is required for correctness and adds a standard way to enforce that, tentatively spelled `__inline_always` and `__inline_never`.

## Introduction
Inlining is a program transformation where the body of a function is copied to the invocation site. Whether or not this improves performance can depend on many factors such as the surrounding code and hardware characteristics. This type of inlining is crucial for good performance because it enables many other optimizations.

Because inlining eliminates the function call, it also affects the stack layout. An inlined function call does not need to push a return address onto the stack which means that inlined calls do not show up in stack traces. There are some instances where this can even affect the correctness of code and this gap is currently filled by compiler-specific annotations.

The `inline` keyword was initially intended as a suggestion to the compiler that inlining the function might be profitable. Today however, due to the effects it has on linkage it mainly serves as a way to be able to define functions in headers without violating ODR. Compilers usually ignore the inlining hint entirely, instead offering compiler-specific annotations for this purpose. There is currently no way in the standard to declare that inlining must or must not happen for a function to be correct and the author is also not aware of any such compiler extensions.

## Motivating Examples

### Stack Walking
Suppose that the programmer wants to implement stack traces. Such code would still like to use functions as an abstraction, but using functions interferes with the stack. The frames of the stack walking code need to be removed from the stack trace since they are an implementation detail, but the implementer cannot possibly know how many frames to skip because the compiler is in charge of the inlining decisions. With `__inline_never` the programmer could count the number of frames in the source code and remove them from the trace; with `__inline_always` the stack frames do not exist at all. Either way under this proposal it would be possible to write this code in a standardized way.

### Inlining in Debug Mode
A complaint that the author has heard from the game industry is that modern C++ abstractions like `std::unique_ptr` can make code unusably slow in debug mode because small accessor functions are not inlined. It would be useful to have a standard language feature to avoid this overhead and it should be applied to the standard library in various places to have confidence that the abstractions are truly zero-overhead, even in debug mode.

This paper focusses on the language feature itself, but a second paper will apply `__inline_always` to the standard library where it makes sense.

### Inlined Functions instead of Macros
With forced inlining in the standard, programmers that previously used macros to avoid function call overhead can use regular functions instead and benefit from the advantages that functions have over macros, such as scoping and no issues with evaluating arguments multiple times. Inlined functions thus could help to eliminate macro use cases.

### Dispelling the Magic of `std::source_location::current`
`std::source_location::current` is currently a bit magical: It is not a macro but a function and somehow knows the location of its call site. No other function in the standard currently behaves this way. Due to the way this proposal specifies `__inline_always`, `std::source_location::current` can simply be marked as such in the standard, no magic needed.

### Wrapping `std::source_location::current`
Similar to `std::source_location::current` itself, with `__inline_always` it is possible to wrap without using macros. Example:
```cpp
__inline_always
void log(char const* message)
{
	auto loc = std::source_location::current();
	std::printf("%s: %s\n", loc.function_name, message);
}

void foo()
{
	log("hello"); // prints "foo: hello"
}
```

## Existing Practise

### C++
GCC and Clang implement the `__attribute__((always_inline))` and `__attribute__((noinline))` [extensions](https://gcc.gnu.org/onlinedocs/gcc-4.7.2/gcc/Function-Attributes.html) as well as the standard attribute forms `[[gnu::always_inline]]` and `[[gnu::noinline]]`. MSVC has `__forceinline` and `__declspec(noinline)`.

What these extension have in common is that they do not make the code fail to compile if a function cannot be inlined. This is a problem, because if a function needs to be inlined for the code to be correct, incorrect code can compile regardless. These attributes are also just hints for the optimizer, whereas in this proposal they are processed by the compiler frontend, see the design section.

### Kotlin
The Kotling programming language supports inlining very similar to what is proposed in this paper as `__inline_always`. The [`inline` keyword](https://kotlinlang.org/docs/reference/inline-functions.html) in Kotlin causes the function body to be copied to every call site.

The Java runtime has a way to retrieve the current location in the source code. When called from a Kotlin `inline` function, this function does not show up in the trace as expected:
```kotlin
inline
fun currentFunction(): String
{
	val trace = Thread.currentThread().stackTrace
	println(trace.joinToString(" -- "))

	// trace[0] is Java's getStackTrace method, which is mapped to .stackTrace in Kotlin
	return trace[1].methodName
}

fun main(args: Array<String>)
{
	println(currentFunction())
}
```
The output can be seen here: https://pl.kotl.in/bSZpvVo4V

## Design
`__inline_never` prevents inlining of a particular function. It is merely a way to communicate to the compiler and optimizer that inlining a function could lead to incorrect code and is thus forbidden.

`__inline_always` tells the compiler frontend to substitute all invocations by the body of the function. The optimizer never sees such a function, as it has already been removed by this point. The function never shows up in the resulting object code and hence does not create a symbol that could be referenced at link time. It is also not a suggestion: If the function cannot be inlined, the program is ill-formed.
Since `__inline_always` functions are processed by the compiler frontend, they also exhibit different restrictions from regular functions:
- Taking the address is prohibited, since the function never actually exists in the resulting binary; diagnostic required
- The call graph should not contain any cycles consisting only of `__inline_always` functions; diagnostic required (The reasoning similar to why `struct X { X x; };` cannot exist.)

It is still illegal to return a pointer to a local variable from an `__inline_always` function and expect it to be valid after the function call. This is because the inlined function is copied into the calling function as if the body was executed in its own scope:

```cpp
__inline_always
int* bar()
{
	int i;
	return &i;
}

void foo()
{
	int* p = bar();
	*p = 42; // undefined behavior, even with `__inline_always`
}
```

The above code is semantically equivalent to the following:

```cpp
void foo()
{
	int* p;

	{
		int i;
		p = &i;
	}

	*p = 42; // undefined behavior
}
```

`std::source_location::current` returns the source location after `__inline_always` functions have been expanded. Example:
```cpp
__inline_always
char const* current_function()
{
	return std::source_location::current().function_name();
}

void foo()
{
	std::printf("%s\n", current_function()); // prints "foo"
}
```
More specifically, the `line` and `column` of the `std::source_location` object point to the function call to `current_function` inside `foo`.

## Impact
This feature is purely a language extension, it does not break existing code.

## Spelling
It seems sensible to use the `inline` keyword for this purpose in some fashion. The inlining decision could be specified in parentheses, such as `inline(always)` and `inline(never)`, making `always` and `never` contextual keywords.
