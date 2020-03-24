# Explicitly Shadowing and Undeclaring Variables

Author: Markus Grech &lt;me at mgrech.dev&gt;
Target: C++23
Document Number: P?????

## Abstract
This proposal adds two new language features, tentatively spelled `__undeclare` and `__shadow` to explicitly undeclare and shadow local variables.

## Introduction
C++ already allows shadowing in nested scopes:

```cpp
int const i = 1;
// here there is only one 'i' and using the name 'i' refers to this 'i'
{
	// 'i' still refers to the first 'i'
	int const i = 2; // ok, creates a new 'i'
	// now 'i' refers to the second definition
}
// 'i' refers to the first 'i' again
```
Note however, that this kind of shadowing has an undesirable property: When reading the source code from top to bottom, there are three regions with different meanings for the name 'i'. On the other hand, shadowing within the same scope is currently not allowed:

```cpp
void g()
{
	int i = 1;
	// 'i' refers to first 'i'

	int i = 2; // error: redefinition of 'i'
	// 'i' would refer to second 'i'
}
```

In this example, there would only be two regions with different meanings of the name 'i' and the second definition marks a clear point where the meaning of 'i' changes for the rest of the scope. When encountering the name 'i', the programmer only needs to scan upwards to the nearest definition within the current scope to find the one the name is referring to. The fact that this kind of shadowing is not allowed is unfortunate, because it is less confusing than nested shadowing and as it turns out can improve code and help to avoid bugs.

With this proposal, the programmer can spell the above example using `__shadow`. In addition, a second feature is introduced: `__undeclare`, which simply removes a variable from the scope.

## Motivational Examples
- Unwrapping and/or transforming: When working with values like `std::optional`, it can be useful to express a series of transformations (that may even change the type) by re-using the variable name:
	```cpp
	void f()
	{
		// this is *the* Foo in this function, so what better name is there, than just 'foo'?
		std::optional<Foo> foo = findFoo();

		if(!foo)
			throw std::runtime_error("foo not found");

		// we don't need the optional anymore, so let's unwrap it and re-use the name
		__shadow auto foo = *foo;
		// ...
	}
	```
	Without this feature, we would need two different names, which could lead to awkward code or even bugs:
	```cpp
	void f()
	{
		std::optional<Foo> fooOpt = findFoo();

		if(!fooOpt)
			throw std::runtime_error("foo not found");

		auto foo = *fooOpt;
		// we could still accidentally use 'fooOpt' here, which may result in things like:
		// - accidentally checking the optional again (performance bug)
		// - passing the wrong variable to another function ('fooOpt' instead of 'foo')
		// - etc.
	}
	```

- Preventing security issues
	```cpp
	void handle(std::string const& input)
	{
		__shadow auto input = sanitize(input);

		// old 'input' no longer accessible, cannot be used by accident
		// there is only one 'input' here, and it's the sanitized one
		doWork(input);
	}
	```
	Or alternatively:
	```cpp
	void handle(std::string const& input)
	{
		auto sanitizedInput = sanitize(input);
		__undeclare input;
		// 'input' no longer accessible

		doWork(input); // error: variable not accessible (bug prevented)
	}
	```
- Marking moved-from objects as inaccessible:
	```cpp
	std::vector<int> v;
	foo(std::move(v));
	__undeclare v;

	// cannot use 'v' by accident
	```

Fundamentally, these two features allow the programmer to avoid bugs by making variables that are no longer needed inaccessible and improve ergonomics by allowing name re-use through an explicit declaration. Reducing the number of variables in scope decreases the amount of things the programmer needs to keep track of.

Unlike shadowing in nested scopes, a variable that has been hidden by a `__shadow` definition in the same scope is hidden for the rest of the function, making it less confusing or dangerous than nested shadowing.

## Existing Practise
The Rust programming language [allows](https://doc.rust-lang.org/rust-by-example/variable_bindings/scope.html) shadowing within the same scope. Indeed, this is a common and encouraged pattern in Rust: In "[The Rust programming language](https://doc.rust-lang.org/1.30.0/book/2018-edition/ch02-00-guessing-game-tutorial.html)", the "Guessing Game" example demonstrates how such shadowing can be used advantageously.

Same-scope shadowing is received favorably in the Rust community and doesn't appear to be a huge source of bugs:
- https://www.reddit.com/r/rust/comments/4hn3lw/local_variable_shadowing_removes_immutability/d2r6fd3/
- https://discuss.kotlinlang.org/t/rust-style-variable-shadowing/16338
- https://users.rust-lang.org/t/inquiring-about-idiomatic-usage-of-shadowing/6177

One important difference between Rust and C++ is that Rust macros are hygienic: It is [not possible](https://doc.rust-lang.org/1.7.0/book/macros.html#hygiene) for a Rust macro to introduce new variables with arbitrary names into the scope.

## Proposed Changes
- Introduce the `__undeclare` statement. The effect of this statement is to remove the given name from the current scope. This statement can only be used at function scope and only with names that refer to local variables. The lifetime of undeclared variables is unaffected. Attempting to re-use the name is an error. Example:
	```cpp
	void f(int x)
	{
		__undeclare f; // error, name does not refer to local variable
		__undeclare x; // ok

		int y;
		__undeclare y; // ok

		int x; // error, redefinition
	}

	__undeclare f; // error, not at function scope (in addition to not naming a local variable)
	```
	For consistency with declarations, it is legal to undeclare multiple variables at once:
	```cpp
	int a, b, c;
	__undeclare a, b, c;
	```

- Introduce the `__shadow` specifier for local variable definitions. The definition `__shadow T x = init;` is equivalent to `__undeclare x; T x = init;`, except that the redefinition is not an error. Usage of the `__shadow` specifier is also allowed in nested scopes that define a new variable with the same name of a variable in an outer scope:
	```cpp
	int x = 1;
	{
		__shadow int x = 2; // ok
	}
	```
	Using `__shadow` on a definition that does not shadow an existing variable is an error:
	```cpp
	int x = 1;
	__shadow int y = 2; // error, does not shadow anything
	```

	For variables defined with `__shadow`, names in the initializer never refer to the variable shadowing, but to the variable being shadowed. Example:

	```cpp
	int x = 1;
	__shadow int x = 2 * x; // previous 'x'
	```

	`__shadow` may be applied to multiple definitions, in which case it applies to all of them:
	```cpp
	int x, y;
	__shadow int x, y; // ok
	__shadow int x, z; // error, z does not shadow
	```

- Deprecate nested shadowing without the `__shadow` specifier:
	```cpp
	int x;
	{
		int x; // deprecated
	}
	```

- Deprecate initializers referring to the variable being defined:
	```cpp
	void* p = &p; // deprecated
	```
	This change applies to all variables, not just local variables.

## Impact
None of the changes introduced break existing code. Usage of the deprecated features may result in compiler diagnostics, allowing programmers to transition away from these features with advanced warning.

## Rationale and Design Decisions
### Making Shadowing Explicit
This is to avoid accidental shadowing, which could happen in [nested index loops](https://erikmcclure.com/blog/name-shadowing-should-be-an-operator/). Another potential source of accidental variables are macros.

An alternative to explicit shadowing is to require the initializers of shadowed variables to mention the shadowed variables, ensuring that the new value is derived from the old value:
```cpp
int x = 1;

int x = 2 * x; // ok, initialized from old 'x'
int x = 0;     // error
```
However, this does not provide complete protection against unrelated shadowing:
```cpp
int x = ((void)x, 0); // uses old 'x', but the new variable can be any arbitrary unrelated value
```
Rather than to try to provide protections against unrelated shadowing, the proposal makes shadowing explicit, allowing code search to find all occurences. Detecting undesirable shadowing can then be added by compilers and linters as a warning option, like Rusts [clippy](https://rust-lang.github.io/rust-clippy/master/#shadow_unrelated).
Finally, by making shadowing explicit, the transition from existing code that uses nested shadowing is easier than if shadowing were restricted.

### Disallowing Redefinitions after `__undeclare`
Like explicit shadowing, this is to prevent accidental redefinitions, either directly or through macro invocation.

### Deprecating Nested Shadowing without `__shadow`
This proposal uses the opportunity of a new `__shadow` specifier to deprecate nested shadowing without it for consistency. Since nested shadowing is more dangerous than shadowing within the same scope, it makes sense to also require it here.

### New Initializer Rules in `__shadow` Definitions
This is necessary for shadowing in the same scope to even be remotely useful. If a shadowed definition could not refer to the previous definition, many handy transformation patterns simply would not work.

### Deprecating Self-Referential Variable Initializers
For consistent semantics between regular variable definitions and `__shadow` definitions this proposal deprecates self-referential initializers.

The author is aware of two use cases for these. One is in C, where the name is used in `malloc` to get the required size:
```cpp
T* p = malloc(sizeof(*p));
```
However, in C++ this code does not compile anyway, due to malloc returning `void*` and the conversion from `void*` to `T*` not being implicit.

Another is to perform recursive initialization:
```cpp
T x{ U{ &x } };
```
Not only are these cases rare, but if `U`'s constructor is non-trivial, this is a source of bugs (the constructor is passed a pointer or reference to an uninitialized value, which could then be used by accident, leading to undefined behavior).

When self-referential initialization is eventually removed, care must be taken to also disallow referencing *any* previous definition with the same name, not just local variables. Consider:
```cpp
int x = 1;

void f()
{
	int* x = &x; // currently refers to local 'x', but to global 'x' with changed rules
}
```
The same example also works within a class with a member variable named `x`. Such a local initialization must be an error, otherwise the meaning of this code would silently change. As a potential counterpoint, many coding standards require global and member variable names to have certain prefixes or suffixes and such cases of coincidental names are likely rare.

Finally, the rules could be left unchanged, resulting in inconsistent initialization semantics.

## Spelling
Potential spellings for `__undeclare a, b;`:
- `undeclare a, b;`
- `kill a, b;`
- `hide a, b;`
- `a, b = delete;`
- `private a, b;`
- `!using a, b;`
- `co_undeclare a, b;`
- `🗑️ <= a, b;`
- ...

Potential spellings for `__shadow T x = init1, y = init2;`:
- `shadow T x = init1, y = init2;`
- `redefine T x = init1, y = init2;`
- `override T x = init1, y = init2;`
- `new T x = init1, y = init2;`
- `co_shadow T x = init1, y = init2;`
- ...