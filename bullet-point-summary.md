# C++ style guide - concise form
- bullet point highlights
- This is a first draft - a starting point for discussion - not definitive  
  There are some hard constraints, but there is room for flexibility.
----------
## Background: terminology
- “**STL**” - an obsolete term that some people use to refer to the templatized containers portion of the C++ standard library. This term is never used in the C++ standard.


- “**C++ standard library**” - a collection of header files and a runtime shared library that must be provided by all C++ compilers.
  Some core syntactic features of C++ rely on the standard library
- “**C++ runtime library**” - A static or dynamic system library that provides some of the features of the C++ standard library


- “**header-only library**” - Any C++ library that provides some features entirely in header files, without requiring a dependency on a static or dynamic library.
  Some features from the C++ standard library are header-only.


- “[**Boost**](https://www.boost.org/)” - A free software collection of high-quality portable C++ constructs.
  - Many libraries in Boost are header-only.
  - Some features in C++11/14/17/20 started out as Boost libraries. 
- “[**Abseil**](https://github.com/abseil/abseil-cpp)” - Another free software C++ library collection. FOSS version of Google’s internal C++ framework.  Provides C++11-compatible implementations of some C++17 features.
----------
## Outline
- Part 1 - constraints
- Part 2 - the bikeshed: names and tabs
- Part 3 - language and library features
----------
## Mono C++ constraints due to toolchain
- C++11
  - unless deprecated in C++14, C++17, C++2x
- Some uncontroversial, header-only C++17 might be ok to adapt
  - `std::experimental::string_view` → `mono::string_view`
----------
## Mono C++ constraints due to embedding
- Mono API
  - all API functions `extern "C"`
  - Only pointers to opaque wrapper structs (like LLVM C API), or POD `struct`s
- Can’t touch some global C++ features
  - `::operator new(size_t)`, `::operator delete(void*)`
  - `std::set_new_handler(new_handler)`
----------
## Mono C++ constraints due to mobile products

Probably shouldn’t use most `<algorithm>` template functions - avoid code size blow up

- We should provide `mono::algorithm::func<T>` templates that
  - are only specialized for a small fixed set of types
  - fail to compile if instantiated with an unapproved type 
----------
## Mono C++ constraints due to historic C codebase

Can’t put anything with a non-trivial destructor in `MonoMempool`
Draconian restriction for safety:

    
No new members with non-trivial destructor in any existing** `**struct**`
(or in any `class` that previously had a trivial destructor)


New non-trivially destructible members only in new classes that are already themselves not trivially destructible.


Can perhaps automate this constraint `static_assert (std::is_trivially_destructible <T>, "don't mempool me, bro")`

----------
## Mono C++ constraints due to `-fno-exceptions`
- no `throw`, `try`/`catch`
- `noexcept` specs don’t do anything. We won’t know if we wrote them incorrectly. Might as well omit them (except where required)
----------
## Mono C++ constraints due to `-fno-rtti`
- no `dynamic_cast<>` - no checked downcasts.
- no `typeid`
- subclassing and `virtual` do work.
----------
## Mono C++ constraints due to no C++ runtime library
- use `mono::new_<T>(…)` instead of `new T(…)` - especially in templates
- use `mono::delete_(ptr)` instead of `delete ptr`
- all classes and new structs derive from `mono::base`
- all polymorphic base classes derive from `mono::polymorphic_base`
  - since `virtual C::~C()` calls  `C::operator delete(void*)`
- no pure virtual methods: `virtual void M () = 0`
  - can’t have abstract base classes or (morally) interfaces 😢 
- no: `std::vector<>` and other containers; some algorithms; streams; no `std::function<>`
- no `std::string<>`
----------
## Some things that work without the C++ runtime library
- header-only library functionality that compiles away is ok
  - `std::type_traits<>`, `std::enable_if<>`
  - `std::tuple<>`
  - `mono::unique_ptr<>` (or `g::ptr<>` or some other name)
- lambdas work
  - but without `std::``function``<>` can only use them in-place
  - (maybe add `mono::function<>` if we can’t live without it)
----------
## Indentation, Naming, Comments and bike shed paint color
- `.cpp` for new files, `.c` (as C++) for old.
- `.h` (no C++) for Mono API headers
- `.h` (C++) for legacy internal headers
- `.hpp` for new C++ only internal headers

We will autoformat `.cpp`  and `.hpp` files (on CI? in a git hook?)
Using `clang-format` with a `mono.clang-format` style

----------
## Indentation
- Mono style - tabs, tab width = 8, function braces go on a new line, space before open paren
- `template_function<TypeArgs> (ValueArgs)`
  - `static_cast<int> (foo)`
- namespaces indent
- member initialization:
    C::C (...) : member1 (init1), member2 (init2) { }

or

    C::C (...)
        : member1 (init1), member2 (init2)
    {
    }
## Identifier naming ♻️ 🔥 
- Google style guide has a naming convention
- Mono C/C# coding guidelines have another
- C++ standard uses a third one

“PRs welcome 😎”

----------
## Other style things
- C++ template specialization comments go with the **template definition**
- Okay to omit “obvious” argument names in function declarations.
  - Not ok to omit mystery `int` or `double`s, etc
- Class member declarations must be in `public`/`protected`/`private` order, don’t mix data/function `static`/non-`static` members.
----------
# Deep dive details

https://github.com/lambdageek/cppguide-wip


----------
## Don’t forward declare classes
  Forward declarations of class hierarchies can break overload sets
----------
## Put declarations into namespaces
- nesting ok
- no anonymous namespaces in headers
- anonymous namespaces ok in `.cpp` files
- no `using namespace Foo` in headers, at `.cpp` file scope, etc.
- no namespace aliases in headers
- avoid new symbols in the global namespace
----------
## No `static C foo` at file and function scope

Unless `C` is *trivially destructible and constant initialized*

- unspecified destructor order effects → shutdown bugs
- no `static C foo = expr`  at global scope,
  - except `constexpr static C foo = expr` is ok
  - unspecified initialization order leads to bugs
- no `static C foo = expr` at function scope
  runtime library dependency `__cxa_guard_acquire`
----------
## No `thread_local` storage specifier
  This is either arbitrary or there’s a C++ runtime library dependency
  
  I haven’t done the research: might be able to lift prohibition


----------
## Don’t use C++11 threading and relaxed memory model primitives

Functions and classes in `<thread>`, `<future>` and `<atomic>`
Haven’t done the research

Concerns:

- Coop: We need coop-aware versions of  `std::mutex<>` etc
- No standard library:  Probably can’t use `std::thread`
- `std::atomic<T>` and `std::memory_order` - Mono has its own primitives - should stick with one form.
----------
## Constructors
- We don’t have exceptions, so all constructors should be as if `noexcept`.
  Make factory methods and mark constructors `private`
- We love `class C { C () = default; }` **in headers**
  - *TriviallyDefaultConstructible* → compiler can optimize all `C data_member` uses
  - but only do this if the class has a natural default state
- We also love `C(const C&) = default` in headers
  - *TriviallyCopyable* if all data members are, too.  Compiler can use `memcpy`
  - if `C` has a single data member, often the whole class will compile away!
- Don’t call `virtual` methods in constructors.  Save it for a builder class.
----------
## Use `explicit` on conversions

No implicit conversion operators or single-argument constructors.

If you want `if (e)` provide `explicit operator bool() const`

----------
## Avoid C-style casts in new code.

Use `const_cast<>`, `static_cast<>` and `reinterpret_cast<>` or constructor-style casts.

----------
## Copyable and Movable
- If a class is copyable, define copy constructor and `operator =`
- If a class is not copyable, use `= delete` on the copy constructor and `operator =`
- If a class is movable define a move constructor and move-assignment
  - If the class is move-only
  - If move is more efficient than copy
- If a class is not-movable and not copyable, `=delete` all 4 operations
----------
## Class vs Struct

Use `class` for objects with behavior, `struct` for data-only

- All fields `private` in classes unless `static const`.  `public` or `protected` accessors
- `struct` can have public fields, but shouldn’t have methods that affect anything other than the `struct` - basically just `init` and `validate` and `reset`
----------
## No multiple inheritance

No multiple implementation inheritance.

No virtual bases.

Except on Windows

Reminder: we can’t have pure abstract base classes.

----------
## Only `public` base classes

Use composition instead of `private` bases.

----------
## Use `override` or `final` in subclasses

Instead of `virtual` on methods that you’re overriding

----------
## Use `final` on classes too

As appropriate

----------
## Mostly don’t overload operators
- Unless exactly matching builtin semantics
- Don’t overload the short-circuiting operators
- Don’t overload  `operator` `""`
----------
## Put large inline method bodies outside the class definition

Put them after the class definitions and mark `inline` if necessary

----------
## Prefer return values.

Use `std::tuple<>`

----------
## By reference arguments

(Controversial!)

- Only `const C&` allowed
- Use either `C*` or `std::reference_wrapper<C>` for by-ref arguments
    void
    callee (const C& c,
            RefArg1* r1, std::reference_wrapper<RefArg2> r2)
    {
        ...
    }
    void
    caller ()
    {
        using std::ref;
        callee (someC, &someR1, ref (someR2));
    }
----------
## Overload sets

“An ‘overload set’ is the atom of C++ API design”

  - (Titus Winters, Google, CPPCON 2018, [slides](https://github.com/CppCon/CppCon2018/blob/master/Presentations/modern_cpp_api_design_pt_1/modern_cpp_api_design_pt_1__titus_winters__cppcon_2018.pdf))
- Only add overloads that the caller doesn’t need to care about.
- Default arguments are effectively overloads
- `const` (and `&&`) overloads should be plentiful
- Avoid interaction of overloading and subclassing.
----------
## Trailing return types

Ok to use `auto foo (...) -> retType`  “trailing return” syntax

But prefer old-style `retType foo (...)` 

----------
## Memory ownership

Use `std::unique_ptr<>` (in its `mono::unique_ptr` guise that calls the right destructor)

Never use `std::auto_ptr<>`

----------
## Rvalue references

rvalue references are allowed. But don’t go crazy.

- Only if you expect `move` to be more efficient than a copy
- Or if you’re working with a class that is movable but not copyable
- or `std::forward<>` for perfect forwarding in templates
----------
## Friends

`friend`s are allowed and form part of the interface of a type

----------
## Increment

Prefer `++t` for if `t` isn’t of a simple scalar type, or if it’s a template type

----------
## Constness

Use `const` and `constexpr` liberally in new code

- `mutable` members are allowed if it helps to keep public methods `const`
  and doesn’t affect value identity
- Make functions `constexpr` if you can do it without contortions.
  C++11 `constexpr` is overly restricted, compared to C++17.
----------
## Preprocessor

Preprocessor macros don’t know about namespaces, so define them outside of namespaces and make sure they work with namespaces.

In general, prefer not to use macros in C++.
(Except the kinds of “syntactic” constructions that we have today like `return_if_nok` )

----------
## Null pointers

Use `nullptr` in new code.

----------
## Type inference
- `auto v` is allowed if it’s covering up the bleedin’ obvious:
  iterators, results of `new_<T>` and `make_unique<T>`, etc.
- `auto v {braced-init-list}` is not allowed
- `auto` and `const auto &` and `auto &&` can be a source of bugs.
  When initializing a var from a complicated expression, specify the type.
----------
## Braced  init lists
- `T f{}` is always allowed
- `C f{arg1, arg2}` is permitted, but you don’t want it in templates
----------
## No default captures in lambdas

Don’t use default captures with lambda expressions
Bad:

    [&]() { return foo; }

Good:

    [&foo]() { return foo; }
----------
## Template meta-programming
- Allowed, but don’t go crazy
    “If you're using recursive template instantiations or type lists or metafunctions or expression templates, or relying on SFINAE or on the `sizeof` trick for detecting function overload resolution, then there's a good chance you've gone too far.”
- Isolate into a separate header and an `impl` or `details` namespace.
  Try to provide a simple interface.
----------
## Boost and Abseil
- Header only libraries if they appear on the list below are allowed
- The list:
  - [none at this time]
- Explicitly disallowed:
  -  `boost::array` (use `std::array`),
  -  `boost::pointer_collection` (use a container of `std::unique_ptr`)
----------
## No specializations of `std::hash<>`

`std::hash` is broken; don’t use.
Don’t define specializations.
If `std` unordered containers were usable, we would pass a hash object for custom hashing.

----------
## GNU or MSVC++ extensions

Don’t use GNU or MSVC++ extensions without hiding them in a portability header

----------
## Type aliasing

Prefer `using FooClass = BarClass` in new code. Works with templates!

    namespace mono {
      template <typename T>
      using unique_ptr = std::unique_ptr<T, MonoDeleter>;
    } // namespace mono
