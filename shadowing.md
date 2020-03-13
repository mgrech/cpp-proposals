# Shadowing Variables within the Same Scope

## Abstract
Allow defining re-defining variables within the same scope, shadowing previous definitions.

## Introduction
C++ already allows shadowing in inner scopes. This can be seen in the following code example:

```cpp
void f()
{
	int a = 1;

	{
		std::cout << a; // prints 1
		int a = 2; // ok, 'a' refers to a new variable
		std::cout << a; // prints 2
	}

	// 'a' refers to the outer 'a'
	std::cout << a; // prints 1
}
```

However, within the same scope it is prohibited:

```cpp
void g()
{
	int a = 1;
	std::cout << a;

	int a = 2; // error: redefinition of 'a'
	std::cout << a;
}
```

This is unfortunate, because as it turns out such shadowing has some interesting properties that can make for better code.

## Motivation

There is a multitude of different reasons why one might want to do this, here is a non-exhaustive list of examples:
- Adding `const` after the fact, for instance:

```cpp
int n = 0;

// compute 'n'
// ...

// computation done, 'n' should not be changed anymore
int const n = n;
// 'n' is now const from this point on
```

- Unwrapping: When working with values like std::optional, it can be useful to give the optional and the unwrapped value the same name:

```cpp
void foo()
{
	std::optional<Foo> foo = findFoo();

	if(!foo)
		throw std::runtime_error("foo not found");

	// we don't need the optional anymore, so let's replace the variable binding
	auto foo = *foo;

	// ...
}
```

- Preventing bugs
```cpp
void onUserInput(std::string const& input)
{
	ParsedInput input = parseAndSanitize(input);
	// the unsanitized 'input' variable is no longer accessible and thus cannot be used by accident
}
```

The last two examples show an interesting property of this feature: It allows the programmer to atomically replace a variable binding with another, in the sense that the old value ceases to be accessible and the name refers to the new value from this point onwards. Unlike nested shadowing, the old variable never becomes accessible again, reducing the number of things the programmer needs to mentally keep track of.

Often programmers need to perform some kind of unwrapping or transformation on a value that changes its type, but because C++ does not permit re-using variable names within the same scope, they need to choose two different but usually very similar names. It can easily happen then that the wrong variable is used on accident, resulting in difficult to find bugs and/or security issues. This proposal allows this entire class of errors to be eliminated.

## Existing Practise
The Rust programming language already [allows][1] this behaviour.

[1]: https://doc.rust-lang.org/rust-by-example/variable_bindings/scope.html
