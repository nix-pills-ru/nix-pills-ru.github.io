> # Functions and Imports

# Функции и импорт

> Welcome to the fifth Nix pill. In the previous [fourth pill](#basics-of-language) we touched the Nix language for a moment.
> We introduced basic types and values of the Nix language, and basic expressions such as `if`, `with` and `let`.
> I invite you to re-read about these expressions and play with them in the repl.

Добро пожаловать в пятую пилюлю Nix. В предыдущей [четвёртой пилюле](04-basics-of-language.md) на мгновенье прикоснулись к языку Nix.
Мы рассказали про базовые типы и значения языка Nix, и базовыми выражениями, такими как `if`, `with` и `let`.
Я приглашаю вас перечитать про выражения и поиграться с ними в REPL.

> Functions help to build reusable components in a big repository like [nixpkgs](https://github.com/NixOS/nixpkgs/).
> The Nix manual has a [great explanation of functions](https://nixos.org/manual/nix/stable/expressions/language-constructs.html#functions).
> Let's go: pill on one hand, Nix manual on the other hand.

Функции помогают строить повторно используемые компоненты в больших хранилищах, таких как [nixpkgs](https://github.com/NixOS/nixpkgs/).
Руководство по языку Nix имеет [великолепное объяснение функций](https://nixos.org/manual/nix/stable/expressions/language-constructs.html#functions).
Поехали: пилюля в одной руке и руководство по Nix — в другой.

> I remind you how to enter the Nix environment: `source ~/.nix-profile/etc/profile.d/nix.sh`

Я напоминаю вам, как запустить среду Nix: `source ~/.nix-profile/etc/profile.d/nix.sh`

> ## Nameless and single parameter

## Безымянные с единственным параметром

> Functions are anonymous (lambdas), and only have a single parameter.
> The syntax is extremely simple.
> Type the parameter name, then "`:`", then the body of the function.

Анонимные функции (лямбды) с одним параметром.
Синтаксис экстремально прост.
Введите имя параметра, затем "`:`", затем тело функции.

```text
nix-repl> x: x*2
«lambda»
```

> So here we defined a function that takes a parameter `x`, and returns `x*2`.
> The problem is that we cannot use it in any way, because it's unnamed... joke!

Подобным образом мы определили функцию, которая принимает параметр `x` и возвращает `x*2`.
Проблема в том, что мы всё равно не можем её использовать, потому что у неё нет имени... шутка!

> We can store functions in variables.

Мы можем хранить функции в переменных.

```text
nix-repl> double = x: x*2
nix-repl> double
«lambda»
nix-repl> double 3
6
```

> As usual, please ignore the special syntax for assignments inside `nix repl`.
> So, we defined a function `x: x*2` that takes one parameter `x`, and returns `x*2`.
> This function is then assigned to the variable `double`.
> Finally we did our first function call: `double 3`.

Как я говорил раньше, пожалуйста, игнорируйте особый синтаксис присваивания в `nix repl`.
Так, мы определили функцию `x: x*2`, которая принимает один параметр `x` и возвращает `x*2`.
Затем эта функции присваевается переменной `double`.
Наконец, мы делаем наш первый вызов функции: `double 3`. 

> *Big note:* it's not like many other programming languages where you write `double(3)`.
> It really is `double 3`.

*Важное примечание:* это отличается от многих других языков программирования, где вы пишите `double(3)`.
Здесь это действительно `double 3`.

> In summary: to call a function, name the variable, then space, then the argument.
> Nothing else to say, it's as easy as that.

В итоге: чтобы вызывать функцию, напишите её имя, затем пробел, затем аргумент.
Больше сказать нечего, настолько всё просто.

> ## More than one parameter

## Когда параметров больше одного

> How do we create a function that accepts more than one parameter?
> For people not used to functional programming, this may take a while to grasp.
> Let's do it step by step.

Как записать функцию, которая принимает больше одного параметра?
Для тех, кто не сталкивался с функциональным программированием, может потребоваться время, чтобы разобраться.
Давайте сделаем это шаг за шагом.

```text
nix-repl> mul = a: (b: a*b)
nix-repl> mul
«lambda»
nix-repl> mul 3
«lambda»
nix-repl> (mul 3) 4
12
```

> We defined a function that takes the parameter `a`, the body returns another function.
> This other function takes a parameter `b` and returns `a*b`.
> Therefore, calling `mul 3` returns this kind of function: `b: 3*b`.
> In turn, we call the returned function with `4`, and get the expected result.

Мы определили функцию, которая принимает параметр `a` и возвращает другую функцию.
Эта другая функция принимает параметр `b` и возвращает `a*b`.
Таким образом, вызов `mul 3` вернёт функцию `b: 3*b`.
Затем мы вызываем полученную функцию с параметром `4` и получаем ожидаемый результат.

> You don't have to use parentheses at all, Nix has sane priorities when parsing the code:

Вам не обязательно использовать скобки вообще, поскольку Nix при разборе выражений учитывает приоритеты:  

```text
nix-repl> mul = a: b: a*b
nix-repl> mul
«lambda»
nix-repl> mul 3
«lambda»
nix-repl> mul 3 4
12
nix-repl> mul (6+7) (8+9)
221
```

> Much more readable, you don't even notice that functions only receive one argument.
> Since the argument is separated by a space, to pass more complex expressions you need parentheses.
> In other common languages you would write `mul(6+7, 8+9)`.

Гораздо проще читается, вы даже можете не обратить внимания, что функции получают только один аргумент.
Поскольку аргументы разделяются пробелом, вам нужны скобки, чтобы передавать более сложные выражения.
В других языках вы бы написали `mul(6+7, 8+9)`.

> Given that functions have only one parameter, it is straightforward to use **partial application**:

Учитывая, что функции имеют только один параметр, несложно использовать **частичное применение**:

```text
nix-repl> foo = mul 3
nix-repl> foo 4
12
nix-repl> foo 5
15
```

> We stored the function returned by `mul 3` into a variable foo, then reused it.

Мы сохранили функцию, которую вернула `mul 3` в переменную `foo`, и затем несколько раз использовали её.

> ## Argument set

## Множество аргументов

> Now this is a very cool feature of Nix.
> It is possible to pattern match over a set in the parameter.
> We write an alternative version of `mul = a: b: a*b` first by using a set as argument, then using pattern matching.

А теперь — очень крутая возможность Nix.
Множество пераметров можно сопоставлять с образцом.
Мы напишем альтернативную версию `mul = a: b: a*b` сначал используя множество аргументов, а затем — сопоставление с образцом.

```text
nix-repl> mul = s: s.a*s.b
nix-repl> mul { a = 3; b = 4; }
12
nix-repl> mul = { a, b }: a*b
nix-repl> mul { a = 3; b = 4; }
12
```

> In the first case we defined a function that accepts a single parameter.
> We then access attributes `a` and `b` from the given set.
> Note how the parentheses-less syntax for function calls is very elegant in this case, instead of doing `mul({ a=3; b=4; })` in other languages.

В первом случае мы определили функцию, котоаря принимает один параметр.
Затем мы берём атрибуты `a` и `b` из этого множества.
Обратите внимание, что бесскобочный синтакис очень элегантен в этом случае, вместо вызова `mul({ a=3; b=4; })` в других языках.

> In the second case we defined an argument set.
> It's like defining a set, except without values.
> We require that the passed set contains the keys `a` and `b`.
> Then we can use those `a` and `b` in the function body directly.

Во втором случае мы определили множество аргументов.
Это похоже на определине множества, только без указания значений.
Мы требуем, чтобы переданное множество содержало ключи `a` и `b`.
Затем мы можем использовать эти `a` и `b` непосредственно в теле функции.

```text
nix-repl> mul = { a, b }: a*b
nix-repl> mul { a = 3; b = 4; c = 6; }
error: anonymous function at (string):1:2 called with unexpected argument `c', at (string):1:1
nix-repl> mul { a = 3; }
error: anonymous function at (string):1:2 called without required argument `b', at (string):1:1
```

> Only a set with exactly the attributes required by the function is accepted, nothing more, nothing less.

Функция принимает только множество с теми атрибутами, которые были записаны, ни более, и не менее.

> ## Default and variadic attributes

## Атрибуты по умолчанию и вариативные атрибуты

> It is possible to specify **default values** of attributes in the argument set:

Можно указывать **значения по умолчанию** для атрибутов в множестве аргументов:

```text
nix-repl> mul = { a, b ? 2 }: a*b
nix-repl> mul { a = 3; }
6
nix-repl> mul { a = 3; b = 4; }
12
```

> Also you can allow passing more attributes (**variadic**) than the expected ones:

Также вы можете позволить передавать больше атрибутов (**вариативных**), чем вам нужно:

```text
nix-repl> mul = { a, b, ... }: a*b
nix-repl> mul { a = 3; b = 4; c = 2; }
```

> However, in the function body you cannot access the "c" attribute.
> The solution is to give a name to the given set with the **@-pattern**:

Однако в теле функции вы не можете получить доступ к атрибуту `c`.
Решение в том, чтобы дать имя всему множеству, с помощью **@-образца**:

```text
nix-repl> mul = s@{ a, b, ... }: a*b*s.c
nix-repl> mul { a = 3; b = 4; c = 2; }
24
```

> That's it, you give a name to the whole parameter with name@ before the set pattern.

То есть, вы даёте имя всему параметру с помощью name@ перед образцом множества.

> Advantages of using argument sets:

Преимущества использования множеств аргументов:

> - Named unordered arguments: you don't have to remember the order of the arguments.
> - You can pass sets, that adds a whole new layer of flexibility and convenience.

- Именованые неупорядоченые аргументы: вы не должня запоминать порядок аргументов.
- Мы можете передвать множества, чтоба добавляет совершенно новый уровень гибкости и удобства.

> Disadvantages:

Недостатки:

> - Partial application does not work with argument sets.
>   You have to specify the whole attribute set, not part of it.

- Частичное применение не работает с множествами аргументов.
  Вы должны определить множество атрибутов целиком, нельзя определить только его часть.

> You may find similarities with [Python \*\*kwargs](https://docs.python.org/3/faq/programming.html#how-can-i-pass-optional-or-keyword-parameters-from-one-function-to-another).

Вы можете обнаружить сходство с [\*\*kwargs из языка Python](https://docs.python.org/3/faq/programming.html#how-can-i-pass-optional-or-keyword-parameters-from-one-function-to-another).

> ## Imports

## Импорт

> The `import` function is built-in and provides a way to parse a `.nix` file.
> The natural approach is to define each component in a `.nix` file, then compose by importing these files.

Функция `import` является встроенный и предоставляет возможность подключать файлы `.nix`.
Естественный подход — определить каждый компонент в файле `.nix`, а затем соединить, импортируя эти файлы.

> Let's start with the bare metal.

Давайте начнём с голого железа. <!-- По-русски "выделенный сервер". В то же время устроявшийся оборот — голый сервер без всякого софта, что-то максимально простое. По смыслу "Давайте начнём с самого простого". Подыскать подходящий аналог среди русских оборотов. -->

`a.nix`:

```nix
3
```

`b.nix`:

```nix
4
```

`mul.nix`:

```nix
a: b: a*b
```

```text
nix-repl> a = import ./a.nix
nix-repl> b = import ./b.nix
nix-repl> mul = import ./mul.nix
nix-repl> mul a b
12
```

> Yes it's really that simple.
> You import a file, and it gets parsed as an expression.
> Note that the scope of the imported file does not inherit the scope of the importer.

Да, это действительно настолько просто.
Вы импортируете файл, и он превращается (<!-- компилируется -->) в выражение.
Обратите внимание, что в импортирующем файле нет доступа к переменным из импортируемого файла.

`test.nix`:

```nix
x
```

```text
nix-repl> let x = 5; in import ./test.nix
error: undefined variable `x' at /home/lethal/test.nix:1:1
```

> So how do we pass information to the module?
> Use functions, like we did with `mul.nix`.
> A more complex example:

Так как же нам передать информацию в модуль?
Используйте функции, как мы это делали с `mul.nix`.
Более сложный пример.

`test.nix`:

```nix
{ a, b ? 3, trueMsg ? "yes", falseMsg ? "no" }:
if a > b
  then builtins.trace trueMsg true
  else builtins.trace falseMsg false
```

```text
nix-repl> import ./test.nix { a = 5; trueMsg = "ok"; }
trace: ok
true
```

> Explaining:

Объяснение:

> - In `test.nix` we return a function.
>   It accepts a set, with default attributes `b`, `trueMsg` and `falseMsg`.
> - `builtins.trace` is a [built-in function](https://nixos.org/manual/nix/stable/expressions/builtins.html) that takes two arguments.
>   The first is the message to display, the second is the value to return.
>   It's usually used for debugging purposes.
> - Then we import `test.nix`, and call the function with that set.

- В `text.nix` вы возвращаем функцию.
  Она принимет множество с атрибутами по умолчанию `b`, `trueMsg` и `falseMsg`.
- `builtins.trace` — [встроенная функция](https://nixos.org/manual/nix/stable/expressions/builtins.html), которая принимает два аргумента.
  Первый — это сообщение для печати, второй — возвращаемое значение.
  Обычно она используется для отладки.
- Затем мы импортируем `text.nix` и вызываем функцию с указанным множеством.

> So when is the message shown? Only when it needs to be evaluated.

Так и когда же показывается сообщение? Только тогда, когда вычисляется соответсвующая ветвь кода.

> ## Next pill...

## В следующей пилюле...

> ...we will finally write our first derivation.

...мы, наконец, напишем своё первое порождение.
