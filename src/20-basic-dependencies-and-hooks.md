> # Basic Dependencies and Hooks

> Welcome to the 20th Nix pill.
> In the previous [19th](#fundamentals-of-stdenv) pill we introduced Nixpkgs\' stdenv, including `setup.sh` script, `default-builder.sh` helper script, and `stdenv.mkDerivation` builder.
> We focused on how stdenv is put together, and how it\'s used, and a bit about the phases of `genericBuild`.

Добро пожаловать на двадцатую пилюлю Nix.
В предыдущей [девятнадцатой пилюле](19-fundamentals-of-stdenv.md) мы познакомились с деривацией `stdenv`, включающей скрипт `setup.sh`, вспомогательный скрипт `default-builder.sh` и функцию сборки `stdenv.mkDerivation`.
Мы сосредоточились на том, как `stdenv` сводит всё это вместе, как он используется и немного — на фазах `genericBuild`.

> This time, we\'ll focus on the interaction of packages built with `stdenv.mkDerivation`.
> Packages need to depend on each other, of course.
> For this we have `buildInputs` and `propagatedBuildInputs` attributes.
> We\'ve also found that dependencies sometimes need to influence their dependents in ways the dependents can\'t or shouldn\'t predict.
> For this we have setup hooks and env hooks.
> Together, these 4 concepts support almost all build-time package interactions.

Сейчас мы сосредоточимся на взаимодействие процесса сборки пакетов с `stdevn.mkDerivation`.
Естественно, пакеты должны зависеть друг от друга.
Для этого у нас есть атрибуты `buildInputs` и `propagatedBuildInputs`.
Так же мы выяснили, что иногда входные пакеты должны влиять на зависимые пакеты так, как зависимые пакеты не могли или не должны были предсказать.
Для этого у нас есть хуки установки и хуки окружения.
Все вмести эти 4 концепции обеспечивает почти любое взаимодействие при сборке пакета.

> > ::: note
> > The complexity of the dependencies and hooks infrastructure has increased, over time, to support cross compilation.
> > Once you learn the core concepts, you will be able to understand the extra complexity.
> > As a starting point, you might want to refer to nixpkgs commit [6675f0a5](https://github.com/nixos/nixpkgs/tree/6675f0a52c0962042a1000c7f20e887d0d26ae25), the last version of stdenv without cross-compilation complexity.
> > :::

> ℹ️ Сложность инфраструктуры зависимостей и хуков выросла с течением времени для того, чтобы поддержать кросс-компиляцию.
> Как только вы изучите основные концепции, вы сможете перейти к более сложным темам.
> В качестве стартовой точки вы можете изучить вот этот коммит[6675f0a5](https://github.com/nixos/nixpkgs/tree/6675f0a52c0962042a1000c7f20e887d0d26ae25) в `nixpkgs`, последняя версия `stdenv` без поддержки кросс-компиляции.

> ## The `buildInputs` Attribute

## Атрибут `buildInputs`

> For the simplest dependencies where the current package directly needs another, we use the `buildInputs` attribute.
> This is exactly the pattern used in our builder in [Pill 8](#generic-builders).
> To demo this, let\'s build GNU Hello, and then another package which provides a shell script that `exec`s it.

В случае простейшей зависимости, когда одному пакету нужен конкретный другой пакет, мы используем атрибут `buildInputs`.
Именно этот паттерн, который мы изучали в нашем скрипте сборки в [Пилюле 8](08-generic-builders.md).
Чтобы продемонстрировать это, давайте соберём пакет GNU Hello, а затем другой пакет, в котором будет скрипт, запускающий программу `hello`.

```nix
let

  nixpkgs = import <nixpkgs> { };

  inherit (nixpkgs) stdenv fetchurl which;

  actualHello = stdenv.mkDerivation {
    name = "hello-2.3";

    src = fetchurl {
      url = "mirror://gnu/hello/hello-2.3.tar.bz2";
      sha256 = "0c7vijq8y68bpr7g6dh1gny0bff8qq81vnp4ch8pjzvg56wb3js1";
    };
  };

  wrappedHello = stdenv.mkDerivation {
    name = "hello-wrapper";

    buildInputs = [
      actualHello
      which
    ];

    unpackPhase = "true";

    installPhase = ''
      mkdir -p "$out/bin"
      echo "#! ${stdenv.shell}" >> "$out/bin/hello"
      echo "exec $(which hello)" >> "$out/bin/hello"
      chmod 0755 "$out/bin/hello"
    '';
  };
in
wrappedHello
```

> Notice that the wrappedHello derivation finds the `hello` binary from the `PATH`.
> This works because stdenv contains something like:

Обратите внимание, что деривация `wrappedHello` находит программу `hello` через переменную `PATH`.
Это работает, потому что `stdenv` содержит что-то подобное:

```nix
pkgs=""
for i in $buildInputs; do
    findInputs $i
done
```

> where `findInputs` is defined like:

где `findInputs` определена, как:

```nix
findInputs() {
    local pkg=$1

    ## Don't need to repeat already processed package
    ## Не надо повторять для уже обработанных пакетов
    case $pkgs in
        *\ $pkg\ *)
            return 0
            ;;
    esac

    pkgs="$pkgs $pkg "

    ## More goes here in reality that we can ignore for now.
    ## Здесь на самом деле есть кое-что ещё, что мы можем пока игнорировать
}
```

> then after this is run:

потом после этого выполняется:

```nix
for i in $pkgs; do
    addToEnv $i
done
```

> where `addToEnv` is defined like:

где `addToEnv` определена как:

```nix
addToEnv() {
    local pkg=$1

    if test -d $1/bin; then
        addToSearchPath _PATH $1/bin
    fi

    ## More goes here in reality that we can ignore for now.
    ## Здесь на самом деле есть кое-что ещё, что мы можем пока игнорировать
}
```

> The `addToSearchPath` call adds `$1/bin` to `_PATH` if the former exists (code [here](https://github.com/NixOS/nixpkgs/blob/6675f0a52c0962042a1000c7f20e887d0d26ae25/pkgs/stdenv/generic/setup.sh#L60-L73)).
> Once all the packages in `buildInputs` have been processed, then content of `_PATH` is added to `PATH`, as follows:

Вызов `addToSearchPath` добавляет `$1/bin` к `_PATH`, если такой путь существует (код [здесь](https://github.com/NixOS/nixpkgs/blob/6675f0a52c0962042a1000c7f20e887d0d26ae25/pkgs/stdenv/generic/setup.sh#L60-L73)).
Как только все пакеты из `buildInputs` будут обработаны, содержимое `_PATH` добавляется в `PATH`, как в этом коде:

```nix
PATH="${_PATH-}${_PATH:+${PATH:+:}}$PATH"
```

> With the real `hello` on the `PATH`, the `installPhase` should hopefully make sense.

Если путь к `hello` прописан в `PATH`, фаза `installPhase` должна завершиться успешно.

> ## The `propagatedBuildInputs` Attribute

## Атрибут `propagatedBuildInputs`

> The `buildInputs` covers direct dependencies, but what about indirect dependencies where one package needs a second package which needs a third?
> Nix itself handles this just fine, understanding various dependency closures as covered in previous builds.
> But what about the conveniences that `buildInputs` provides, namely accumulating in `pkgs` environment variable and inclusion of `pkg/bin` directories on the `PATH`?
> For this, stdenv provides the `propagatedBuildInputs`:

`buildInputs` покрывает прямые зависимости, но как быть с косвенными зависимостями, когда одному пакету нужен другой пакет, которому нужен третий?
Nix и сам прекрасно с этим справляется, понимая различные замыкания зависимостей, возникших при сборке предыдущих пакетов.
Но что насчёт удобств, которые предоставляет `buildInputs`, а конкретно — аккумулирования каталогов `pkg/bin` в переменной окружения `pkgs` с последюущим включением их в `PATH`?
Для это `stdenv` предоставляет `propagatedBuildInputs`:

```nix
let

  nixpkgs = import <nixpkgs> { };

  inherit (nixpkgs) stdenv fetchurl which;

  actualHello = stdenv.mkDerivation {
    name = "hello-2.3";

    src = fetchurl {
      url = "mirror://gnu/hello/hello-2.3.tar.bz2";
      sha256 = "0c7vijq8y68bpr7g6dh1gny0bff8qq81vnp4ch8pjzvg56wb3js1";
    };
  };

  intermediary = stdenv.mkDerivation {
    name = "middle-man";

    propagatedBuildInputs = [ actualHello ];

    unpackPhase = "true";

    installPhase = ''
      mkdir -p "$out"
    '';
  };

  wrappedHello = stdenv.mkDerivation {
    name = "hello-wrapper";

    buildInputs = [
      intermediary
      which
    ];

    unpackPhase = "true";

    installPhase = ''
      mkdir -p "$out/bin"
      echo "#! ${stdenv.shell}" >> "$out/bin/hello"
      echo "exec $(which hello)" >> "$out/bin/hello"
      chmod 0755 "$out/bin/hello"
    '';
  };
in
wrappedHello
```

> See how the intermediate package has a `propagatedBuildInputs` dependency, but the wrapper only needs a `buildInputs` dependency on the intermediary.

Обратите внимание, что в пакете `intermediary` зависимость описана в `propagatedBuildInputs`, в то время, как `wrappedHello` получает зависимость через `buildInputs`.

> How does this work?
> You might think we do something in Nix, but actually it\'s done not at eval time but at build time in bash.
> let\'s look at part of the `fixupPhase` of stdenv:

Как это работает?
Вы можете подумать, что это каким-то образом делает Nix, но на самом деле это происходит не при выполнении кода, а во время сборки программы в `bash`.
Давайте взглянем на часть `fixupPhase` из `stdenv`:

```nix
fixupPhase() {

    ## Elided

    if test -n "$propagatedBuildInputs"; then
        mkdir -p "$out/nix-support"
        echo "$propagatedBuildInputs" > "$out/nix-support/propagated-build-inputs"
    fi

    ## Elided

}
```

> This dumps the propagated build inputs in a so-named file in `$out/nix-support/`.
> Then, back in `findInputs` look at the lines at the bottom we elided before:

Этот код сохраняет сборки, описанные в `propagatedBuildInputs` в одноимённом файле в каталоге `$out/nix-support`.
Теперь, возвращаясь обратно к `findInputs`, взгляните на строки внизу, которые мы ранее пропустили:

```nix
findInputs() {
    local pkg=$1

    ## More goes here in reality that we can ignore for now.
    ## Здесь на самом деле есть кое-что ещё, что мы можем пока игнорировать

    if test -f $pkg/nix-support/propagated-build-inputs; then
        for i in $(cat $pkg/nix-support/propagated-build-inputs); do
            findInputs $i
        done
    fi
}
```

> See how `findInputs` is actually recursive, looking at the propagated build inputs of each dependency, and those dependencies\' propagated build inputs, etc.

Посмотрите, что `findInputs` на самом деле рекурсивна: исследует `propagatedBuildInputs` каждой зависимости и далее `propagatedBuildInputs` этих зависимостей и т. д.

> We actually simplified the `findInputs` call site from before; `propagatedBuildInputs` is also looped over in reality:

Мы на самом деле упростили место вызова `findInputs` в прошлых примерах; На самом деле `propagatedBuildInputs` тоже зациклен:

```nix
pkgs=""
for i in $buildInputs $propagatedBuildInputs; do
    findInputs $i
done
```

> This demonstrates an important point. For the *current* package alone, it doesn\'t matter whether a dependency is propagated or not.
> It will be processed the same way: called with `findInputs` and `addToEnv`.
> (The packages discovered by `findInputs`, which are also accumulated in `pkgs` and passed to `addToEnv`, are also the same in both cases.)
> Downstream however, it certainly does matter because only the propagated immediate dependencies are put in the `$out/nix-support/propagated-build-inputs`.

Это демострирует важный момент. Для *текущего" пакета неважно, является ли зависимость propagated или нет.
Все они будут обработаны одним и тем же способом: вызваны через `findInputs` и переданы в `addToEnv`.
(Пакеты, найденные функций `findInputs`, которые собраны в `pkgs` и переданы в `addToEvn`, в обоих случае будут одни и те же.)
Однако в целом это безуслоно имеет значение, поскольку в `$out/nix-support/propagated-build-inputs` помещаются только непосредственные распространяемые зависимости.

> ## Setup Hooks

## Хуки установки

> As we mentioned above, sometimes dependencies need to influence the packages that use them in ways other than just *being* a dependency.
> [^1] `propagatedBuildInputs` can actually be seen as an example of this: packages using that are effectively \"injecting\" those dependencies as extra `buildInputs` in their downstream dependents.
> But in general, a dependency might affect the packages it depends on in arbitrary ways.
> *Arbitrary* is the key word here.
> We could teach `setup.sh` things about upstream packages like `pkg/nix-support/propagated-build-inputs`, but not arbitrary interactions.

Как мы уже упоминали выше, иногда зависимости должны влиять на пакеты, которые их используют, иными способами, чем просто *быть* зависимостью.
[^1] В качестве примера на самом деле может быть рассмотрен параметр `propagatedBuildInputs`: пакеты, которые его используют эффективно "внедряют" эти зависимости, как дополнительные `buildInputs` в нижестоящие зависимые пакеты.
Но, в целом, зависимости могут оказывать влияние на пакеты, которые от них зависят, произвольным образом.
*Произвольным* здесь ключевое слово.
Мы могли бы научить `setup.sh` вещам, касающимся вышестоящих пакетов, наподобие `pkg/nix-support/propagated-build-inputs`, но не произвольным взаимодействиям.

> Setup hooks are the basic building block we have for this.
> In nixpkgs, a \"hook\" is basically a bash callback, and a setup hook is no exception.
> Let\'s look at the last part of `findInputs` we haven\'t covered:

Хуки установки — основные строительные блоки, которые у нас для этого есть.
В `nixpkgs` "хук" по сути является функцией обратного вызова в `bash`, и хук установки не является исключением.
Давайте взглянем на последнюю часть `findInputs`, которую мы пока игнорировали:

```nix
findInputs() {
    local pkg=$1

    ## More goes here in reality that we can ignore for now.
    ## Здесь на самом деле есть кое-что ещё, что мы можем пока игнорировать

    if test -f $pkg/nix-support/setup-hook; then
        source $pkg/nix-support/setup-hook
    fi

    ## More goes here in reality that we can ignore for now.
    ## Здесь на самом деле есть кое-что ещё, что мы можем пока игнорировать

}
```

> If a package includes the path `pkg/nix-support/setup-hook`, it will be sourced by any stdenv-based build including that as a dependency.

Если в пакете есть скрипт с именем `pkg/nix-support/setup-hook`, он будет запущен с помощью команды `source` любым пакетом, основанным на `stdenv` и включающим первый пакет, как зависимость.

> This is strictly more general than any of the other mechanisms introduced in this chapter.
> For example, try writing a setup hook that has the same effect as a *propagatedBuildInputs* entry.
> One can almost think of this as an escape hatch around Nix\'s normal isolation guarantees, and the principle that dependencies are immutable and inert.
> We\'re not actually doing something unsafe or modifying dependencies, but we are allowing arbitrary ad-hoc behavior.
> For this reason, setup-hooks should only be used as a last resort.

Это гарантированно более общий, чем любой из остальных механизмов, описанных в этой главе.
Например, попробуйте написать хук установки с тем же эффектом, что и у параметра `propagatedBuildInputs`.
Это можно рассматривать, как аварийный выход и обычных гарантий изолированности Nix, и принчипа, что зависимости неизменны и инертны.
На самом деле мы не делаем ничего небезопасного и не модифицируем зависимости, но мы допускаем произвольное поведение для решения возникающих задач.
По этой причине, хуки установки следует применять только в крайнем случае.

> ## Environment Hooks

## Хуки окружения

> As a final convenience, we have environment hooks.
> Recall in [Pill 12](#inputs-design-pattern) how we created `NIX_CFLAGS_COMPILE` for `-I` flags and `NIX_LDFLAGS` for `-L` flags, in a similar manner to how we prepared the `PATH`.
> One point of ugliness was how anti-modular this was.
> It makes sense to build the `PATH` in a generic builder, because the `PATH` is used by the shell, and the generic builder is intrinsically tied to the shell.
> But `-I` and `-L` flags are only relevant to the C compiler.
> The stdenv isn\'t wedded to including a C compiler (though it does by default), and there are other compilers too which may take completely different flags.

Для окончательного удобства у нас есть хуки окружения.
Вспомните, как в [Пилюле 12](12-inputs-design-pattern.md) мы создали `NIX_CFLAGS_COMPILE` для флагов `-I` и `NIX_LDFLAGS` для флагов `-L` в менере, похожей на ту, где мы подговили `PATH`.
Одним из неприятных моментов оказалось то, насколько это было не-модульным.
Имеет смысле учитывать `PATH` в универсальном скрипте сборки, потому что `PATH` используется оболочкой и универсальный скрипт неразрывно связан с оболочкой.
Но флаги `-I` и `-L` относятся только к компилятору C.
Пакет `stdenv` не обязан как-то по особенному относиться к компилятору С (хотя, по факту, отностися), и существуют также другие компиляторы, у которых могут быть совершенно другие флаги.

> As a first step, we can move that logic to a setup hook on the C compiler; indeed that\'s just what we do in CC Wrapper.
> [^2] But this pattern comes up fairly often, so somebody decided to add some helper support to reduce boilerplate.
В качестве первого шага мы можем переместить эту логику в хук устновки для компилятора C; на самом деле именно это мы сделали в обёртке над CC.
[^2] Но этот паттерн встречается достаточно часто, так что кто-то решил добавить несколько вспомогательных функций, чтобы сократить объём кода.

> The other half of `addToEnv` is:

Вторая половина `addToEnv` выглядит так:

```nix
addToEnv() {
    local pkg=$1

    ## More goes here in reality that we can ignore for now.
    ## Здесь на самом деле есть кое-что ещё, что мы можем пока игнорировать

    # Run the package-specific hooks set by the setup-hook scripts.
    # Запустить специфичные для пакета хуки, установленные скриптами хуков установки
    for i in "${envHooks[@]}"; do
        $i $pkg
    done
}
```

> Functions listed in `envHooks` are applied to every package passed to `addToEnv`.
> One can write a setup hook like:

Функции, перечисленные в `envHooks` применяются к каждому пакету, переданному в `addToEnv`.
Можно написать такой хук установки:

```nix
anEnvHook() {
    local pkg=$1

    echo "I'm depending on \"$pkg\""
}

envHooks+=(anEnvHook)
```

> and if one dependency has that setup hook then all of them will be so `echo`ed.
> Allowing dependencies to learn about their *sibling* dependencies is exactly what compilers need.

и если какая-то зависимость имеет такой хук установки, все из них будут напечатаны.
Позволить зависимостям узнать о своих *родственных* зависимостях — именно то, что нужно компиляторам.

> ## Next pill\...

## В следующей пилюле

> ...I\'m not sure!
> We could talk about the additional dependency types and hooks which cross compilation necessitates, building on our knowledge here to cover stdenv as it works today.
> We could talk about how nixpkgs is bootstrapped.
> Or we could talk about how `localSystem` and `crossSystem` are elaborated into the `buildPlatform`, `hostPlatform`, and `targetPlatform` each bootstrapping stage receives.
> Let us know which most interests you!

...я не уверен!
Мы могли бы поговорить о дополнительных типах зависимостей и хуках, которые нужны при кросс-компиляции, опираясь на наши знания о том, как работает `stdenv`.
Мы могли бы поговорить о том, как происходит загрузка `nixpkgs`.
Или мы могли бы поговорить о том, как `localSystem` и `crossSystem` превращаются в `buildPlatform`, `hostPlatform` и `targetPlatform`, у каждой из которых есть свой этап загрузки.
Дайте мне знать, что вам интересно!

> [^1]: We can now be precise and consider what `addToEnv` does alone the minimal treatment of a dependency: i.e. a package that is *just* a dependency would *only* have `addToEnv` applied to it.

[^1]: Теперь мы можем точнее и утверждать, что `addToEnv` выполняет минимальную обработку зависимости, то есть пакет, который является *просто* зависимостью, будет применяться только функция `addToEnv`.

> [^2]: It was called [GCC Wrapper](https://github.com/NixOS/nixpkgs/tree/6675f0a52c0962042a1000c7f20e887d0d26ae25/pkgs/build-support/gcc-wrapper) in the version of nixpkgs suggested for following along in this pill;
>       Darwin and Clang support hadn\'t yet motivated the rename.

[^2]: В версии `nixpkgs`, актуальной на момент написания пилюли он назывался [Обёрткой над GCC](https://github.com/NixOS/nixpkgs/tree/6675f0a52c0962042a1000c7f20e887d0d26ae25/pkgs/build-support/gcc-wrapper); Поддержка компилятором Darwin и Clang не послужила достаточным основанием, чтобы его переименовать.