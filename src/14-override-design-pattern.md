> # Override Design Pattern

# Паттерн проектирования Переопределение

> Welcome to the 14th Nix pill.
> In the previous [13th](#callpackage-design-pattern) pill, we introduced the `callPackage` pattern and used it to simplify the composition of software in a repository.

Добро пожаловать на четырнадцутую пилюлю Nix.
В предыдущей [тринадцатой пилюле](13-callpackage-design-pattern.md) мы рассказали о паттерне `callPackage` и использовали его, чтобы упростить объединение пакетов в репозиторий.

> The next design pattern is less necessary, but is useful in many cases and is a good exercise to learn more about Nix.

Следующий паттерн проектирования не настолько нужен, но полезен в большом количестве случаев и позволяет лучше изучить Nix.

> ## About composability

## О композиции

> Functional languages are known for being able to compose functions.
> In particular, these languages gain expressivity from functions that manipulate an original value into a new value having the same structure.
> This allows us to compose multiple functions to perform the desired modifications.

Функциональные языки известны тем, что в них можно делать композицию функций.
В частности, эти языки усиливают выразительность функций, которые преобразуют исходное значение в новое значение, имеющее ту же структуру.
Это позволяет нам объединять несколько функций, чтобы выполнить желаемые изменения.

> In Nix, we mostly talk about **functions** that accept inputs in order to return **derivations**.
> In our world, we want utility functions that are able to manipulate those structures.
> These utilities add some useful properties to the original value, and we\'d like to be able to apply more utilities on top of the result.

В Nix мы обычно говорим о **функциях**, которые принимают входящие параметры и затем возвращают **деривации**.
В нашем мире нам нужны функции-утилиты, которые позволяют манипулировать этими структурами.
Эти утилиты добавляет несколько полезных свойств к исходным значениям, и нам бы хотелось иметь больше возможностей для применения утилит к последнему полученному результату.

> For example, let\'s say we have an initial derivation `drv` and we want to transform it into a `drv` with debugging information and custom patches:

Например, скажем, у нас есть начальная деривация `drv` и мы хотим преобразовать её в `drv` с отладочной информацией и пользовательскими патчами:

```nix
debugVersion (applyPatches [ ./patch1.patch ./patch2.patch ] drv)
```

> The final result should be the original derivation with some changes.
> This is both interesting and very different from other packaging approaches, which is a consequence of using a functional language to describe packages.

Конечным результатом должна стать оригинальная деривация с некоторыми изменениями.
Это интересный и в то же время очень отличный от других подход к созданию пакетов, что является следствием использования функционального языка для описания пакетов.

> Designing such utilities is not trivial in a functional language without static typing, because understanding what can or cannot be composed is difficult.
But we try to do our best.

Разработка таких утилит нетривиальна в функциональном языке без статической типизации, потому что трудно разобраться, что можно и что нельзя компоновать.
Но мы постараемся сделать всё возможное.

> ## The override pattern

## Паттерн Переопределение (override)

> In [pill 12](#inputs-design-pattern) we introduced the inputs design pattern.
> We do not return a derivation picking dependencies directly from the repository; rather we declare the inputs and let the callers pass the necessary arguments.

В [пилюле 12](12-inputs-design-pattern.md) мы рассказали о паттерне проектирования Входящие.
Мы не возвращаем деривацию, выбирая зависимости напрмую из репозитория; вместо этого мы объявляем входящие параметры и позволяем вызывающим функциям передавать необходимые аргументы.

> In our repository we have a set of attributes that import the expressions of the packages and pass these arguments, getting back a derivation.
Let\'s take for example the graphviz attribute:

В нашем репозитории есть набор атрибутов, которые импортируют выражения пакетов и передают эти аргументы, получая в ответ деривацию.
Для примера возьмём атрибут `graphviz`:

```nix
graphviz = import ./graphviz.nix { inherit mkDerivation gd fontconfig libjpeg bzip2; };
```

> If we wanted to produce a derivation of graphviz with a customized gd version, we would have to repeat most of the above plus specifying an alternative gd:

Если мы хотим создать деривацию `graphviz` с отличающейся версией `gd`, нам придётся повторить большую часть приведённого кода, плюс указать альтернативу `gd`:

```nix
{
  mygraphviz = import ./graphviz.nix {
    inherit
      mkDerivation
      fontconfig
      libjpeg
      bzip2
      ;
    gd = customgd;
  };
}
```

> That\'s hard to maintain.
> Using `callPackage` would be easier:

Такой код трудно поддерживать.
Использовать `callPackage` уже проще:

```nix
mygraphviz = callPackage ./graphviz.nix { gd = customgd; };
```

> But we may still be diverging from the original graphviz in the repository.

Но мы всё ещё наследуем оригинальный пакет `graphviz` из репозитория.

> We would like to avoid specifying the nix expression again.
> Instead, we would like to reuse the original graphviz attribute in the repository and add our overrides like so:

Нам бы хотелось избавиться от повторного указания выражения Nix.
Вместо этого нам бы хотелось повторно использоваль оригинальный атрибут `graphviz` из репозитория и добавить наши переопределения, как показано ниже:

```nix
mygraphviz = graphviz.override { gd = customgd; };
```

> The difference is obvious, as well as the advantages of this approach.

Отличие очевидно, так же, как и преимущества такого подхода.

> [Note:]{.underline} that `.override` is not a \"method\" in the OO sense as you may think.
> Nix is a functional language.
> The`.override` is simply an attribute of a set.

Обратите внимание, что `.override` не является "методом" в смысле ООП, как вы могли бы подумать.
Nix — функциональный язык.
Так что `.override` — это всего лишь атрибут набора.

> ## The override implementation

## Реализация паттерна Переопределение

> Recall that the graphviz attribute in the repository is the derivation returned by the function imported from `graphviz.nix`.
> We would like to add a further attribute named \"`override`\" to the returned set.

Вспомним, что атрибут `graphviz` в репозитории — это деривация, возвращаемая функцией, импортируемой из `graphviz.nix`.
Нам бы хотелось добавить дополнительный атрибут `override` к возвращаемому набору.

> Let\'s start by first creating a function \"`makeOverridable`\".
> This function will take two arguments: a function (that must return a set) and the set of original arguments to be passed to the function.

Давайте начнём с создания функции "`makeOverridable`".
Эта функция получает два аргумента: функцию (которая должна возвращать набор) и набор исходных аргументов, которые будут переданы этой функции.

> We will put this function in a `lib.nix`:

Мы поместим эту функцию в `lib.nix`:

```nix
{
  makeOverridable =
    f: origArgs:
    let
      origRes = f origArgs;
    in
    origRes // { override = newArgs: f (origArgs // newArgs); };
}
```

> `makeOverridable` takes a function and a set of original arguments.
> It returns the original returned set, plus a new `override` attribute.

`makeOverridable` принимает функцию и набор исходных аргументов.
Она возвращает исходный набор плюс новый атрибут `override`.

> This `override` attribute is a function taking a set of new arguments, and returns the result of the original function called with the original arguments unified with the new arguments.
> This is admittedly somewhat confusing, but the examples below should make it clear.

Этот атрибут `override` — это функция, принимающая набор новых аргументов и возвращающая результат вызова исходной функции с объединёнными исходными и новыми аргументами.
Вероятно, это объяснение немного запутанное, но примеры ниже должны прояснить ситуацию.

> Let\'s try it with `nix repl`:

Давайте посмотрим, как это работает в `nix repl`:

```bash
$ nix repl
:l lib.nix
Added 1 variables.
f = { a, b }: { result = a+b; }
f { a = 3; b = 5; }
{ result = 8; }
res = makeOverridable f { a = 3; b = 5; }
res
{ override = «lambda»; result = 8; }
res.override { a = 10; }
{ result = 15; }
```

> Note that, as we specified above, the function `f` does not return the plain sum.
> Instead, it returns a set with the sum bound to the name `result`.

Обратите внимание, что, как мы писали выше, функция `f` не возвращает просто сумму.
Вместо этого она возвращает набор, в котором сумма привязана к атрибуту `result`.

> The variable `res` contains the result of the function call without any override.
> It\'s easy to see in the definition of `makeOverridable`.
> In addition, you can see that the new `override` attribute is a function.

Переменная `res` содержит результат выполнения функции без переопределения.
Это легко увидеть в определении `makeOverridable`.
Дополнительно, как видите, появился новый атрибут `override`, значеним которого является функция.

> Calling `res.override` with a set will invoke the original function with the overrides, as expected.

Вызов `res.override`, как и ожидалось, вызовет исходную функцию с переопределёнными атрибутами.

> This is a good start, but we can\'t override again! This is because the returned set (with `result = 15`) does not have an `override` attribute of its own.
> This is bad; it breaks further composition.

Хорошее начало, но мы не может повторно переопределить атрибуты! Всё портому, что возвращаемый набор (с `result = 15`) не имеет собственного атрибута `override`.
Это плохо, так как мешает композиции функций.

> The solution is simple: the `.override` function should make the result overridable again:

Решение простое: функция `.override` должна вернуть переопределяемый результат:

```nix
rec {
  makeOverridable =
    f: origArgs:
    let
      origRes = f origArgs;
    in
    origRes // { override = newArgs: makeOverridable f (origArgs // newArgs); };
}
```

> Please note the `rec` keyword.
> It\'s necessary so that we can refer to `makeOverridable` from `makeOverridable` itself.

Обратите внимание на ключевое слово `rec`.
Оно необходимо, чтобы мы могли вызвать `makeOverridable` из самой `makeOverridable`.

> Now let\'s try overriding twice:

Теперь попробуем двойное переопределение:

```bash
nix-repl> :l lib.nix
Added 1 variables.
nix-repl> f = { a, b }: { result = a+b; }
nix-repl> res = makeOverridable f { a = 3; b = 5; }
nix-repl> res2 = res.override { a = 10; }
nix-repl> res2
{ override = «lambda»; result = 15; }
nix-repl> res2.override { b = 20; }
{ override = «lambda»; result = 30; }
```

> Success!
> The result is 30 (as expected) because `a` is overridden to 10 in the first override, and `b` is overridden to 20 in the second.

Получилось!
Результа равен 30 (как и ожидалось) потому что `a` получила занчение 10 в первом переопределении, а `b` — 20 во втором.

> Now it would be nice if `callPackage` made our derivations overridable.
> This is an exercise for the reader.

Теперь было бы неплохо, если бы `callPackage` делал наши деривации переопределяемыми.
Пусть это будет упражнением для читателя.

> ## Conclusion

## Заключение

> The \"`override`\" pattern simplifies the way we customize packages starting from an existing set of packages.
> This opens a world of possibilities for using a central repository like `nixpkgs` and defining overrides on our local machine without modifying the original package.

Паттерн "`Переопределение`" упрощает настройку пакетов, беря за основу существующий набор пакетов.
Это открывает целый мир возможностей для использования центрального репозитория, такого как `nixpkgs` и переопределения пакетов на нашей локальной машине без изменения исходников.

> We can dream of a custom, isolated `nix-shell` environment for testing graphviz with a custom gd:

Мы можем мечтать о специальном изолированном окружении `nix-shell` для тестирования `graphviz` с альтернативной версией `gd`:

```nix
debugVersion (graphviz.override { gd = customgd; })
```

> Once a new version of the overridden package comes out in the repository, the customized package will make use of it automatically.

Как только появится новая версия пакета, который мы переопределили, она будет автоматически использоваться в переопределенном пакете.

> The key in Nix is to find powerful yet simple abstractions in order to let the user customize their environment with highest consistency and lowest maintenance time, by using predefined composable components.

Ключевой особенностью Nix является поиск мощных и простых абстракций, дающих пользователю возможность настроивать свою окружение с максимальной согласованностью и минимальными усилиями для поддержки, используя предопределенные композируемые компоненты.

> ## Next pill

## В следующей пилюле

> In the next pill, we will talk about Nix search paths.
> By \"search path\", we mean a place in the file system where Nix looks for expressions.
> This answers the question of where `<nixpkgs>` comes from.

В следующей пилюле мы поговорим о путях поиска Nix.
Под "путём поиска" мы подразумеваем место в файловой системе, где Nix ищет выражения.
Это отвечает на вопрос, откуда берётся `<nixpkgs>`.
