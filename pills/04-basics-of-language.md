# The Basics of the Language

Welcome to the fourth Nix pill. In the [previous article](#enter-environment) we learned about Nix environments.
We installed software as a user, managed their profile, switched between generations, and queried the Nix store.
Those are the very basics of system administration using Nix.

The [Nix language](https://nixos.org/manual/nix/stable/expressions/expression-language.html) is used to write expressions that produce derivations.
The [nix-build](https://nixos.org/manual/nix/stable/command-ref/nix-build.html) tool is used to build derivations from an expression.
Even as a system administrator that wants to customize the installation, it's necessary to master Nix.
Using Nix for your jobs means you get the features we saw in the previous articles for free.

The syntax of Nix is quite unfamiliar, so looking at existing examples may lead you to think that there's a lot of magic happening.
In reality, it's mostly about writing utility functions to make things convenient.

On the other hand, the same syntax is great for describing packages, so learning the language itself will pay off when writing package expressions.

<div class="important">

In Nix, everything is an expression, there are no statements. This is common in functional languages.

</div>

<div class="important">

Values in Nix are immutable.

</div>

## Value types

Nix 2.0 contains a command named `nix repl` which is a simple command line tool for playing with the Nix language.
In fact, Nix is a [pure, lazy, functional language](https://nixos.org/manual/nix/stable/expressions/expression-language.html), not only a set of tools to manage derivations.
The `nix repl` syntax is slightly different to Nix syntax when it comes to assigning variables, but it shouldn't be confusing so long as you bear it in mind.
I prefer to start with `nix repl` before cluttering your mind with more complex expressions.

Launch `nix repl`. First of all, Nix supports basic arithmetic operations: `+`, `-`, `*` and `/`.
(To exit `nix repl`, use the command `:q`. Help is available through the `:?` command.)

Attempting to perform division in Nix can lead to some surprises.

What happened?
Recall that Nix is not a general purpose language, it's a domain-specific language for writing packages.
Integer division isn't actually that useful when writing package expressions.
Nix parsed `6/3` as a relative path to the current directory.
To get Nix to perform division instead, leave a space after the `/`.
Alternatively, you can use `builtins.div`.

Other operators are `||`, `&&` and `!` for booleans, and relational operators such as `!=`, `==`, `<`, `>`, `<=`, `>=`. In Nix, `<`, `>`, `<=` and `>=` are not much used.
There are also other operators we will see in the course of this series.

Nix has integer, floating point, string, path, boolean and null [simple](https://nixos.org/manual/nix/stable/expressions/language-values.html) types.
Then there are also lists, sets and functions.
These types are enough to build an operating system.

Nix is strongly typed, but it's not statically typed.
That is, you cannot mix strings and integers, you must first do the conversion.

As demonstrated above, expressions will be parsed as paths as long as there's a slash not followed by a space.
Therefore to specify the current directory, use `./.`
In addition, Nix also parses urls specially.

Not all urls or paths can be parsed this way.
If a syntax error occurs, it's still possible to fallback to plain strings.
Literal urls and paths are convenient for additional safety.

## Identifier

There's not much to say here, except that dash (`-`) is allowed in identifiers.
That's convenient since many packages use dash in their names.
In fact:

As you can see, `a-b` is parsed as identifier, not as a subtraction.

## Strings

It's important to understand the syntax for strings. When learning to read Nix expressions, you may find dollars (`$`) ambiguous, but they are very important.
Strings are enclosed by double quotes (`"`), or two single quotes (`''`).

In other languages like Python you can also use single quotes for strings (e.g. `'foo'`), but not in Nix.

It's possible to [interpolate](https://nixos.org/manual/nix/stable/expressions/language-values.html)
whole Nix expressions inside strings with the `${...}` syntax and only that syntax, not `$foo` or `{$foo}` or anything else.

Note: ignore the `foo = "strval"` assignment, special syntax in `nix repl`.

As said previously, you cannot mix integers and strings.
You need to explicitly include conversions.
We'll see this later: function calls are another story.

Using the syntax with two single quotes is useful for writing double quotes inside strings without needing to escape them:

Escaping `${...}` within double quoted strings is done with the backslash.
Within two single quotes, it's done with `''`:

## Lists

Lists are a sequence of expressions delimited by space (*not* comma):

Lists, like everything else in Nix, are immutable.
Adding or removing elements from a list is possible, but will return a new list.

## Attribute sets

An attribute set is an association between string keys and Nix values.
Keys can only be strings.
When writing attribute sets you can also use unquoted identifiers as keys.

For those reading Nix expressions from nixpkgs: do not confuse attribute sets with argument sets used in functions.

To access elements in the attribute set:

Yes, you can use strings to address keys which aren't valid identifiers.

Inside an attribute set you cannot normally refer to elements of the same attribute set:

To do so, use [recursive attribute sets](https://nixos.org/manual/nix/stable/expressions/language-constructs.html#recursive-sets):

This is very convenient when defining packages, which tend to be recursive attribute sets.

## If expressions

These are expressions, not statements.

You can't have only the `then` branch, you must specify also the `else` branch, because an expression must have a value in all cases.

## Let expressions

This kind of expression is used to define local variables for inner expressions.

The syntax is: first assign variables, then `in`, then an expression which can use the defined variables.
The value of the whole `let` expression will be the value of the expression after the `in`.

Let's write two `let` expressions, one inside the other:

With `let` you cannot assign twice to the same variable.
However, you can shadow outer variables:

You cannot refer to variables in a `let` expression outside of it:

You can refer to variables in the `let` expression when assigning variables, like with recursive attribute sets:

So beware when you want to refer to a variable from the outer scope, but it's also defined in the current let expression.
The same applies to recursive attribute sets.

## With expression

This kind of expression is something you rarely see in other languages.
You can think of it like a more granular version of `using` from C++, or `from module import *` from Python.
You decide per-expression when to include symbols into the scope.

That's it, it takes an attribute set and includes symbols from it in the scope of the inner expression.
Of course, only valid identifiers from the keys of the set will be included.
If a symbol exists in the outer scope and would also be introduced by the `with`, it will *not* be
shadowed.
You can however still refer to the attribute set:

## Laziness

Nix evaluates expressions only when needed.
This is a great feature when working with packages.

Since `a` is not needed, there's no error about division by zero, because the expression is not in need to be evaluated.
That's why we can have all the packages defined on demand, yet have access to specific packages very quickly.

## Next pill...

...we will talk about functions and imports.
In this pill I've tried to avoid function calls as much as possible, otherwise the post would have been too long.
