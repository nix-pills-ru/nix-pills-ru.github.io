> # The Basics of the Language

# Основы языка

> Welcome to the fourth Nix pill.
> In the [previous article](#enter-environment) we learned about Nix environments.
> We installed software as a user, managed their profile, switched between generations, and queried the Nix store.
> Those are the very basics of system administration using Nix.

Добро пожаловать в четвёртую пилюлю Nix.
В [предыдущей статье](03-enter-environment.md) мы познакомились с окружениями Nix.
Вы устанавливали программы из под пользователя, управляли их профилем, переключались между поколениями, и писали запросы к хранилищу Nix.
Это основы системного администрирования с помощью Nix.

> The [Nix language](https://nixos.org/manual/nix/stable/expressions/expression-language.html) is used to write expressions that produce derivations.
> The [nix-build](https://nixos.org/manual/nix/stable/command-ref/nix-build.html) tool is used to build derivations from an expression.
> Even as a system administrator that wants to customize the installation, it's necessary to master Nix.
> Using Nix for your jobs means you get the features we saw in the previous articles for free.

[Язык Nix](https://nixos.org/manual/nix/stable/expressions/expression-language.html) используется для написания выражений, которые конструируют порождения.
Для построения порождений из выражений используется утилита [nix-build](https://nixos.org/manual/nix/stable/command-ref/nix-build.html).

> The syntax of Nix is quite unfamiliar, so looking at existing examples may lead you to think that there's a lot of magic happening.
> In reality, it's mostly about writing utility functions to make things convenient.

Синтаксис Nix достаточно непривычный, так что исследование существующих примеров может заставить вас думать, что здесь замешана какая-то магия.ю
На самом деле речь в основном идёт о написании служебных функций, чтобы делать вещи удобными.

> On the other hand, the same syntax is great for describing packages, so learning the language itself will pay off when writing package expressions.

С другой стороны, этот синтаксис прекрасно подходит для описания пакетов, так что изучение языка окупится при написании пакетных выражений.

> [!IMPORTANT]
> In Nix, everything is an expression, there are no statements. This is common in functional languages.

[!IMPORTANT]
В Nix всё является выражением, там нет операторов. Это обычное дело в функциональных языках.

> [!IMPORTANT]
> Values in Nix are immutable.

[!IMPORTANT]
Значения в Nix неизменяемые (иммутабельные).

> ## Value types

## Типы значений

> Nix 2.0 contains a command named `nix repl` which is a simple command line tool for playing with the Nix language.
> In fact, Nix is a [pure, lazy, functional language](https://nixos.org/manual/nix/stable/expressions/expression-language.html), not only a set of tools to manage derivations.
> The `nix repl` syntax is slightly different to Nix syntax when it comes to assigning variables, but it shouldn't be confusing so long as you bear it in mind.
> I prefer to start with `nix repl` before cluttering your mind with more complex expressions.

В Nix 2.0 tcnm команда `nix repl`, это простая утилита командной строки для экспериментирования с языком Nix.
Фактически, Nix — это [чистый ленивый функциональный язык](https://nixos.org/manual/nix/stable/expressions/expression-language.html), а не только набор утилит для работы с порождениями.
Синтаксис `nix repl` немного отличается от синтаксиса Nix, когда речь заходит о присваивании переменных, но это не должно сбивать вас с толку, пока вы помните об этом.
Я предпочитаю начать с `nix repl`, прежде чем загромождать вашу голову более сложными выражениями.

> Launch `nix repl`. First of all, Nix supports basic arithmetic operations: `+`, `-`, `*` and `/`.
> (To exit `nix repl`, use the command `:q`.
Help is available through the `:?` command.)

Запустите `nix repl`. Прежде всего, Nix поддерживает основные арифметические операции: `+`, `-`, `*` и `/`.
(Чтобы выйти из `nix repl`, введите команду `:q`.
Команда `:?` выводит справку.)

```text
nix-repl> 1+3
4

nix-repl> 7-4
3

nix-repl> 3*2
6
```

> Attempting to perform division in Nix can lead to some surprises.

Попытка выполнить деление в Nix может привезти к небольшим сюрпризам.

```text
nix-repl> 6/3
/home/nix/6/3
```

> What happened?
> Recall that Nix is not a general purpose language, it's a domain-specific language for writing packages.
> Integer division isn't actually that useful when writing package expressions.
> Nix parsed `6/3` as a relative path to the current directory.
> To get Nix to perform division instead, leave a space after the `/`.
> Alternatively, you can use `builtins.div`.

Что произошло?
Вспомним, что Nix не является языком общего назначения, это предметно-ориентированный язык для написания пакетов.
Деление чисел не является настолько полезным при написании пакетных выражений.
Для Nix `6/3` — это путь, построенный относительно текущего каталога.
Чтобы заставить Nix выполнить деление, добавьте пробел после `/`.
В качестве альтернативы вы можете использовать `builtins.div`.

```text
nix-repl> 6/ 3
2

nix-repl> builtins.div 6 3
2
```

> Other operators are `||`, `&&` and `!` for booleans, and relational operators such as `!=`, `==`, `<`, `>`, `<=`, `>=`. In Nix, `<`, `>`, `<=` and `>=` are not much used.
> There are also other operators we will see in the course of this series.

Другие операторы — это `||`, `&&` и `|` для булевых значений, и операторы сравнения, такие как `!=`, `==`, `<`, `>`, `<=`, `>=`. В Nix `<`, `>`, `<=` and `>=` используются нечасто.

> Nix has integer, floating point, string, path, boolean and null [simple](https://nixos.org/manual/nix/stable/expressions/language-values.html) types.
> Then there are also lists, sets and functions.
> These types are enough to build an operating system.

В Nix есть такие [простые](https://nixos.org/manual/nix/stable/expressions/language-values.html) типы: целые числа, числа с плавающей запятой, строки, пути, булевы значения и null.
Кроме того, есть списки, множества и функции.
Этих типов достаточно, чтобы собрать операционную систему.

> Nix is strongly typed, but it's not statically typed.
> That is, you cannot mix strings and integers, you must first do the conversion.

Nix является сильно типизированным, но не статически типизированным языком.
То есть, вы не можете смешивать строки и целые числа, сначала вы должны выполнить преобразование типа.

> As demonstrated above, expressions will be parsed as paths as long as there's a slash not followed by a space.
> Therefore to specify the current directory, use `./.`
> In addition, Nix also parses urls specially.

Как было показано выше, выражения считаются путями, пока вы не вставили пробел после символа деления.
Поэтому, чтобы указать текущий каталог, испоьлзуйте `./.`
В дополнение, Nix также умеет распознавать url'ы.

> Not all urls or paths can be parsed this way.
> If a syntax error occurs, it's still possible to fallback to plain strings.
> Literal urls and paths are convenient for additional safety.

Не все url'ы или пути могут быть распознаны обычным образом.
Если возникает ошибка распознавания, вы всегда можете вернуться к обычным строкам.
Строковые url'ы и пути удобны для дополнительной безопасности.

> ## Identifier

## Идентификатор

> There's not much to say here, except that dash (`-`) is allowed in identifiers.
> That's convenient since many packages use dash in their names.
> In fact:

Идентификаторы в Nix такие же, как в других языках, за исключением того, что позволяют использовать дефис (`-`).
Это удобно — имея много пакетов, использовать дефис в имени.
Фактически:

```text
nix-repl> a-b
error: undefined variable `a-b' at (string):1:1
nix-repl> a - b
error: undefined variable `a' at (string):1:1
```

> As you can see, `a-b` is parsed as identifier, not as a subtraction.

Как видите, `a-b` распознаётся как идентификатор, а не как деление.

> ## Strings

## Строки

> It's important to understand the syntax for strings.
> When learning to read Nix expressions, you may find dollars (`$`) ambiguous, but they are very important.
> Strings are enclosed by double quotes (`"`), or two single quotes (`''`).

Важно разобраться с синтаксисом строк.
Читая выражения Nix, вы можете посчитать знаки доллара (`$`) двусмысленными, но они очень важны.
Строки заключаются в двойные кавычки (`"`) или в пару одиночных кавычек (`''`).

> In other languages like Python you can also use single quotes for strings (e.g. `'foo'`), but not in Nix.

В других языках, скажем, в Python, вы также можете заключать строки в одиночные кавычки (например, `'foo'`), но не в Nix.

> It's possible to [interpolate](https://nixos.org/manual/nix/stable/expressions/language-values.html) whole Nix expressions inside strings with the `${...}` syntax and only that syntax, not `$foo` or `{$foo}` or anything else.

Можно [интерполировать](https://nixos.org/manual/nix/stable/expressions/language-values.html) целые выражения Nix внутри строк с помощью синтаксиса `${...}`, и только его, ни `$foo`, ни `{$foo}`, ни какого-то другого.

```text
nix-repl> foo = "strval"
nix-repl> "$foo"
"$foo"
nix-repl> "${foo}"
"strval"
nix-repl> "${2+3}"
error: cannot coerce an integer to a string, at (string):1:2
```

> Note: ignore the `foo = "strval"` assignment, special syntax in `nix repl`.

Обратите внимание: присваивание `foo = "strval"` — это специальный синтаксис, доступный в `nix repl` и недоступный в обычном языке.

> As said previously, you cannot mix integers and strings.
> You need to explicitly include conversions.
> We'll see this later: function calls are another story.

Как говорилось ранее, вы не можете смешивать целые числа и строки.
Вы должны в явном виде привести тип.
Мы вернёмся к этому позже: вызовы функций это ещё одна история на потом.

> Using the syntax with two single quotes is useful for writing double quotes inside strings without needing to escape them:

Используя строки, заключённые в пару одиночных кавычек, можно писать двойные кавычки внутри строк без необходимости их экранировать.

```text
nix-repl> ''test " test''
"test \" test"
nix-repl> ''${foo}''
"strval"
```

> Escaping `${...}` within double quoted strings is done with the backslash.
> Within two single quotes, it's done with `''`:

Экранирование `${...}` в строках с двойными кавычками делается с помощью обратной косой линии (бекслеша).
В строках с парой одиночных кавычек это делается с помощью `''`:

```text
nix-repl> "\${foo}"
"${foo}"
nix-repl> ''test ''${foo} test''
"test ${foo} test"
```

> ## Lists

## Списки

> Lists are a sequence of expressions delimited by space (*not* comma):

Списки это последовательности выражений, разделённые пробелом (*не* запятой):

```text
nix-repl> [ 2 "foo" true (2+3) ]
[ 2 "foo" true 5 ]
```

> Lists, like everything else in Nix, are immutable.
> Adding or removing elements from a list is possible, but will return a new list.

Списки, как и всё в Nix, иммутабельны.
Добавление или удаление элементов в списке возможно, но вернёт новый список.

> ## Attribute sets

## Множества атрибутов

> An attribute set is an association between string keys and Nix values.
> Keys can only be strings.
> When writing attribute sets you can also use unquoted identifiers as keys.

Множество атрибутов — это ассоциативный массив со строковыми ключами и значениями Nix.
Ключи могут быть только строками.
Записывая выражение с множеством атрибутов, вы можете использовать идентификаторы без кавычек в качестве ключей.

```text
nix-repl> s = { foo = "bar"; a-b = "baz"; "123" = "num"; }
nix-repl> s
{ "123" = "num"; a-b = "baz"; foo = "bar"; }
```

> For those reading Nix expressions from nixpkgs: do not confuse attribute sets with argument sets used in functions.

Для тех, кто читает выражения Nix из nixpkgs: не путайте множества атрибутов с множествами аргументов, используемых в функциях.

> To access elements in the attribute set:

Для доступа к элементам в множестве атрибутов:

```text
nix-repl> s.a-b
"baz"
nix-repl> s."123"
"num"
```

> Yes, you can use strings to address keys which aren't valid identifiers.

Да, вы можете использовать строки для доступа к ключам, которые не являются правильными идентификаторами.

> Inside an attribute set you cannot normally refer to elements of the same attribute set:

Внутри множества атрибутов, вы не можете обычным образом ссылаться на элементы или на само множество:

```text
nix-repl> { a = 3; b = a+4; }
error: undefined variable `a' at (string):1:10
```

> To do so, use [recursive attribute sets](https://nixos.org/manual/nix/stable/expressions/language-constructs.html#recursive-sets):

Но вы можете делать это, используя [рекурсивные множества атрибутов](https://nixos.org/manual/nix/stable/expressions/language-constructs.html#recursive-sets):

```text
nix-repl> rec { a = 3; b = a+4; }
{ a = 3; b = 7; }
```

> This is very convenient when defining packages, which tend to be recursive attribute sets.

Это очень удобно при описании пакетов, которые имеют тенденцию быть рекурсивными множествами атрибутов.

> ## If expressions

## Выражения 'if'

> These are expressions, not statements.

Это всё ещё выражения, не операторы.

```text
nix-repl> a = 3
nix-repl> b = 4
nix-repl> if a > b then "yes" else "no"
"no"
```

> You can't have only the `then` branch, you must specify also the `else` branch, because an expression must have a value in all cases.

Вы не можете записать только ветку `then`, вы обязаны записать также ветку `else`, потому что выражение должно иметь значение при любом раскладе.

> ## Let expressions

## Выражения 'let'

> This kind of expression is used to define local variables for inner expressions.

Этот вид выражений используется, чтобы определить локальные переменные для внутренних выражений.

```text
nix-repl> let a = "foo"; in a
"foo"
```

> The syntax is: first assign variables, then `in`, then an expression which can use the defined variables.
> The value of the whole `let` expression will be the value of the expression after the `in`.

Синтаксис такой: сначала назначаем переменные, затем пишем ключевое слово `in`, затем выражение, которое может содержать определённые переменные.
Значением всего выражения `let` станет значение выражения после `in`.

```text
nix-repl> let a = 3; b = 4; in a + b
7
```

> Let's write two `let` expressions, one inside the other:

Давайте запишем два выражения `let`, одно внутри другого:

```text
nix-repl> let a = 3; in let b = 4; in a + b
7
```

> With `let` you cannot assign twice to the same variable.
> However, you can shadow outer variables:

С помощью `let` нельзя назначить переменной другое значение.
Однако, можно перекрывать внешние переменные:

```text
nix-repl> let a = 3; a = 8; in a
error: attribute `a' at (string):1:12 already defined at (string):1:5
nix-repl> let a = 3; in let a = 8; in a
8
```

> You cannot refer to variables in a `let` expression outside of it:

Вы не может ссылаться на переменные в выражении `let` снаружи:

```text
nix-repl> let a = (let c = 3; in c); in c
error: undefined variable `c' at (string):1:31
```

> You can refer to variables in the `let` expression when assigning variables, like with recursive attribute sets:

Вы можете ссылаться на переменные в выражении `let`, когда назначаете значения другим перменным, как в рекурсивных множествах атрибутов.

```text
nix-repl> let a = 4; b = a + 5; in b
9
```

> So beware when you want to refer to a variable from the outer scope, but it's also defined in the current let expression.
> The same applies to recursive attribute sets.

В общем, остерегайтесь ситуации, когда вы хотите сослаться на внешнюю переменную, но переменная с таким же именем также определена в текущем выражении `let`.

> ## With expression

## Выражения 'with'

> This kind of expression is something you rarely see in other languages.
> You can think of it like a more granular version of `using` from C++, or `from module import *` from Python.
> You decide per-expression when to include symbols into the scope.

Этот тип выражений нечасто можно встретить в других языках.
Вы можете думать о нём, как о более детализированной версии оператора `using` из C++, или `from module import*` из Python.
Вы можете включить перменные в текущую область действия.

```text
nix-repl> longName = { a = 3; b = 4; }
nix-repl> longName.a + longName.b
7
nix-repl> with longName; a + b
7
```

> That's it, it takes an attribute set and includes symbols from it in the scope of the inner expression.
> Of course, only valid identifiers from the keys of the set will be included.
> If a symbol exists in the outer scope and would also be introduced by the `with`, it will *not* be shadowed.
> You can however still refer to the attribute set:

Оператор берёт множество атрибутов и включает символы из него в область видимости самого вложенного выражения.
Естественно, только корректные иднетификаторы из множества попадают в область видимости.
Если символ существует во внешней области и также вводится с помощью `with`, он *не* будет перекрыт.
В любом случае, вы можете и далее ссылаться на множество атрибутов:

```text
nix-repl> let a = 10; in with longName; a + b
14
nix-repl> let a = 10; in with longName; longName.a + b
7
```

> ## Laziness

## Ленивые вычисления

> Nix evaluates expressions only when needed.
> This is a great feature when working with packages.

Nix вычисляет выражения только когда это нужно.
Это отличная особенность, нужная при работе с пакетами.

```text
nix-repl> let a = builtins.div 4 0; b = 6; in b
6
```

> Since `a` is not needed, there's no error about division by zero, because the expression is not in need to be evaluated.
> That's why we can have all the packages defined on demand, yet have access to specific packages very quickly.

Поскольку значение `a` не требуется, не будет ошибки деления на ноль, потому что выражение не надо вычислять.
Вот почему мы можем определять пакеты по мере надобности, при этом получать доступ к нужным пакетам очень быстро.

> ## Next pill...

## В следующей пилюле...

> ...we will talk about functions and imports.
> In this pill I've tried to avoid function calls as much as possible, otherwise the post would have been too long.

...поговорим о функциях и импорте.
В этой пилюле я по возможности старался избегать функций, поскольку иначе пост стал бы слишком большим.