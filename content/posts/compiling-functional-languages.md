+++
title  = "Compiling Functional Languages"
date   = "2024-12-30"
author = "Jon"
tags   = ["project"]
+++

# Motivation & Background

Advent of Code is an annual challenge starting on Dec. 1st that asks users to solve LeetCode-style coding problems through Dec. 25th. I didn't *really* participate this year, as I didn't try to solve all of the problems (I especially did not solve them as they were released) but I used this year's Advent of Code as an opportunity to hone my knowledge around functional languages and how they are compiled. More specifically, I wanted to create a compiler for a functional language of my own design, and solve at least one Advent of Code problem with it. This having been written after-the-fact, I succeeded at this task, and the code is available in [this](https://github.com/jontab/aoc-2024) GitHub repository.

Compiling functional languages is not an easy task. A simpler one would be to just write an interpreter for it. An interpreter is just a program that evaluates the source code as it goes, the upshot being: the source code doesn't have to be translated to another language. The source code is parsed into a syntax tree unto which the interpreter descends, evaluating the values of nodes as it encounters them. If you think about it, machine code is kind of like source code for your computer to interpret: your computer being the ultimate interpreter.

> A digression about the distinction between compilers and transpilers: what I have written in that repository would actually be classified as a transpiler. A transpiler is the pretentious way of referring to a compiler that doesn't target machine code. A program that converts Python to C code, then, would be a Python-to-C transpiler. However, ambiguous distinctions of what constitutes machine code aside, if I called that Python-to-C program a compiler instead, then you would still understand my point.

If you are familiar with functional languages, then what I have implemented is a compiler for a custom language that resembles [Standard ML](https://en.wikipedia.org/wiki/Standard_ML), the precedent of modern-day OCaml. It features variant-type declarations, recursive let-statements with let-polymorphism, and [type inference](https://en.wikipedia.org/wiki/Hindley%E2%80%93Milner_type_system) with support for isorecursive types. Some of the stretch-goals that I haven't implemented for it at the time of writing are:

- **Garbage collection via reachability analysis**. Modern-day OCaml (and other functional languages) actually use something called **tagged pointers** to detect when objects go out of scope. In 64-bit OCaml, integers are actually only 63-bits because the last bit is set when the object you have is **atomic**. If it is not set, then it is not atomic, and you can follow the pointer to look at the object's properties. What this means, then, is that you can detect when object's go out of scope by performing reachability analysis via **the root set**, i.e., objects that are immediately reachable (perhaps they are on the stack). If you cannot reach a given object from the root set, then it is out-of-scope and can be freed / garbage collected. Reachability analysis is more powerful than garbage collection via reference counting, as reference counting can not handle mutually recursive structures (not that my language supports them).
- **Generic variant types**. Modern-day OCaml also allows you to define "generic" variant types like: `type 'a myList`.
- **Tuple-literals beyond length 2**. I didn't implement this because type inference with unspecified tuple lengths is less straightforward than forcing all tuples to be pairs. My first thought for how to implement this would be inferring from `x[5]` that `x` must be a tuple of at-least length 6 and, when two "at-least" tuple type-constructors are unified, we'd "set" the shorter one to the longer one. We'd impose an at-most (and at-least) restriction when we unify that with a tuple-literal (it has a specific length).
- **Curried functions**. "Currying" is an optimization when you convert expressions like `fun x -> fun y ->` to a function that takes multiple parameters in the first place: `fun x y ->`. It's an optimization so it wasn't high on my priority list.
- **"define" or "def" statements**. These statements would let you declare variables or functions that would be in-scope for sibling nodes. At the top-level, then, that would let you structure your program as you would in most procedural languages. You'd have your top-level functions, and you'd have your "main" value that calls those functions. My first thought on implementing this would just be desugaring those def-statements to contain their later sibling nodes as an explicit scope, effectively turning them into let-statements.

If you are not familiar with functional languages, then read on! I will be touching on those features in the following sections.

---

It can be said that there are two types of programming languages: procedural and functional. A simple way to identify such languages is by observing how code is written. For example:

```python
def myFunction():
    doThis()
    doThat()
    aVariable = doEverything()
    return aVariable
```

If the code flows step-by-step like this, it's likely procedural. These languages are characterized by their linear "flow of execution," which is read from top to bottom. In contrast, functional programming langauges emphasize the composition of functions. For example, the following Python code:

```python
(lambda x: x + 1)(5)
```

mimics a functional language. The terms "procedural" and "functional" refer to programming paradigms. Interestingly, a "procedural" language like Python can also support "functional" programming, making it "multi-paradigm." \*\*

A key feature of functional programming is its focus on "pure" functions. A "pure" function produces no side effects, meaning that it only depends on its input and returns a result. Here's an example of an impure function in Python:

```python
aNumber = 0
def giveMeTheNextNumber():
    global aNumber
    aNumber += 1
    return aNumber
```

In this case, the `giveMeTheNextNumber` function not only modifies a global variable (`aNumber`) but depends on it for its return value, making it impure. Procedural languages, by contrast, are built around manipulating state --- whether that state is global or local. To translate a functional program into a procedural one, then, we must introduce something called "linearization." This is when we re-introduce the step-by-step execution flow characteristic of procedural languages. For instance, consider the functional statement:

```
multiplyByTwo (divideByTwo 2)
```

Assuming `divideByTwo` and `multiplyByTwo` do as they are named, this evaluates to `1`. `multiplyByTwo` is called with the result of `divideByTwo 2`. To linearize this, we can use a let-statement to sequence execution:

```
let arg = divideByTwo 2 in
    multiplyByTwo arg
```

The procedural equivalent of this would be:

```
arg = divideByTwo(2)
return multiplyByTwo(arg)
```

Notice how the number of let-statements is directly proportional to the number of steps in its equivalent procedural program. Linearization, through a later step we will call **alpha-normalization**, brings functional code closer to its procedural counterpart.

# Overview

Compiling a language in general typically involves parsing its source code into an abstract syntax tree, or AST, and applying a series of transformations. These transformations might annotate the AST with additional data (e.g., with the types of variables) or replace parts of it entirely (e.g., when we "desugar" fancy syntax). The steps we will be following to compile our functional source code into C code are:

- Parsing
- Alpha-renaming
- Type inference and type checking
- Alpha-normalization
- Closure conversion
- Code generation

# Parsing

Parsing takes serialized source code and converts it back into an abstract syntax tree, or AST. The process begins with **lexing**, where the source text is split into tokens, and continues by interpreting the meaning of these tokens (semantics) in context to reconstruct the AST based on the grammar of the language. For instance, given the following source code:

```
let multiplyByTwenty = fun x -> muli x 20 in
    multiplyByTwenty 4
```

The lexer might produce the following tokens:

```
LET
IDENTIFIER multiplyByTwenty
EQUALS
FUNCTION
IDENTIFIER x
ARROW
IDENTIFIER muli
IDENTIFIER x
INTEGER 20
IN
IDENTIFIER multiplyByTwenty
INTEGER 4
```

These tokens are then parsed into an AST, like this:

```
LET
    name: multiplyByTwenty
    body: FUN
        argname: x
        value: APPLY
            left: VARIABLE
                name: x
            right: INTEGER
                value: 20
    value: APPLY
        left: VARIABLE
            name: multiplyByTwenty
        right: INTEGER
            value: 4
```

Manually writing lexers and parsers can be tedious, so in my [Advent of Code repository](https://github.com/jontab/aoc-2024), I use the Python library [lark](https://lark-parser.readthedocs.io/en/latest/) to handle this automatically. It is at this point that, if you were writing an interpreter, you might interpret the AST at this point --- executing the code without further processing. The interpreter could recursively traverse this AST while maintaining a dictionary of bound variables and their values. However, this post focuses on compiling our source language, not interpreting it. With parsing complete, the next step is alpha-renaming.

# Alpha-renaming

Alpha-renaming ensures that all variables in the syntax tree have unique names, eliminating "shadowed" variables. Shadowing occurs when a variable with the same name in an inner scope hides a variable in an outer scope. For example:

```
let x = 5 in let x = 2 in x (* returns 2 *)
```

Here, the `x` in the inner `let` shadows the `x` in the outer `let`. Alpha-renaming transforms this code to make variable names unique:

```
let x1 = 5 in let x2 = 2 in x2 (* clearly returns 2*)
```

With alpha-renaming, every variable in the source code maps directly to its definition, making later stages like type inference more straightforward. For instance, when dealing with **free variables** (variables not bound within their scope but used in a function), alpha-renaming removes ambiguity by making it clear which definition a variable belongs to. If you choose to maintain a global mapping of variable names to types or values, alpha-renaming is essential. However, in my implementation, I avoid such a global mapping, so theoretically, alpha-renaming could be skipped. That said, it is still a helpful first step to simplify reasoning about the AST.

Next, we proceed to **type inference and type checking**.

# Type Inference & Type Checking

If your language is **explicitly typed**, you can skip type inference entirely. Explicitly typed languages, such as C, require the programmer to declare the types of all variables and function parameters upfront. For example, in C, you might write:

```c
int x = 5;
```

Implicitly typed languages, on the other hand, do not require the user to annotate variables with their types. Instead, the compiler deduces these types through a process called type inference. It's important to distinguish type inference from static and dynamic typing:

- Static typing means the types of variables are determined at compile-time, ensuring that type errors (e.g., adding a string to an integer) cannot occur during runtime.
- Dynamic typing means types are checked at runtime, allowing more flexibility but at the cost of potential runtime errors. Python is an example of a dynamically-typed language.

Our compiler operates in a statically, implicitly typed setting. This means:

- The user does not have to explicitly annotate variables with their types.
- The compiler uses type inference to deduce types based on how variables are used.
- Errors related to type mismatches are caught at compile-time, not during execution.

#### Hindley-Milner Type Inference

The [Hindley-Milner type system](https://en.wikipedia.org/wiki/Hindley%E2%80%93Milner_type_system) provides a structured way to deduce types of programs or expressions. This process involves mapping variables to type variables and creating equations or "constraints" whenever those ariables are used in expressions (e.g., arithmetic or logical operations). To determine the type of an expression, we solve this system of equations, yielding the most general type for the given expression. For example, consider the following expression:

```
addi :: int -> int -> int (* it takes two ints and returns an int *)
addi x 1
```

The inference process works bottom-up on the syntax tree:
- When we see `x`, we introduce a new type variable, `T`, and assign it as the type of `x`.
- The `1` is recognized immediately as an `int`, so its type is directly assigned.
- The type of `addi` is looked up in the type environment: `int -> int -> int`.
- The type inference becomes meaningful during function application.

For function application like:

```
APPLY
    left: left
    right: right
```

We know `left` must be of the form `T1 -> T2`. If `right` has the type `T3` (or a concrete type like an `int`), we add the constraint `T1 = T3` to our system of equations and infer `T2` as the type of the application expression. So, returning to the example:

```
APPLY
    left: int -> int -> int
    right: T
```

Here, `T` must equal `int`, so we assign `T = int` and return the type `int -> int`. Then, we come across:

```
APPLY
    left: int -> int
    right: int (* from 1 *)
```

We see that `int = int`, and then we return `int` --- the right-hand side of the function type.

#### Type Checking

If the left-hand side of an application is not a function, or if there's a mismatch between the expected and actual types, we throw a type error:

- Example 1:

```
APPLY
    left: int
    right: T
```

The left-hand side is not a function, so this results in a type error.

- Example 2:

```
APPLY
    left: bool -> T
    right: int
```

Here, the function expects a `bool` but receives an `int`, resulting in a type error.

#### Unification

A key concept in Hindley-Milner type systems is **unification**. Unification involves combining multiple systems of equations into one unified system that, when solved, determines the type of the expression. 

For example, both the left and right-hand sides of an application may have their own complex systems of equations. Unification finds mappings that make these systems compatible. A type error occurs when there is an error in unification.

We've already been subconciously doing this "unification" wrt. the earlier examples.

#### Let-Polymorphism

While **let-polymorphism** isn't necessary for a language to be [Turing-complete](https://en.wikipedia.org/wiki/Turing_completeness), it significantly enhances flexibility. Everything we've covered so far falls under the category of monotypes. Let's [break it down](https://tenor.com/lV62WLIwu7q.gif):

- A **monotype** type variable `T` has exactly one solution. For example, `T` could be an `int` or `bool`.
- A **polytype** like `∀T. T -> T` has infinitely many solutions. The classic example of a value that should fit this type is the identity function:

```
let id = fun x -> x in id 5; id true
```

Without let-polymorphism, this would result in a type error. After encountering `id 5`, the compiler would map `id` to `int -> int`. Then, trying to use `id` with `true` would fail, as the type `int -> int` is incompatible with `bool -> bool`. However, let-polymorphism allows this functionality. While not strictly required for Turing-completeness, it is a powerful and desirable feature.

---

The algorithm for let-polymorphism is as follows:

1. **Generalization**. When encountering a let-statement, like `let NAME = VALUE in BODY`, generalize `NAME` by collecting all the **free type variables** (type variables that are not bound with the universal quantifier "∀") in `VALUE`. These free type variables are then bound using `∀`. For example:

```
NAME :: ∀T1,T3. typeof(VALUE)
```

2. **Instantiation**. When `NAME` is used, instantiate it by creating new type variables for each quantified variable (e.g., `T1`, `T3`). Substitute these variables with new ones in the body of `typeof(VALUE)`. For example, given the identity function:

```
let id = fun x -> x in id 5; id true
```

- When first encountering `id`, generalization gives `id :: ∀T. T -> T`.
- At its first use with `id 5`, `id` is instantiated as `id :: T1 -> T1`, which gets unified to `id :: int -> int`.
- At its second use with `id true`, a new instantiation occurs: `id :: T2 -> T2`, which gets unified to `id :: bool -> bool`.

Let-polymorphism enables functions like `id` to work with multiple types in a single program, increasing the expressiveness of the language without sacrificing anything.

#### Recursive Types
In the programming language literature, there are two distinct schools of thought for implementing recursive types: isorecursive, and equirecursive. They concern how a recursive type:

```
IntList = μX. {leaf:unit, cons:int * X}
```

relates to it's one-step **unfolding**:

```
IntListBody = {leaf:unit, cons:int * (μX. {leaf:unit, cons:int * X})}
```

> Note that `{}` denotes a variant type, with two type constructors: `leaf` and `cons`.

In the isorecursive case, these two types are isomorphic, and the **fold** / **unfold** operations witness this isomorphism. However, whenever we want to access the types of these constructors, we need to **unfold** it. In modern-day OCaml (that uses isorecursive typing), recursive variant types are folded at construction and are unfolded when being scrutinized by a match-statement. In our implementation, this is more-or-less what we do as well.

Equirecursive types would argue that `IntList` and `IntListBody` are actually equivalent, without **folding** / **unfolding**. We will not discuss them any further.

---

When encountering a match-statement, we can perform implicit unfolding as follows:

```
match x with
    | constructor1 var1 -> body1
    | constructor2 var2 -> body2
```

- Look up `constructor1` in the environment. It's type should be a function type: `T1 -> T`, where `T` is the type of the variant and `x :: T`.
- Bind `var1` to the left-hand side of the type of `constructor1` (the "unfolding" step): `var1 :: T1`.
- Compute the type of `body1` with `var1` bound.
- Repeat the same steps for `constructor2` and `body2`.
- Unify the resultant types of `body1` and `body2`.

---

#### A Digression on the Unit Type

Types can be thought of as sets. For example:

- The type `bool` represents a finite set with two elements: `true` and `false`.
- The type `int` is an infinite set: `{0, 1, -1, 2, -2, ...}` or `{x | x ∈ ℤ}`.

In `C`, the `void` type might seem like it has no values ("void" literally means "empty" in Latin), but conceptually, it represents a type with exactly **one** value. The only "value" of the `void` type can be thought of as an implicit element passed to or returned by a `void` function.

In functional programming, this idea corresponds to the `unit` type. The `unit` type has precisely one value: `() :: unit`. Functions with no parameters can be thought of as receiving this single element of `unit`. Functions are mappings from elements of input types to elements of output types, and `unit` serves as the simplest input / output type with exactly one element.

If you're still curious on this subject, Googling "programs as proofs and proofs as programs" can provide a cool perspective on the relationship between type theory and logic. It's a deep rabbit hole.

---

That wraps up the discussion on type inference and type checking. Everything we've covered is implemented in my compiler, so feel free to look at the code. There are different famous implementations of the "unification" algorithm: and the one I've used is equivalent to the famous "union-find" implementation. There are also some links in the comments that might be worth checking out.

# Alpha-normalization

As we've discussed earlier, alpha-normalization is the process of linearizing functional code by reducing expression to "atomic" syntax nodes, defined as:

- Constants like ints, bools, etc., and
- Variables.

For insight into what occurs during alpha-normalization: for example, when encountering an application, the left and right arguments are recursively alpha-normalized into atomic syntax nodes via let-statements before re-creating the application node. There are notable cases where we might not want to extract let-statements out of the body of a syntactic element.

- **if-expressions**. It's fine to extract the condition, but let-binding values from the true or false branches is not allowed. This ensures that side-effects can only occur when the specific branch is executed, preserving semantics for external C function calls that might have side-effects.
- **fun-expressions**. Let-binding values out of a function's body would create additional free variables in the body of the function, which complicates a later step called closure conversion. A function is also eventually hoisted to the global scope. Allowing let-binding extraction here would, philisophically, push code to the global scope, which is not allowed.

These constraints maintain logical consistency and are implemented in the compiler. Additional cases may exist but are handled appropriately. Refer to the code for more details and follow any links for further reading.

# Closure Conversion

Closure conversion eliminates free variables in functions by transforming them into closures. A closure is an object that contains a reference to which function we are closing over, and an array of values called the **environment** that provides values for the free variables in the function. The free variables are captured by value in a closure at the time a function is defined. In Python, a closure object might look like:

```python
@dataclass
class Closure:
    fun: Callable[[object, list[object]], object]
    env: list[object]

    def call(self, arg: object) -> object:
        return self.fun(arg, self.env)
```

This is more-or-less how it is implemented in C. For an example of some source code that will necessitate a closure, consider the following:

```
let x = 5 in
    let iAlwaysReturnX = fun y -> x in
        iAlwaysReturnX 1
```

In Python, this might be compiled to:

```python
def myFun(arg, env):
    return env[0]

x = 5
iAlwaysReturnX = Closure(fun=myFun, env=[x])
iAlwaysReturnX.call(1)
```

Here:
- `x` is a free variable inside of the function `fun y -> x`.
- Closure conversion transforms `fun y -> x` into a closure whose
    - Code: Points to a `fun arg env -> env[0]`.
    - Environment: Captures the value of `x` (in this case, `[x]`).
- Free variables in the function bdoy are replaced by indices into the closure's environment `x -> env[0]`.

Whenever we have to evaluate a function application, then, we can assume the left-hand side is already a closure. In the Python example, when we'd uncover `iAlwaysReturnX 1`, we'd execute: `iAlwaysReturnX.call(1)`. This assumption holds even if the function has no free variables, because then it makes compilation easier: the environment in that case would just be empty, and nothing inside that function would try to index into the environment array.

The result is code whose functions are all free of external dependencies, which is crucial for the next step of code generation.

# Code Generation

Generating code from the newly-transformed AST at this point is mainly a straightforward process, but there is still some stuff to talk about.

---

Variant types are implemented in the target language C via a structure with a `int type;` field. When you declare a variant type with constructors `Leaf` and `Cons`, those arguments are stored in this variant structure with either `0` or `1` as the value of the `type` field. That value depends on the index of that constructor in the list of constructors for that variant type: in our case, `Leaf` maps to `0`; and `Cons` maps to `1`. Then, when we encounter a match-statement, we generate a `switch` statement examining the value of that `type` field and then jump to the appropriate case. Besides knowing which index a constructor maps to, we can remove the variant type declarations from the code tree --- it is in a sense, metadata, and does not really mean anything to be generated in the target language.

---

There is an extensive standard library that was originally "injected" in the type inference step. Now, to provide the definitions for those standard library variables, we implement them as static variables in C whose values are closures, pointing to the correct code. I implemented these all in C and it was really interesting to create code in such a way that the compiler could use it. For example, for `geti :: unit -> int` (retrieving an integer from the user), we have, in C:

```
pluh_obj_t pluh_rt_geti(pluh_obj_t o, pluh_env_t *e)
{
    int i = 0;
    scanf("%d", &i);
    return (pluh_obj_t)(intptr_t)(i);
}
```

The closure is generated in a `pluh_init` function:

```
geti = pluh_closure_create((pluh_obj_t)(pluh_rt_geti), 0);
```

Note that that syntax for `pluh_closure_create` takes the (1) function we are pointing to, (2) the length of the evironment, and (3) the environment values themselves. There are no free variables in our runtime ("rt") `geti` function, so it's `0`.

---

There are probably more things to talk about here but this article is already getting long. Again, for implementation details, see the code --- each stage of the compilation process is compartmentalized into a single file.

# Conclusion

That's it! My brain is fried after writing all of that, so thanks for sticking around to the end. If I ever feel like it, I might come back and flesh out some of these explanations further. Oh, and I also chose "pluh" for the name of the language because it's funny. See you later!

# Footnotes

\*\*. The ability to perform similar tasks in both functional and procedural languages can be traced back to the Church-Turing Thesis. In the early 20th century, Alonzo Church developed the "lambda calculus" which underpins functional programming. His student, Alan Turing, created the "Turing machine," a model resembling procedural programming. The Church-Turing Thesis later demonstrated that these paradigms are equivalent in expressive power.
