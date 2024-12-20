> # Callpackage Design Pattern

# Паттерн проектирования `callPackage` (Вызов пакета)

> Welcome to the 13th Nix pill.
> In the previous [12th pill](#inputs-design-pattern), we introduced the first basic design pattern for organizing a repository of software.
> In addition, we packaged graphviz so that we had two packages to bundle into an example repository.

Добро пожаловать на тринадцатую пилюлю Nix.
В предыдущей [двенадцатой пилюле](12-inputs-design-pattern.md) мы рассказали о первом паттерне проектирования для организации программного репозитория.

> The next design pattern we will examine is called the `callPackage` pattern.
> This technique is extensively used in [nixpkgs](https://github.com/NixOS/nixpkgs), and it\'s the current de facto standard for importing packages in a repository.
> Its purpose is to reduce the duplication of identifiers between package derivation inputs and repository derivations.

Следующий паттерн проектирования, с которым мы познакомимся, называется `callPackage` (`Вызов пакета`).
Этот подход широко применяется в [nixpkgs](https://github.com/NixOS/nixpkgs) и в настоящее время является стандартом де-факто для импорта пакетов в репозитории.

> ## The callPackage convenience

## Удобство Вызова пакета

> In the previous pill, we demonstrated how the `inputs` pattern decouples packages from the repository.
> This allowed us to manually pass the inputs to the derivation; the derivation declares its inputs, and the caller passes the arguments.

В предыдущей пилюле мы продемонстрировали, как паттерн `Входящие` позволяет отвязать пакеты от репозитория.
Это позволило нам вручную передавать аргументы в деривацию; деривация объявляет свои входящие параметры, и вызывающая сторона передаёт аргументы.

> However, as with usual programming languages, there is some duplication of work: we declare parameter names and then we pass arguments, typically with the same name.
> For example, if we define a package derivation using the `inputs` pattern such as:

Тем не менее, как это часто бывает в языках программирования, есть некоторое дублирование работы: мы объявляем имена параметров, и затем мы передаём аргументы, как правило, с теми же именами.

```nix
{ input1, input2, ... }:
...
```

> we would likely want to bundle that package derivation into a repository via a an attribute set defined as something like:

> скорее всего, мы захотели бы присоединить эту деривацию к репозиторию через набор атрибутов, определенный как-то так:

```nix
rec {
  lib1 = import package1.nix { inherit input1 input2; };
  program2 = import package2.nix { inherit inputX inputY lib1; };
}
```

> There are two things to note.
> First, that inputs often have the same name as attributes in the repository itself.
> Second, that (due to the `rec` keyword), the inputs to a package derivation may be other packages in the repository itself.

Есть две вещи, на которые стоит обратить внимание.
Во-первых, эти входящие параметры часто имеют то же имя, что и атрибуты в самом репозитории.
Во-вторых, из-за ключевого слова `rec` входящие параметры деривации могут ссылаться на другие пакеты в репозитории.

> Rather than passing the inputs twice, we would prefer to pass those inputs from the repository automatically and allow for manually overriding defaults.

Вместо того, чтобы дважды передавать входящие параметры, мы бы предпочли, чтобы они передавались из репозитория автоматически, позволяя нам, в случае необходимости, переопределить их вручную.

> To achieve this, we will define a `callPackage` function with the following calling convention:

Чтобы достичь этого, мы определим функцию `callPackage` с таким набором параметров:

```nix
{
  lib1 = callPackage package1.nix { };
  program2 = callPackage package2.nix { someoverride = overriddenDerivation; };
}
```

> We want `callPackage` to be a function of two arguments, with the following behavior:

Нам надо, чтобы `callPackage` была функцией с двумя аргументами, которая выполняет следующие действия:

> -   Import the given expression contained in the file of the first argument, and return a function.
>     This function returns a package derivation that uses the inputs pattern.
> -   Determine the name of the arguments to the function (i.e., the names of the inputs to the package derivation).
> -   Pass default arguments from the repository set, and let us override those arguments if we wish to customize the package derivation.

- Импортирует выражение, заданное в файле, который передаётся в первом аргументе, и возвращает функцию.
  Эта функция возвращает деривацию пакета, которая использует паттерн Входящие.
- Определяет имена аргументов функции, то есть имена входящих параметров деривации.
- Передаёт значения аргументов по умолчанию из наборе репозитория, и позволяет переопределить эти аргументы, если мы хотим настроить деривацию пакета.

> ## Implementing `callPackage`

## Реализуем `callPackage`

> In this section, we will build up the `callPackages` pattern from scratch.
> To start, we need a way to obtain the argument names of a function (in this case, the function that takes \"inputs\" and produces a package derivation) at runtime.
> This is be cause we want to automatically pass such arguments.

В этом разделе мы реализуем паттерн `callPackage` с нуля.
Для начала нам нужно научиться получать имена аргументов функции (в нашем случае, функции, которая получает входящие параметры и создаёт деривацию пакета) во время выполнения.
Это нужно для того, чтобы передавать такие аргументы автоматически.

> Nix provides a builtin function to do this:

Nix для этого предоставляет встроенную функцию:

```nix
nix-repl> add = { a ? 3, b }: a+b
nix-repl> builtins.functionArgs add
{ a = true; b = false; }
```

> In addition to returning the argument names, the attribute set returned by `functionArgs` indicates whether or not the argument has a default value.
> For our purposes, we are only interested in the argument names; we do not care about the default values right now.

Вместе с именами аргументов, набор атрибутов, возвращаемый `functionArgs` содержит признак, есть ли у аргумента значение по умолчанию.
Для наших целей нам достаточно только имён аргументов; прямо сейчас нет смысла беспокоиться о значениях по умолчанию.

> The next step is to make `callPackage` automatically pass inputs to our package derivations based on the argument names we\'ve just obtained with `functionArgs`.

Следующий шаг — сделать так, чтобы `callPackage` автоматически передавал значения аргументов в наши деривации, основываясь на именах аргументов, которые мы узнали благодаря `functionArgs`.

> To do this, we need two things:

Для этого нам нужны две вещи:

> -   A package repository set containing package derivations that match the arguments names we\'ve obtained
> -   A way to obtain an auto-populated attribute set combining the package repository and the return value of `functionArgs`.

- Набор репозитория с деривациями пакетов, чьи имена совпадают с именам полученных нами аргументов.
- Способ автоматически собрать набор атрибутов, объединяющий репозиторий пакетов и значения, полученные из `functionArgs`.

> The former is easy: we just have to set our package derivation\'s inputs to be package names in a repository, such as `nixpkgs`.
> For the latter, Nix provides another builtin function:

Первое просто: достаточно в качестве имён аргументов использовать имена пакетов из репозитория, например, из `nixpkgs`.
Для второго Nix предоставляет ещё одну встроенную функцию:

```nix
nix-repl> values = { a = 3; b = 5; c = 10; }
nix-repl> builtins.intersectAttrs values (builtins.functionArgs add)
{ a = true; b = false; }
nix-repl> builtins.intersectAttrs (builtins.functionArgs add) values
{ a = 3; b = 5; }
```

> The `intersectAttrs` returns an attribute set whose names are the intersection of both arguments\' attribute names, with the attribute values taken from the second argument.

`intersectAttrs` возвращает набор атрибутов, имена которые являются пересечением имён атрибутов обоих аргументов, а значение получены из второго аргумента.

> This is all we need to do: we have obtained the argument names from a function, and populated these with an existing set of attributes.
> This is our simple implementation of `callPackage`:

Это всё, что нам нужно: мы получили имена аргументов из функции и заполнили их значениями из существующего набора атрибутов.
Вот наша простая реализация `callPackage`:


```nix
nix-repl> callPackage = set: f: f (builtins.intersectAttrs (builtins.functionArgs f) set)
nix-repl> callPackage values add
8
nix-repl> with values; add { inherit a b; }
8
```

> Let\'s dissect the above snippet:

Давайте проанализируем приведённый выше фрагмент:

> -   We define a `callPackage` variable which is a function.
> -   The first parameter to the `callPackage` function is a set of name-value pairs that may appear in the argument set of the function we wish to \"autocall\".
> -   The second parameter is the function to \"autocall\"
> -   We take the argument names of the function and intersect with the set of all values.
> -   Finally, we call the passed function `f` with the resulting intersection.

- Мы определяем переменную `callPackage`, которая на самом деле функция.
- Первый параметр функции `callPackage` — это набор пар имя-значение. Часть из них может совпадать с набором аргументов фукнции, которую мы вызовем.
- Второй параметр — функция, которую мы вызовем.
- Мы получаем имена аргументов функции и находим их пересечение с набором всех значений.
- Наконец, мы вызываем функцию `f` с полученным пересечением.

> In the snippet above, we\'ve also demonstrated that the `callPackage` call is equivalent to directly calling `add a b`.

Фрагмент выше демонстрирует, что вызов `callPackage` эквивалентен прямому вызову `add a b`.

> We achieved most of what we wanted: to automatically call functions given a set of possible arguments.
> If an argument is not found within the set we used to call the function, then we receive an error (unless the function has variadic arguments denoted with `...`, as explained in the [5th pill](#functions-and-imports)).

У нас получилось почти всё, что мы хотели: автоматически вызывать функции, передавая им набор аргументов.
Если аргумент не найден в наборе, который мы использовали для вызова функции, мы получим ошибку (если только у функции не переменное число аргументов, объявленных через `...`, как мы объясняли в [пятой таблетке](05-functions-and-imports.md)).

> The last missing piece is allowing users to override some of the parameters.
> We may not want to always call functions with values taken from the big set.
> Thus, we add a third parameter which takes a set of overrides:

Последний отсутствущий косок — дать пользователям возможность переопределять некоторые параметры.
Мы можем не хотеть всегда вызывать функции с параметрами, полученными из большого набора.
Поэтому мы добавляем третий параметр, который принимает набор переопределений:

```nix
nix-repl> callPackage = set: f: overrides: f ((builtins.intersectAttrs (builtins.functionArgs f) set) // overrides)
nix-repl> callPackage values add { }
8
nix-repl> callPackage values add { b = 12; }
15
```

> Apart from the increasing number of parentheses, it should be clear that we simply take a set union between the default arguments and the overriding set.

Не смотря на увеличение количества скобок, должно быть ясно, что мы всего навсего объединяем набор аргументов по умолчанию с набором переопределений.

> ## Using callPackage to simplify the repository

### Используем `callPackage`, чтобы упростить код репозитория

> Given our `callPackages`, we can simplify the repository expression in `default.nix`:

Имея на руках функцию `callPackage`, мы можем упростить выражение репозитория в `default.nix`:

```nix
let
  nixpkgs = import <nixpkgs> { };
  allPkgs = nixpkgs // pkgs;
  callPackage =
    path: overrides:
    let
      f = import path;
    in
    f ((builtins.intersectAttrs (builtins.functionArgs f) allPkgs) // overrides);
  pkgs = with nixpkgs; {
    mkDerivation = import ./autotools.nix nixpkgs;
    hello = callPackage ./hello.nix { };
    graphviz = callPackage ./graphviz.nix { };
    graphvizCore = callPackage ./graphviz.nix { gdSupport = false; };
  };
in
pkgs
```

> Let\'s examine this in detail:

Разберёмся, как это работает:

> -   The expression above defines our own package repository, which we call `pkgs`, that contains `hello` along with our two variants of `graphviz`.
> -   In the `let` expression, we import `nixpkgs`.
>     Note that previously, we referred to this import with the variable `pkgs`, but now that name is taken by the repository we are creating ourselves.
> -   We needed a way to pass `pkgs` to `callPackage` somehow.
>     Instead of returning the set of packages directly from `default.nix`, we first assign it to a `let` variable and reuse it in `callPackage`.
> -   For convenience, in `callPackage` we first import the file instead of calling it directly.
>     Otherwise we would have to write the `import` for each package.
> -   Since our expressions use packages from `nixpkgs`, in `callPackage` we use `allPkgs`, which is the union of `nixpkgs` and our packages.
> -   We moved `mkDerivation` into `pkgs` itself, so that it also gets passed automatically.

- Выражение выше определяет наш собственный репозиторий пакетов, который мы называем `pkgs`, который содержит пакет `hello` и два варианта `graphviz`.
- В выражении `let` мы импортируем `nixpkgs`.
  Обратите внимание, что раньше мы ссылались на это значение с помощью переменной `pkgs`, но теперь это имя занято репозиторием, который мы создаём.
- Нам нужен способ как-то передать `pkgs` в `callPackage`.
  Вместо того, чтобы напрямую вернуть набор пакетов из `default.nix`, мы сначала присваиваем его переменной в выражении `let`, что позволяет передать его в `callPackage`.
- В целях упрощения, в `callPackage` не получаем функцию в качестве аргумента, мы получаем файл и импоритируем функцию из файла.
  Иначе нам бы пришлось вручную импортировать каждый пакет.
- Поскольку наши выражения используют пакеты из `nixpkgs`, в `callPackage` мы описываем переменную `allPkgs`, в которую помещаем пакеты из `nixpkgs` и наши пакеты.
- Мы переместили функцию `mkDerivation` в `pkgs`, чтобы она автоматически передавалась в пакеты.

> Note how easily we overrode arguments in the case of `graphviz` without `gd`.
> In addition, note how easy it was to merge two repositories: `nixpkgs` and our `pkgs`!

Обратите внимание, как легко мы переопределили аргументы для создания `graphviz` без `gd`.
Кроме того, обратите внимание, как легко слить два репозитория: `nixpkgs` и наш `pkgs`!

> The reader should notice a magic thing happening.
> We\'re defining `pkgs` in terms of `callPackage`, and `callPackage` in terms of `pkgs`.
> That magic is possible thanks to lazy evaluation: `builtins.intersectAttrs` doesn\'t need to know the values in `allPkgs` in order to perform intersection, only the keys that do not require `callPackage` evaluation.

Читатель должен заметить магическую вещь, которая здесь происходит.
Чтобы определить `pkgs`, мы используем `callPackage`, а при определении `callPackage` мы используем `pkgs`.
Эта магия работает благодаря ленивым вычислениям: `builtins.intersectAttrs` не должен знать все значения из `allPkgs`, чтобы найти пересечение, а только ключи, которые не требуют вычисления `callPackage`.

> ## Conclusion

## Заключение

> The \"`callPackage`\" pattern has simplified our repository considerably.
> We were able to import packages that require named arguments and call them automatically, given the set of all packages sourced from `nixpkgs`.

Паттерн "`Вызов пакета`" значительно упростил код репозитория.
Мы смогли импортировать пакеты с именованными аргументами и вызывать их автоматически, имея набор всех пакетов из `nixpkgs`.

> We\'ve also introduced some useful builtin functions that allows us to introspect Nix functions and manipulate attributes.
> These builtin functions are not usually used when packaging software, but rather act as tools for packaging.
> They are documented in the [Nix manual](https://nixos.org/manual/nix/stable/expressions/builtins.html).

Также мы познакомились с некоторыми полезными встроенными функциями, которые позволяют нам получать информацию о функциях Nix и манипулировать атрибутами.
Эти встроенные функции обычно используются не для установки пакета, а, скорее, как инструменты при установке.
Они описаны в [Руководстве по Nix](https://nixos.org/manual/nix/stable/expressions/builtins.html).

> Writing a repository in Nix is an evolution of writing convenient functions for combining the packages.
> This pill demonstrates how Nix can be a generic tool to build and deploy software, and how suitable it is to create software repositories with our own conventions.

Создание репозитория в Nix — это эволюция разработки удобных функций для комбинирования пакетов.
Эта пилюля демострирует как Nix может быть универсальным инструментом для сборки и развёртывания программного обеспечения, и как он подходит для создания репозиториев программ с нашими собственными соглашениями.

> ## Next pill

## В следующей пилюле

> In the next pill, we will talk about the \"`override`\" design pattern.
> The `graphvizCore` seems straightforward.
> It starts from `graphviz.nix` and builds it without gd.
> In the next pill, we will consider another point of view: starting from `pkgs.graphviz` and disabling gd?

В следюущей пилюле мы поговорим о паттерне проектирования "`Переопределение`".
`graphvizCore` кажется простым.
Он использует скрипт `graphviz.nix` и собирает пакет без поддержки `gd`.
В следующей пилюле мы посмотрим на этот процесс с другой точки зрения: может быть использовать функцию `pkgs.graphviz` и отключать `gd` при её вызове?
