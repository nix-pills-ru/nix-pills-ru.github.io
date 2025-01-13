> # Fundamentals of Stdenv

# Основы StdEnv

> Welcome to the 19th Nix pill.
> In the previous [18th](#nix-store-paths) pill we dived into the algorithm used by Nix to compute the store paths, and also introduced fixed-output store paths.

Добро пожаловать на девятнадцатую пилюлю Nix.
В предыдущей [восемнадцатой пилюле](18-nix-store-pathsmd) мы погрузились в алгоритм, который Nix использует для вычисления путей хранения, а также познакомились с фиксированными выходными путями хранения.

> This time we will instead look into `nixpkgs`, in particular one of its core derivations: `stdenv`.

Сейчас настало время исследовать `nixpkgs`, а конкретно, одну из его основных дериваций — `stdenv`.

> The `stdenv` is not treated as a special derivation by Nix, but it\'s very important for the `nixpkgs` repository.
> It serves as a base for packaging software.
> It is used to pull in dependencies such as the GCC toolchain, GNU make, core utilities, patch and diff utilities, and so on: basic tools needed to compile a huge pile of software currently present in `nixpkgs`.

Деривация `stdenv` не считается какой-то особенной в Nix, но она очень важна для репозитория `nixpkgs`.
Она служит основой для создания пакетов.
Она используется, чтобы подгрузить зависимости от инструментария GCC, GNU make, утилит ядра, утилит `patch` и `diff` и т. д.: основные утилиты нужные для сборки огромной кучи программ, в настоящий момент присутствующих в данный момент в `nixpkgs`.

> ## What is stdenv?

## Что такое `stdenv`?

> First of all, `stdenv` is a derivation, and it\'s a very simple one:

Прежде всего, `stdenv` — это деривация и она очень простая:

```bash
$ nix-build '<nixpkgs>' -A stdenv
/nix/store/k4jklkcag4zq4xkqhkpy156mgfm34ipn-stdenv
$ ls -R result/
result/:
nix-support/  setup

result/nix-support:
propagated-user-env-packages
```

> It has just two files: `/setup` and `/nix-support/propagated-user-env-packages`.
> Don\'t worry about the latter.
> It\'s empty, in fact.
> The important file is `/setup`.

В ней всего лишь два файла: `/setup` и `/nix-support/propagated-user-env-packages`.
Последний можно не принимать во внимание.
По факту, он пустой.
Важный файл — это `/setup`.

> How can this simple derivation pull in all of the toolchain and basic tools needed to compile packages?
> Let\'s look at the runtime dependencies:

Как такая простая деривация может включать в себя весь набор инструментов и базовых утилит для сборки пакетов?
Давайте взглянем на зависимости времени выполнения:

```bash
$ nix-store -q --references result
/nix/store/3a45nb37s0ndljp68228snsqr3qsyp96-bzip2-1.0.6
/nix/store/a457ywa1haa0sgr9g7a1pgldrg3s798d-coreutils-8.24
/nix/store/zmd4jk4db5lgxb8l93mhkvr3x92g2sx2-bash-4.3-p39
/nix/store/47sfpm2qclpqvrzijizimk4md1739b1b-gcc-wrapper-4.9.3
...
```

> How can it be?
> The package must be referring to those other packages somehow.
> In fact, they are hardcoded in the `/setup` file:

Как такое может быть?
Пакет должен как-то ссылаться на другие пакеты.
По факту, они прописаны в файле `/setup`:

```bash
$ head result/setup
export SHELL=/nix/store/zmd4jk4db5lgxb8l93mhkvr3x92g2sx2-bash-4.3-p39/bin/bash
initialPath="/nix/store/a457ywa1haa0sgr9g7a1pgldrg3s798d-coreutils-8.24 ..."
defaultNativeBuildInputs="/nix/store/sgwq15xg00xnm435gjicspm048rqg9y6-patchelf-0.8 ..."
```

> ## The setup file

## Файл `/setup`

> Remember our generic `builder.sh` in [Pill 8](#generic-builders)?
> It sets up a basic `PATH`, unpacks the source and runs the usual autotools commands for us.

Помните наш обобщённый скрипт `builder.sh` из [восьмой пилюли](08-generic-builders.md)?
Он инициализирует переменную `PATH`, распаковывает исходники и запускает для нас обычные команды `autotools`.

> The [stdenv setup file](https://github.com/NixOS/nixpkgs/blob/master/pkgs/stdenv/generic/setup.sh) is exactly that.
> It sets up several environment variables like `PATH` and creates some helper bash functions to build a package.
> I invite you to read it.

[Файл `/setup` из `stdenv`](https://github.com/NixOS/nixpkgs/blob/master/pkgs/stdenv/generic/setup.sh) — точно такой же.
Он инициализирует несколько переменных окружения, таких как `PATH` и создаёт несколько вспомогательных функций для `bash` для сборки пакетов.
Приглашаю вас его прочитать.

> The hardcoded toolchain and utilities are used to initially fill up the environment variables so that it\'s more pleasant to run common commands, similar to what we did with our builder with `baseInputs` and `buildInputs`.

Прописанные наборы инструментов и утилиты используются для начального заполнения переменных окружения, чтобы было удобно запускать основные команды, подобно тому, как мы делали с нашим скриптом сборки с `baseInputs` и `buildInputs`.

> The build with `stdenv` works in phases.
> Phases are like `unpackPhase`, `configurePhase`, `buildPhase`, `checkPhase`, `installPhase`, `fixupPhase`.
> You can see the default list in the `genericBuild` function.

Сборка в `stdenv` разбита на фазы.
Фазы — это `unpackPhase`, `configurePhase`, `buildPhase`, `checkPhase`, `installPhase`, `fixupPhase`.
Список по умолчанию можно увидеть в функции `genericBuild`.

> What `genericBuild` does is just run these phases.
> Default phases are just bash functions.
> You can easily read them.

Функция просто запускает эти фазы.
Фазы по-умолчанию — всего лишь функции `bash`.
Вы легко можете прочитать их.

> Every phase has hooks to run commands before and after the phase has been executed.
> Phases can be overwritten, reordered, whatever, it\'s just bash code.

У каждой фазы есть хуки для запуска команд до или после выполнения фазы.
Фазы могут быть переопределены, переставлены местами, да что угодно, это просто код на `bash`.

> How to use this file?
> Like our old builder.
> To test it, we enter a fake empty derivation, source the `stdenv` `setup`, unpack the hello sources and build it:

Как использовать этот файл?
Как наш старый скрипт сборки.
Чтобы проверить его, мы сделаем фиктивную пустую деривацию, запустим `setup` из `stdenv` с помощью `source`, распакуем исходники `hello` и скомпилируем их:

```bash
$ nix-shell -E 'derivation { name = "fake"; builder = "fake"; system = "x86_64-linux"; }'
nix-shell$ unset PATH
nix-shell$ source /nix/store/k4jklkcag4zq4xkqhkpy156mgfm34ipn-stdenv/setup
nix-shell$ tar -xf hello-2.10.tar.gz
nix-shell$ cd hello-2.10
nix-shell$ configurePhase
...
nix-shell$ buildPhase
...
```

> *I unset `PATH` to further show that the `stdenv` is sufficiently self-contained to build autotools packages that have no other dependencies.*

*Я очистил `PATH`, чтобы ещё раз показать, что `stdenv` имеет всё необходимое для сборки пакетов `autotools`, не имеющих других зависимостей.*

> So we ran the `configurePhase` function and `buildPhase` function and they worked.
> These bash functions should be self-explanatory.
> You can read the code in the `setup` file.

Мы запустили функции `configurePhase` и `buildPhase`, и он выполнились.
Эти функции `bash` имеют «говорящие» имена.
С кодом вы можете ознакомиться в файле `setup`.

> ## How the setup file is built

## Как создаётся файл `setup`

> Until now we worked with plain bash scripts.
> What about the Nix side?
> The `nixpkgs` repository offers a useful function, like we did with our old builder.
> It is a wrapper around the raw derivation function which pulls in the `stdenv` for us, and runs `genericBuild`.
> It\'s [stdenv.mkDerivation](https://github.com/NixOS/nixpkgs/blob/master/pkgs/stdenv/generic/make-derivation.nix).

До сих пор мы работали с обычными скриптами `bash`.
А что насчёт Nix?
В репозитории `nixpkgs` есть полезная функция, похожая на ту, которую мы написали в нашем старом скрипте.
Она вызывает функцию деривации, подтягивая `stdenv` и запуская `genericBuild`.
Это [stdenv.mkDerivation](https://github.com/NixOS/nixpkgs/blob/master/pkgs/stdenv/generic/make-derivation.nix).

> Note how `stdenv` is a derivation but it\'s also an attribute set which contains some other attributes, like `mkDerivation`.
> Nothing fancy here, just convenience.

Обратите внимание, что `stdenv` — это не только деривация, но и набор атрибутов, содержащий некоторые другие атрибуты, такие как `mkDerivation`.
Это сделано для удобства.

> Let\'s write a `hello.nix` expression using this newly discovered `stdenv`:

Давайте напишем выражение `hello.nix`, используя `stdenv`:

```nix
with import <nixpkgs> { };
stdenv.mkDerivation {
  name = "hello";
  src = ./hello-2.10.tar.gz;
}
```

> Don\'t be scared by the `with` expression.
> It pulls the `nixpkgs` repository into scope, so we can directly use `stdenv`.
> It looks very similar to the hello expression in [Pill 8](#generic-builders).

Не пугайтесь выражения `with`.
Оно подгружает репозиторий `nixpkgs` в область видимости, чтобы мы могли напрямую использовать `stdenv`.
Это выглядит очень похожим на выражение `hello` из [пилюли 8](08-generic-builders.md).

> It builds, and runs fine:

Программа отлично собирается и работает:

```bash
$ nix-build hello.nix
...
/nix/store/6y0mzdarm5qxfafvn2zm9nr01d1j0a72-hello
$ result/bin/hello
Hello, world!
```

> ## The stdenv.mkDerivation builder

## Сборщик `stdenv.mkDerivation`

> Let\'s take a look at the builder used by `mkDerivation`.
> You can read the code [here in nixpkgs](https://github.com/NixOS/nixpkgs/blob/master/pkgs/stdenv/generic/make-derivation.nix):

Взглянем на скрипт сборки, используемый `mkDerivation`.
Вы можете найти код [здесь в `nixpkgs`](https://github.com/NixOS/nixpkgs/blob/master/pkgs/stdenv/generic/make-derivation.nix):

```nix
{
  # ...
  builder = attrs.realBuilder or shell;
  args =
    attrs.args or [
      "-e"
      (attrs.builder or ./default-builder.sh)
    ];
  stdenv = result;
  # ...
}
```

> Also take a look at our old derivation wrapper in previous pills!
> The builder is bash (that shell variable), the argument to the builder (bash) is `default-builder.sh`, and then we add the environment variable `$stdenv` in the derivation which is the `stdenv` derivation.

Взглянем также на нашу старую обёртку над деривациями из предыдущих пилюль!
В данном случае пакет собирается с помощью `bash` (переменная `shell`), аргументом которого является `default-builder.sh` и затем мы добавляем переменную кружения `$stdenv` в деривацию, которая является деривацией `stdenv`.

> You can open [default-builder.sh](https://github.com/NixOS/nixpkgs/blob/master/pkgs/stdenv/generic/default-builder.sh) and see what it does:

Вы можете открыть [default-builder.sh](https://github.com/NixOS/nixpkgs/blob/master/pkgs/stdenv/generic/default-builder.sh) и посмотреть, что он делает:

```bash
source $stdenv/setup
genericBuild
```

> It\'s what we did in [Pill 10](#developing-with-nix-shell) to make the derivations `nix-shell` friendly.
> When entering the shell, the setup file only sets up the environment without building anything.
> When doing `nix-build`, it actually runs the build process.

Это то же самое, что мы делали в [Пилюле 10](10-developing-with-nix-shell.md) чтобы сделать утилиту `nix-shell` дружелюбной для дериваций.
При входе в оболочку, файл `setup` только настраивает окружение, но не запускает сборку.
А при запуске `nix-build` он действительно запускает процесс сборки.

> To get a clear understanding of the environment variables, look at the .drv of the hello derivation:

Чтобы получить ясное понимание переменных окружения, взгляните на файл `.drv` деривации `hello`:

```bash
$ nix derivation show $(nix-instantiate hello.nix)
warning: you did not specify '--add-root'; the result might be removed by the garbage collector
{
  "/nix/store/abwj50lycl0m515yblnrvwyydlhhqvj2-hello.drv": {
    "outputs": {
      "out": {
        "path": "/nix/store/6y0mzdarm5qxfafvn2zm9nr01d1j0a72-hello"
      }
    },
    "inputSrcs": [
      "/nix/store/9krlzvny65gdc8s7kpb6lkx8cd02c25b-default-builder.sh",
      "/nix/store/svc70mmzrlgq42m9acs0prsmci7ksh6h-hello-2.10.tar.gz"
    ],
    "inputDrvs": {
      "/nix/store/hcgwbx42mcxr7ksnv0i1fg7kw6jvxshb-bash-4.4-p19.drv": [
        "out"
      ],
      "/nix/store/sfxh3ybqh97cgl4s59nrpi78kgcc8f3d-stdenv-linux.drv": [
        "out"
      ]
    },
    "platform": "x86_64-linux",
    "builder": "/nix/store/q1g0rl8zfmz7r371fp5p42p4acmv297d-bash-4.4-p19/bin/bash",
    "args": [
      "-e",
      "/nix/store/9krlzvny65gdc8s7kpb6lkx8cd02c25b-default-builder.sh"
    ],
    "env": {
      "buildInputs": "",
      "builder": "/nix/store/q1g0rl8zfmz7r371fp5p42p4acmv297d-bash-4.4-p19/bin/bash",
      "configureFlags": "",
      "depsBuildBuild": "",
      "depsBuildBuildPropagated": "",
      "depsBuildTarget": "",
      "depsBuildTargetPropagated": "",
      "depsHostBuild": "",
      "depsHostBuildPropagated": "",
      "depsTargetTarget": "",
      "depsTargetTargetPropagated": "",
      "name": "hello",
      "nativeBuildInputs": "",
      "out": "/nix/store/6y0mzdarm5qxfafvn2zm9nr01d1j0a72-hello",
      "propagatedBuildInputs": "",
      "propagatedNativeBuildInputs": "",
      "src": "/nix/store/svc70mmzrlgq42m9acs0prsmci7ksh6h-hello-2.10.tar.gz",
      "stdenv": "/nix/store/6kz2vbh98s2r1pfshidkzhiy2s2qdw0a-stdenv-linux",
      "system": "x86_64-linux"
    }
  }
}
```

> It\'s so short I decided to paste it entirely above.
> The builder is bash, with `-e default-builder.sh` arguments.
> Then you can see the `src` and `stdenv` environment variables.

Он настолько короткий, что я решил вставить его полностью.
Программа сборки это `bash` с аргументами `-d default-builder.sh`.
Здесь вы можете видеть переменные окружения `src` и `stdenv`.

> The last bit, the `unpackPhase` in the setup, is used to unpack the sources and enter the directory.
> Again, like we did in our old builder.

Последняя нерассмотренная нами фаза `unpackPhase` в `setup` используется для распаковки исходных кодов и перехода в каталог.
И снова — здесь всё точно также, как в наших старых скриптах сборки.

> ## Conclusion

## Заключение

> The `stdenv` is the core of the `nixpkgs` repository.
> All packages use the `stdenv.mkDerivation` wrapper instead of the raw derivation.
> It does a bunch of operations for us and also sets up a pleasant build environment.

Деривация `stdenv` — это ядро репозитория `nixpkgs`.
Все пакеты используют обёртку `stdenv.mkDerivation` вместо прямого вызова дериваций.
Она выполняет для нас несколько операций а также создаёт удобное окружение для сборки.

> The overall process is simple:

Процесс в целом прост:

-   `nix-build`
-   `bash -e default-builder.sh`
-   `source $stdenv/setup`
-   `genericBuild`


> That\'s it.
> Everything you need to know about the stdenv phases is in the [setup file](https://github.com/NixOS/nixpkgs/blob/master/pkgs/stdenv/generic/setup.sh).

И это всё.
Всё, что вам надо знать о фазах `stdenv` есть в [файле `setup`](https://github.com/NixOS/nixpkgs/blob/master/pkgs/stdenv/generic/setup.sh).

> Really, take your time to read that file.
> Don\'t forget that juicy docs are also available in the [nixpkgs manual](http://nixos.org/nixpkgs/manual/#chap-stdenv).

Серьёзно, найдите время его прочитать.
И не забывайте, что в [руководстве по `nixpkgs`](http://nixos.org/nixpkgs/manual/#chap-stdenv) тоже есть «сладкие» документы.

> ## Next pill\...

## В следующей пилюле

> \...we will talk about how to add dependencies to our packages with `buildInputs` and `propagatedBuildInputs`, and influence downstream builds with setup hooks and env hooks.
> These concepts are crucial to how `nixpkgs` packages are composed.

Мы поговорим о том, как добавить зависимости в наши пакеты с помощью `buildInputs` и `propagatedBuildInputs`, и влиять на последующие сборки с помощью хуков настройки и хуков окружения.
Эти концепции очень важны для понимания, как `nixpkgs` собирает пакеты.
