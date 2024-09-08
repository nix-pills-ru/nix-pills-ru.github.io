> # Package Repositories and the Inputs Design Pattern {#inputs-design-pattern}

# Репозитории паттернов и паттерн проектирования Входы

> Welcome to the 12th Nix pill.
> In the previous [11th pill](#garbage-collector), we stopped packaging and cleaned up the system with the garbage collector.

Добро пожаловать на двенадцатую пилюлю Nix.
В предыдущей [одиннадцатой пилюле](11-garbage-collector.md) мы приостановили разговор о пакетах и очистили систему с помощью сборщика мусора.

> This time, we will resume packaging and improve different aspects of it.
> We will also demonstrate how to create a repository of multiple packages.

Сейчас мы вернёмся к пакетам и улушчим некоторые их аспекты.
Мы также продемонстрируем, как создать репозиторий для множества пакетов.

> ## Repositories in Nix

## Репозитории в Nix

> Package repositories in Nix arose naturally from the need to organize packages.
> There is no directory structure or packaging policy prescribed by Nix itself; Nix, as a full, functional programming language, is powerful enough to support multiple different repository formats.

Репозитории пакетов в Nix появились натруально из-за необходимость организовать пакеты.
Не существует предустановленной структуры каталогов или политики пакетирования, которая была бы предписана в самом Nix; Nix, как полный, функциональный язык программирования, достаточно мощен, чтобы поддерживать много различных форматов репозиториев.

> Over time, the `nixpkgs` repository evolved a particular structure.
> This structure reflects the history of Nix as well as the design patterns adopted by its users as useful tools in building and organizing packages.
> Below, we will examine some of these patterns in detail.

С течением времени репозиторий `nixpkgs` приобрёл определённую структуру.
Эта структура отражает историю Nix, также, как и паттерны проектирования, принятые его пользователями, как полезные интрументы построения и организации пакетов.
Ниже мы подробнее познакомимся с некоторыми из этих паттернов.

> ## The single repository pattern

## Паттерн Один репозиторий

> Different operating system distributions have different opinions about how package repositories should be organized.
> Systems like Debian scatter packages in several small repositories (which tends to make tracking interdependent changes more difficult, and hinders contributions to the repositories), while systems like Gentoo put all package descriptions in a single repository.

Различные дистрибутивы операционных системы имеют различные мнения о том, как репозитории пакетов должны быть организованы.
Такие системы, как Debian, распределяют пакеты по нескольким маленьким репозиториям (что, как правило, делает отслеживание взаимозависимых изменений более трудоёмким и мешает внесению правок в репозитории), в том время как системы наподобие Gentoo складывают все описания пакетов в Один репозиторий.

> Nix follows the \"single repository\" pattern by placing all descriptions of all packages into [nixpkgs](https://github.com/NixOS/nixpkgs).
> This approach has proven natural and attractive for new contributions.

Nix следует паттерну "Один репозиторий", размещая все описания всех пакетов в [nixpkgs](https://github.com/NixOS/nixpkgs).
Этот подход зарекомендовал себя, как естественный и привликательный для внесения новых изменений.

> For the rest of this pill, we will adopt the single repository pattern.
> The natural implementation in Nix is to create a top-level Nix expression, followed by one expression for each package.
> The top-level expression imports and combines all package expressions in an attribute set mapping names to packages.

Остаток этой пилюли мы посвятим разбору паттерна Один репозиторий.
Естественная для Nix реализация — создать выражение верхнего уровня, за которым следуют выражения для пакета — по одному на каждый.
Выражение верхнего уровня импортирует и объединяет все выражения пакетов в наборы атрибутов, отображая имена в пакеты.

> In some programming languages, such an approach \-- including every possible package description in a single data structure \-- would be untenable due to the language needing to load the entire data structure into memory before operating on it.
> Nix, however, is a lazy language and only evaluates what is needed.

В некоторых языках программирования такой подход — включение каждого возможного описания пакета в единую структуру данных — был бы слишком ресурсоёмким, поскольку языку потребовалось бы загрузить всю структуру данных в память перед тем, как работать с ней.
В то же время Nix — ленивый язык и вычисляет только когда это нужно.

> ## Packaging `graphviz`

## Упаковываем  `graphviz`

> We have already packaged GNU `hello`.
> Next, we will package a graph-drawing program called `graphviz` so that we can create a repository containing multiple packages.
> The `graphviz` package was selected because it uses the standard autotools build system and requires no patching.
> It also has optional dependencies, which will give us an opportunity to illustrate a technique to configure builds to a particular situation.

Мы уже создали пакет из GNU `hello`.
Теперь мы создадим пакет bp программы для рисования графиков под называется `graphviz`, чтобы можно было создать репозиторий, содержащий несколько пакетов.
Пакет `graphviz` был выбран потому, что он использует стандартную систему сборки `autotools` и не требует никаких патчей.
В нём также есть опциональные зависимости, которые позволяют нам проиллюстрировать технику конфигурирования сборки в зависимости от ситуации.

> First, we download `graphviz` from [gitlab](https://gitlab.com/api/v4/projects/4207231/packages/generic/graphviz-releases/2.49.3/graphviz-2.49.3.tar.gz).
> The `graphviz.nix` expression is straightforward:

Вначале мы загружаем `graphviz` из [gitlab](https://gitlab.com/api/v4/projects/4207231/packages/generic/graphviz-releases/2.49.3/graphviz-2.49.3.tar.gz).
Выражение `graphviz.niz` достаточно простое:

```nix
let
  pkgs = import <nixpkgs> { };
  mkDerivation = import ./autotools.nix pkgs;
in
mkDerivation {
  name = "graphviz";
  src = ./graphviz-2.49.3.tar.gz;
}
```

> If we build the project with `nix-build graphviz.nix`, we will get runnable binaries under `result/bin`.
> Notice how we reused the same `autotools.nix` of `hello.nix.`

Собрав проект командой `nix-build graphviz.nix`, мы получим готовые исполняемые файлы в катлоге `result/bin`.
Обратите внимание, что мы повторно используем скрипт `autotools.nix`, созданный нами при написании `hello.nix`.

> By default, `graphviz` does not compile with the ability to produce `png` files.
> Thus, the derivation above will build a binary supporting only the native output formats, as we see below:

По умолчанию `graphviz` не включает в себя возможность сохранять файлы `png`.
Таким образом, приведённая выше деривация создат программу, которая поддерживает только собственные форматы вывода, как мы видим ниже:

```bash
$ echo 'graph test { a -- b }'|result/bin/dot -Tpng -o test.png
Format: "png" not recognized. Use one of: canon cmap [...]
```

> If we want to produce a `png` file with `graphviz`, we must add it to our derivation.
> The place to do so is in `autotools.nix`, where we created a `buildInputs` variable that gets concatenated to `baseInputs`.
> This is the exact reason for this variable: to allow users of `autotools.nix` to add additional inputs from package expressions.

Если мы хотим сохранять `png` файлы с помощью `graphviz`, мы должны добавить его поддержку формата в деривацию.
Это можно сделать в `autotools.niz`, где мы описали переменную `buildInputs`, которая объединяется с `baseInputs`.
Именно затем и нужна эта переменная: позволить пользователям `autotools.nix` добавлять дополнительные входные деривации из выражения для пакета.

> Version 2.49 of `graphviz` has several plugins to output `png`.
> For simplicity, we will use `libgd`.

В `graphviz` версии 2.49 есть несколько плагинов для вывода `png`.
Для простоты будем использовать `libgd`.

> ## Passing library information to `pkg-config` via environment variables

## Передаём информацию о библиотеках в `pkg-config` через переменные окружения

> The `graphviz` configuration script uses `pkg-config` to specify which flags are passed to the compiler.
> Since there is no global location for libraries, we need to tell `pkg-config` where to find its description files, which tell the configuration script where to find headers and libraries.

Конфигурационный скрипт `graphviz` использует `pkg-config` для того, чтобы определить, какие флаги должны быть переданы компилятору.
Поскольку не существует глобального каталога, гед собраны библиотеки, мы должны сказать `pkg-config` где искать файлы описания, которые подскажут конфигурационному скрипту, где искать заголовки и библиотеки.

> In classic POSIX systems, `pkg-config` just finds the `.pc` files of all installed libraries in system folders like `/usr/lib/pkgconfig`.
> However, these files are not present in the isolated environments presented to Nix.

В классических POSIX системах, `pkg-config` просто ищет файлы `.pc` для всех установленных библиотек в системном каталоге наподобие `/usr/lib/pkgconfig`.
Однако, эти файлы не представлены в изолированных окружениях, существующих в Nix.

> As an alternative, we can inform `pkg-config` about the location of libraries via the `PKG_CONFIG_PATH` environment variable.
> We can populate this environment variable using the same trick we used for `PATH`: automatically filling the variables from `buildInputs`.
> This is the relevant snippet of `setup.sh`:

В качестве альтернативы мы можем информацировать `pkg-config` о расположении библиотек через переменную окружения `PKG_CONFIG_PATH`.
Мы можем определить эту переменную окружения, используя тот же трюк, который мы использовали для `PATH`: автоматически заполнив пути из `buildInputs`.
Вот соответствующий фрагмент `setup.sh`:

```bash
for p in $baseInputs $buildInputs; do
    if [ -d $p/bin ]; then
        export PATH="$p/bin${PATH:+:}$PATH"
    fi
    if [ -d $p/lib/pkgconfig ]; then
        export PKG_CONFIG_PATH="$p/lib/pkgconfig${PKG_CONFIG_PATH:+:}$PKG_CONFIG_PATH"
    fi
done
```

> Now if we add derivations to `buildInputs`, their `lib/pkgconfig` and `bin` paths are automatically added in `setup.sh`.

Теперь, если мы добавим деривации к `buildInputs`, их подкаталоги `lib/pkgconfig` и `bin` автоматически добавятся в `setup.sh`.

> ## Completing graphviz with `gd`

# Завершаем `graphviz` с помощью `gd`

> Below, we finish the expression for `graphviz` with `gd` support.
> Note the use of the `with` expression in `buildInputs` to avoid repeating `pkgs`:

Ниже мы завершаем выражение для `graphviz`, включив в него поддержку `gd`.
Обратите внимание, что использование выражения `with` c `buildInputs` позволяет избежать дублирования `pkgs`:

```nix
let
  pkgs = import <nixpkgs> { };
  mkDerivation = import ./autotools.nix pkgs;
in
mkDerivation {
  name = "graphviz";
  src = ./graphviz-2.49.3.tar.gz;
  buildInputs = with pkgs; [
    pkg-config
    (pkgs.lib.getLib gd)
    (pkgs.lib.getDev gd)
  ];
}
```

> We add `pkg-config` to the derivation to make this tool available for the configure script.
> As `gd` is a package with [split outputs](https://nixos.org/manual/nixpkgs/stable/#sec-multiple-outputs-), we need to add both the library and development outputs.

Мы добавляем к деривации `pkg-config` чтобы сделать эту утилиту доступной для конфигурационного скрипта.
Поскольку `gd` — пакет с [несколькими выходными путями](https://nixos.org/manual/nixpkgs/stable/#sec-multiple-outputs-), мы вынуждены добавить оба выходных пути — `lib` и `dev`.

> After building, `graphviz` can now create `png`s.

После сборки `graphviz` теперь может создавать `png` файлы.

> ## The repository expression

## Выражение для репозитория

> Now that we have two packages, we want to combine them into a single repository.
> To do so, we\'ll mimic what `nixpkgs` does: we will create a single attribute set containing derivations.
> This attribute set can then be imported, and derivations can be selected by accessing the top-level attribute set.

Теперь, когда у нас есть два пакета, мы хотим объединить их в один репозиторий.
Чтобы это сделать, будем подражать `nixpkgs`: созданим один набор атрибутов, содержащий деривации.
Впоследствии этот набор атрибутов можно будет импортировать, и деривации могут быть выбраны путём доступа через набор атрибутов верхнего уровня.

> Using this technique we are able to abstract from the file names.
> Instead of referring to a package by `REPO/some/sub/dir/package.nix`, this technique allows us to select a derivation as `importedRepo.package` (or `pkgs.package` in our examples).

Используя эти технику, мы можем абстрагиваться от имён файлов.
Вместо ссылки на пакет `REPO/some/sub/dir/package.nix` эта техника позволяет выбрать деривацию через `importedRepo.package` (или `pkgs.package`` в нашем примере).

> To begin, create a default.nix in the current directory:

Чтобы начать, создадим `default.nix` в текущем каталоге:

```nix
{
  hello = import ./hello.nix;
  graphviz = import ./graphviz.nix;
}
```

> This file is ready to use with `nix repl`:

Этот файл можно использовать из `nix repl`:

```bash
$ nix repl
nix-repl> :l default.nix
Added 2 variables.
nix-repl> hello
«derivation /nix/store/dkib02g54fpdqgpskswgp6m7bd7mgx89-hello.drv»
nix-repl> graphviz
«derivation /nix/store/zqv520v9mk13is0w980c91z7q1vkhhil-graphviz.drv»
```

> With `nix-build`, we can pass the -A option to access an attribute of the set from the given `.nix` expression:

Для `nix-build` мы должны передать параметр `-A` чтобы получить доступ к атрибуту из набора указанного выражения `.nix`:

```bash
$ nix-build default.nix -A hello
[...]
$ result/bin/hello
Hello, world!
```

> The `default.nix` file is special. When a directory contains a `default.nix` file, it is used as the implicit nix expression of the directory.
> This, for example, allows us to run `nix-build -A hello` without specifying `default.nix` explicitly.

Файл `default.nix` — особенный. Если каталог содержит `default.nix`, он используется как неявное выражение NIX этого каталога.
Это, например, позволяет запустить `nix-build -A hello`, без явного указания `default.nix`.

> We can now use `nix-env` to install the package into our user environment:

Теперь мы можем использовать `nix-env` для установки пакета в наше пользовательское окружение:

```bash
$ nix-env -f . -iA graphviz
[...]
$ dot -V
```

> Taking a closer look at the above command, we see the following options:

Бросив внимательный взглядт на команду выше, мы видим следующие параметры:

> -   The -f option is used to specify the expression to use.
>     In this case, the expression is the `./default.nix` of the current directory.
> -   The -i option stands for \"installation\".
> -   The -A is the same as above for `nix-build`.

- Параметр `-f` используется для указания нужного выражения.
  В нашем случае это выражения из `./default.nix` текущего каталога.
- Параметр `-i` означает "инсталлировать".
- Параметр `-A` имеет такой же смысл, как и в `nix-build`.

> We reproduced the very basic behavior of `nixpkgs`: combining multiple derivations into a single, top-level attribute set.

Мы воспроизвели самое базовое поведение `nixpkgs`: объединили несколько дериваций в один набор атрибутов верхнего уровня.

> ## The inputs pattern

## Паттерн Входы

> The approach we\'ve taken so far has a few problems:

У подхода, который мы рассмотрели, есть несколько проблем:

> -   First, `hello.nix` and `graphviz.nix` are dependent on `nixpkgs`, which they import directly.
>     A better approach would be to pass in `nixpkgs` as an argument, as we did in `autotools.nix`.
> -   Second, we don\'t have a straightforward way to compile different variants of the same software, such as `graphviz` with or without `libgd` support.
> -   Third, we don\'t have a way to test `graphviz` with a particular `libgd` version.

- Во-первых, `hello.nix` и `graphviz.niz` зависят от `nigpkgs`, который они импортируют напрямую.
  Лучшим подходом была бы передача `nixpkgs` в качестве аргумента, как это делали в `autotools.nix`.
- Во-вторых, у нас нет простого способа комплириовать различные варианты одной и той же программы, скажем, `graphviz` с поддержкой и без поддержки `libgd`.
- В-третьих, у нас нет возможности протестировать `graphviz` с определённой версией `libgd`.

> Until now, our approach to addressing the above problems has been inadequate and required changing the nix expression to match our needs.
> With the `inputs` pattern, we provide another answer: let the user change the `inputs` of the expression.

До сих пор наш подход к решению всех этих проблем был неадекватным требовал изменения выражения Nix в зависимости от наших потребностей.
С помощью паттерна `Входы` мы предлагает другое решение: пусть пользователь меняет параметр `inputs` выражения.

> When we talk about \"the inputs of an expression\", we are referring to the set of derivations needed to build that expression.
> In this case:

Когда мы говорим о "входах выражения", мы имеем в виду деривации, нужные для сборки выражения.
В нашем случае:

> -   `mkDerivation` from `autotools`. Recall that `mkDerivation` has an implicit dependency on the toolchain.
> -   `libgd` and its dependencies.

- `mkDerivation` из `autotools`. Напомним, что `mkDerivation` имеет неявную зависимость от инструментария.
- `libgd` и её зависимости.

> The `./src` directory is also an input, but we wouldn\'t change the source from the caller.
> In `nixpkgs` we prefer to write another expression for version bumps (e.g. because patches or different inputs are needed).

Каталог `./scr` также является входом, но мы не стали бы менять исходный код в скрипте сборки.
В `nixpkgs` мы предпочитаем написать ещё одно выражение при повышении версии (в том числе из-за патчей или отличных входящих зависимостей).

> Our goal is to make package expressions independent of the repository.
> To achieve this, we use functions to declare inputs for a derivation.
> For example, with `graphviz.nix`, we make the following changes to make the derivation independent of the repository and customizable:

Наша цель — создать выражение для пакета, независимое от репозитория.
Чтобы добиться этого, мы используем функции для объявления входящих зависимостей для деривации.
Например, в `graphviz.nix` мы делает следующие изменения, чтобы деривация была независимой от репозитория и настраиваемой:

```nix
{ mkDerivation, lib, gdSupport ? true, gd, pkg-config }:

mkDerivation {
  name = "graphviz";
  src = ./graphviz-2.49.3.tar.gz;
  buildInputs =
    if gdSupport
      then [
        pkg-config
        (lib.getLib gd)
        (lib.getDev gd)
      ]
      else [];
}
```

> Recall that \"`{...}: ...`\" is the syntax for defining functions accepting an attribute set as argument; the above snippet just defines a function.

Напомню, что `{...}: ...` — это синтаксис определения функции, принимающей набор атрибутов в качестве аргумента; пример выше просто определяет функцию.

> We made `gd` and its dependencies optional.
> If `gdSupport` is true (which it is by default), we will fill `buildInputs` and `graphviz` will be built with `gd` support.
> Otherwise, if an attribute set is passed with `gdSupport = false;`, the build will be completed without `gd` support.

Мы сделали `gd` и её зависимости опциональными.
Если `gdSupport` истинна (по-умолчанию это именно так), мы заполняем `buildInputs` и `graphviz` будет собираться с поддержкой `gd`.
В противном случае, если набор атрибутов передаётся с `gdSupport = false;`, сборка будет завершена без поддержки `gd`.

> Going back to back to `default.nix`, we modify our expression to utilize the inputs pattern:

Вернёмся к `default.nix`, и модифицируем наше выражение, чтобы использовать шаблон Входящие зависимости.

```nix
let
  pkgs = import <nixpkgs> { };
  mkDerivation = import ./autotools.nix pkgs;
in
with pkgs;
{
  hello = import ./hello.nix { inherit mkDerivation; };
  graphviz = import ./graphviz.nix {
    inherit
      mkDerivation
      lib
      gd
      pkg-config
      ;
  };
  graphvizCore = import ./graphviz.nix {
    inherit
      mkDerivation
      lib
      gd
      pkg-config
      ;
    gdSupport = false;
  };
}
```

> We factorized the import of `nixpkgs` and `mkDerivation`, and also added a variant of `graphviz` with `gd` support disabled.
> The result is that both `hello.nix` (left as an exercise for the reader) and `graphviz.nix` are independent of the repository and customizable by passing specific inputs.

Мы разделили импорт `nixpkgs` и `mkDerivation`, и также добавили вариант сборки `graphviz` без поддержки `gd`.
Теперь и `hello.nix` (оставленный читателю в качестве упражнения), и `graphviz.nix` не зависят от репозитория и настраиваются с помощью Входных параметров.

> If we wanted to build `graphviz` with a specific version of `gd`, it would suffice to pass `gd = ...;`.

Если мы захотим собрать `graphviz` с нужной версией `gd`, достаточно будет передать `gd = ...;`

> If we wanted to change the toolchain, we would simply pass a different `mkDerivation` function.

Если мы захотим изменить инструмент сборки, мы можем передать другую реализацию функции `mkDerivation`.

> Let\'s talk a closer look at the snippet and dissect the syntax:

Давайте внимательные познакомимся с этим фрагментом и разберём синтаксис:

> -   The entire expression in `default.nix` returns an attribute set with the keys `hello`, `graphviz`, and `graphvizCore`.
> -   With \"`let`\", we define some local variables.
> -   We bring `pkgs` into the scope when defining the package set.
>     This saves us from having to type `pkgs`\" repeatedly.
> -   We import `hello.nix` and `graphviz.nix`, which each return a function.
>     We call the functions with a set of inputs to get back the derivation.
> -   The \"`inherit x`\" syntax is equivalent to \"`x = x`\".
>     This means that the \"`inherit gd`\" here, combined with the above \"`with pkgs;`\", is equivalent to \"`gd = pkgs.gd`\".

- Выражение `default.nix` возвращает набор атрибутов с ключами `hello`, `graphviz` и `graphvizCore`.
- С помощью `let` мы определяем несколько локальных переменных.
- Мы включаем `pkgs` в область видимости, определяя набор пакетов.
  Это избавляет нас от необходимост многократно набирать `pkgs`.
- Мы импортируем `hello.nix` и `graphviz.nix`, каждый из которых возвращает функцию.
  Мы вызываем функции с набором входных параметров, чтобы получить деривации.
- Синтаксис `inherit x` эквивалентен `x = x`.
  Строка `inherit gd` скомбинированная с `with pkgs;` эквивалентна `gd = pkgs.gd`.

> The entire repository of this can be found at the [pill 12](https://gist.github.com/tfc/ca800a444b029e85a14e530c25f8e872) gist.

Весь репозиторий, посвящённый пилюле 12, можно найти в этом [GitHub Gist](https://gist.github.com/tfc/ca800a444b029e85a14e530c25f8e872).[^1]

> ## Conclusion

## Заключение

> The \"`inputs`\" pattern allows our expressions to be easily customizable through a set of arguments.
> These arguments could be flags, derivations, or any other customizations enabled by the nix language.
> Our package expressions are simply functions: there is no extra magic present.

Паттерн **Входные параметры** позволяет нашим выражениям быть легко настраиваемыми через набор аргументов.
Эти аргументы могут быть флагами, деривациями или любыми другими настройками, доступными в языке Nix.
Наши пакетные выражения — всего лишь функции: здесь нет никакой скрытой магии.

> The \"`inputs`\" pattern also makes the expressions independent of the repository.
> Given that we pass all needed information through arguments, it is possible to use these expressions in any other context.

Паттерн **Входящие параметры** также позволяет создавать выражения, независимые от репозитория.
Поскольку мы передаём все нужные данные через аргументы, их можно исопльзовать в любом другом контексте.

> ## Next pill

## В следующей пилюле

> In the next pill, we will talk about the \"`callPackage`\" design pattern.
> This removes the tedium of specifying the names of the inputs twice: once in the top-level `default.nix`, and once in the package expression.
> With `callPackage`, we will implicitly pass the necessary inputs from the top-level expression.

В следующей пилюле мы поговорим про паттерн проектирования **Вызов пакета**.
Он избавляет от необходимости указывать имена входных данных дважды: на верхнем уровне в `default.nix`, и в пакетном выражении.
Благодаря **Вызову пакета** мы можем неявно передавать нужные входные параметры из верхне-уровневого выражения.

[^1] У термина **gist** нет устоявшегося перевода на русский язык. В целом это средство, которое позволяет ссылаться не на весь код в репозитории, а только на самые важные его кусочки из разных файлов.