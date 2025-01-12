> # Nixpkgs Overriding Packages

# Переопределение пакетов nixpkgs

> Welcome to the 17th Nix pill.
> In the previous [16th](#nixpkgs-parameters) pill we have started to dive into the [nixpkgs repository](http://github.com/NixOS/nixpkgs).
> `Nixpkgs` is a function, and we\'ve looked at some parameters like `system` and `config`.

Добро пожаловать на семнадцатую пилюлю Nix.
В предыдущей [шестнадцатой пилюле](16-nixpkgs-parameters.md) мы начали погружение в [репозиторий nixpkgs](http://github.com/NixOS/nixpkgs).
'Nixpkgs` — это функция и мы посмотрели на несколько параметров, таких как `system` и `config`.

> Today we\'ll talk about a special attribute: `config.packageOverrides`.
> Overriding packages in a set with fixed point can be considered another design pattern in nixpkgs.

Сегодня мы поговорим о специальном атрибуте: `config.packageOverrides`.
Переопределение пакетов в наборе с неподвижной точкой можно рассматривать как ещё один паттерн проектирования в `nixpkgs`.

> ## Overriding a package

## Переопределение пакета

> Recall the override design pattern from the [nix pill 14](#override-design-pattern).
> Instead of calling a function with parameters directly, we make the call (function + parameters) overridable.

Вспомните паттерн проектирования переопределение из [пилюли 14](14-override-design-pattern.md).
Вместо прямого вызова функции с параметрами, мы делаем вызов (функция + параметры) переопределяемым.

> We put the override function in the returned attribute set of the original function call.

Мы помещаем функцию override в возвращаемый набор атрибутов вызова оригинальной функции.

> Take for example graphviz.
> It has an input parameter xorg.
> If it\'s null, then graphviz will build without X support.

Возьмём к примеру `graphviz`.
У него есть входящий параметр `xorg`.
Если он равен `null`, `graphviz` будет собран без поддержки X Window System.

```bash
$ nix repl
nix-repl> :l <nixpkgs>
Added 4360 variables.
nix-repl> :b graphviz.override { withXorg = false; }
```

> This will build graphviz without X support, it\'s as simple as that.

Этот вызов соберёт `graphviz` без поддержки X, всё настолько просто.

> However, let\'s say a package `P` depends on graphviz, how do we make `P` depend on the new graphviz without X support?

Однако, пусть, скажем, пакет `P` зависит от `graphviz`, как сделать, чтобы `P` зависел от новой версии `graphviz` без поддержки X?

> ## In an imperative world\...

## В императивном мире...

> ...you could do something like this:

...вы могли бы сделать что-то вроде этого:

```nix
pkgs = import <nixpkgs> {};
pkgs.graphviz = pkgs.graphviz.override { withXorg = false; };
build(pkgs.P)
```

> Given `pkgs.P` depends on `pkgs.graphviz`, it\'s easy to build `P` with the replaced graphviz.
> In a pure functional language it\'s not that easy because you can assign to variables only once.

Учитывая, что `pkgs.P` зависит от `pkgs.graphviz`, легко собрать `P` с изменённым `graphviz`.
В чистом функциональном языке это не так просто, поскольку значение переменной можно назначить только один раз.

> ## Fixed point

## Неподвижная точка

> The fixed point with lazy evaluation is crippling but about necessary in a language like Nix.
> It lets us achieve something similar to what we\'d do imperatively.

Неподвижная точка с ленивым вычислением — это неудобно, но необходимо в таком языке, как Nix.
Она позволяет нам достигнуть чего-то подобного тому, что мы сделали императивно.

> Follows the definition of fixed point in [nixpkgs](https://github.com/NixOS/nixpkgs/blob/f224a4f1b32b3e813783d22de54e231cd8ea2448/lib/fixed-points.nix#L19):

Взгляните на определение неподвижной точки в [nixpkgs](https://github.com/NixOS/nixpkgs/blob/f224a4f1b32b3e813783d22de54e231cd8ea2448/lib/fixed-points.nix#L19):

```nix
{
  # Take a function and evaluate it with its own returned value.
  fix =
    f:
    let
      result = f result;
    in
    result;
}
```

> It\'s a function that accepts a function `f`, calls `f result` on the result just returned by `f result` and returns it.
> In other words it\'s `f(f(f(....`

Это функция, которая принимает функцию `f`, вызывает `f result`, где `result` возвращается из вызова `f result` и возвращает его.
Иными словами, это `f(f(f(...`

> At first sight, it\'s an infinite loop.
> With lazy evaluation it isn\'t, because the call is done only when needed.

На первый взгляд, это бесконечный цикл.
Но с ленивым выполнением — нет, поскольку вызов осуществляется только если он нужен.

```bash
nix-repl> fix = f: let result = f result; in result
nix-repl> pkgs = self: { a = 3; b = 4; c = self.a+self.b; }
nix-repl> fix pkgs
{ a = 3; b = 4; c = 7; }
```

> Without the `rec` keyword, we were able to refer to `a` and `b` of the same set.

У нас получилось сослаться на `a` и `b` из того же набора без ключевого слова `rec`.

> -   First `pkgs` gets called with an unevaluated thunk `(pkgs(pkgs(...)`
> -   To set the value of `c` then `self.a` and `self.b` are evaluated.
> -   The `pkgs` function gets called again to get the value of `a` and `b`.

- Сначала вызывается `pkgs` с невычисленными параметрами `(pkgs(pkgs(...)`
- Чтобы установить значение `c`, вычисляются `self.a` и `self.b`.
- Функция `pkgs` вызывается снова, чтобы получить значения `a` и `b`.

> The trick is that `c` is not needed to be evaluated in the inner call, thus it doesn\'t go in an infinite loop.

Трюк заключается в том, что значение `c` не нужно во внутреннем вызове, поэтому мы не входим в бесконечный цикл.

> Won\'t go further with the explanation here.
> A good post about fixed point and Nix can be [found here](http://r6.ca/blog/20140422T142911Z.html).

Не стану вдаваться здесь в дальнейшие объяснения.
Хороший пост о неподвижной точке в Nix может быть [найден здесь](http://r6.ca/blog/20140422T142911Z.html).

> ### Overriding a set with fixed point

### Переопределение набора с помощью неподвижной точки

> Given that `self.a` and `self.b` refer to the passed set and not to the literal set in the function, we\'re able to override both `a` and `b` and get a new value for `c`:

Учитывая, что `self.a` и `self.b` ссылаются на переданный набор, а не на буквальный набор в функции, мы можем переопределить и `a`, и `b`, и получить новое значение для `c`:

```nix
nix-repl> overrides = { a = 1; b = 2; }
nix-repl> let newpkgs = pkgs (newpkgs // overrides); in newpkgs
{ a = 3; b = 4; c = 3; }
nix-repl> let newpkgs = pkgs (newpkgs // overrides); in newpkgs // overrides
{ a = 1; b = 2; c = 3; }
```

> In the first case we computed pkgs with the overrides, in the second case we also included the overridden attributes in the result.

В первом случае мы вычислили `pkgs` с переопределённым набором, во втором мы также включили переопределённые атрибуты в результат.

> ## Overriding nixpkgs packages

## Переопределение пакетов nixpkgs

> We\'ve seen how to override attributes in a set such that they get recursively picked by dependent attributes.
> This approach can be used for derivations too, after all `nixpkgs` is a giant set of attributes that depend on each other.

Мы увидели, как переопределять атрибуты в наборе так, чтобы они рекурсивно извлекались атрибутами, которые зависят от них.
Этот подход можно использовать и для дериваций, поскольку `nixpkgs` — это гигантский набор атрибутов, которые зависят друг от друга.

> To do this, `nixpkgs` offers `config.packageOverrides`.
> So `nixpkgs` returns a fixed point of the package set, and `packageOverrides` is used to inject the overrides.

Для этого `nixpkgs` предлагает `config.packageOverrides`.
Таким образом `nixpkgs` возвращает неподвижную точку набора пакетов, а `packageOverrides` используется для внедрения переопределений.

> Create a `config.nix` file like this somewhere:

Создайте файл `config.nix` следующего вида:

```nix
{
    packageOverrides = pkgs: {
    graphviz = pkgs.graphviz.override {
      # запретить поддержку xorg
      withXorg = false;
    };
  };
}
```

> Now we can build e.g. asciidoc-full and it will automatically use the overridden graphviz:

Теперь мы можем собрать, скажем, `asciidoc-full` и он автоматически использует переопределённый `graphviz`:

```bash
nix-repl> pkgs = import <nixpkgs> { config = import ./config.nix; }
nix-repl> :b pkgs.asciidoc-full
```

> Note how we pass the `config` with `packageOverrides` when importing `nixpkgs`.
> Then `pkgs.asciidoc-full` is a derivation that has graphviz input (`pkgs.asciidoc` is the lighter version and doesn\'t use graphviz at all).

Обратите внимание, как мы передаём `config` с атрибутом `packageOverrides`, когда импортируем `nixpkgs`.
Здесь `pkgs.asciidoc-full` — это деривация с входящим параметром `graphviz` (при этом `pkgs.asciidoc` — это облегчённая версия, которая не использует `graphviz` совсем).

> Since there\'s no version of asciidoc with graphviz without X support in the binary cache, Nix will recompile the needed stuff for you.

Поскольку в кэше исполняемых файлов нет версии `asciidoc` с `graphviz` бзе поддержки X Window System, Nix перекомпилирует нужные вам компоненты.

> ## The ~/.config/nixpkgs/config.nix file

> ## Файл ~/.config/nixpkgs/config.nix

> In the previous pill we already talked about this file.
> The above `config.nix` that we just wrote could be the content of `~/.config/nixpkgs/config.nix` (or the deprecated location `~/.nixpkgs/config.nix`).

Мы уже обсуждали этот файл в предыдущей пилюле.
Содержимым файла `~/.config/nixpkgs/config.nix` может быть что-то похожее на `config.nix`, который мы только что написали.
Раньше этот файл находился в `~/.nixpkgs/config.nix`.

> Instead of passing it explicitly whenever we import `nixpkgs`, it will be automatically [imported by nixpkgs](https://github.com/NixOS/nixpkgs/blob/32c523914fdb8bf9cc7912b1eba023a8daaae2e8/pkgs/top-level/impure.nix#L28).

Вместо явной передачи всякий раз, когда мы импортируем `nixpkgs`, он автоматически [импортируется в `nixpkgs`](https://github.com/NixOS/nixpkgs/blob/32c523914fdb8bf9cc7912b1eba023a8daaae2e8/pkgs/top-level/impure.nix#L28).

> ## Conclusion

## Заключение

> We\'ve learned about a new design pattern: using fixed point for overriding packages in a package set.

Мы узнали о новом паттерне проектирования: использовании неподвижной точки для переопределения пакетов в наборе пакетов.

> Whereas in an imperative setting, like with other package managers, a library is installed replacing the old version and applications will use it, in Nix it\'s not that straight and simple.
> But it\'s more precise.

В отличие от имеративных настроек, которые используются в других пакетных менеджерах, где библиотека устанавливается поверх старой версии и приложения начинают использовать её, в Nix всё не так просто и прямолинейно.
Но более аккуратно.

> Nix applications will depend on specific versions of libraries, hence the reason why we have to recompile asciidoc to use the new graphviz library.

Приложения Nix зависят от определённых версий библиотек, вот почему нам приходится перекомпилировать `asciidoc`, чтобы использовать новую библиотеку `graphviz`.

> The newly built asciidoc will depend on the new graphviz, and old asciidoc will keep using the old graphviz undisturbed.

Новая версия `asciidoc` будет зависеть от новой версии `graphviz`, а старая версия `asciidoc` без помех сможет продолжать использовать старую версию `graphviz`.

> ## Next pill

## В следующей пилюле

> ...we will stop studying `nixpkgs` for a moment and talk about store paths.
> How does Nix compute the path in the store where to place the result of builds?
> How to add files to the store for which we have an integrity hash?
