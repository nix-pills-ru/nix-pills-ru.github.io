# Основы `stdenv`

Добро пожаловать на девятнадцатую пилюлю Nix.
В предыдущей [восемнадцатой пилюле](18-nix-store-pathsmd) мы изучили алгоритм, который Nix использует для генерации путей хранения и узнали, как он работает с фиксированными выходными путями.

Настало время исследовать `nixpkgs` и, конкретно, одну из его основных дериваций — `stdenv`.

Деривация `stdenv` не считается какой-то особенной, но очень важна для репозитория `nixpkgs`.
Она служит базой для создания пакетов, поскольку подгружает зависимости для инструментария GCC, GNU make, утилит ядра, утилит `patch` и `diff` и т. д.
Благодаря ей, нам доступны основные утилиты, необходимые для сборки огромной кучи программ, присутствующих в `nixpkgs` в данный момент.

## Что такое `stdenv`?

Прежде всего, `stdenv` — это очень простая деривация:

```bash
$ nix-build '<nixpkgs>' -A stdenv
/nix/store/k4jklkcag4zq4xkqhkpy156mgfm34ipn-stdenv
$ ls -R result/
result/:
nix-support/  setup

result/nix-support:
propagated-user-env-packages
```

В ней всего лишь два файла: `/setup` и `/nix-support/propagated-user-env-packages`.
Последний можно не принимать во внимание, поскольку, по факту, он пустой.
Действительно важный файл — это `/setup`.

Как такая простая деривация может включать в себя весь набор инструментов и базовых утилит для сборки пакетов?
Взглянем на зависимости времени выполнения:

```bash
$ nix-store -q --references result
/nix/store/3a45nb37s0ndljp68228snsqr3qsyp96-bzip2-1.0.6
/nix/store/a457ywa1haa0sgr9g7a1pgldrg3s798d-coreutils-8.24
/nix/store/zmd4jk4db5lgxb8l93mhkvr3x92g2sx2-bash-4.3-p39
/nix/store/47sfpm2qclpqvrzijizimk4md1739b1b-gcc-wrapper-4.9.3
...
```

Как такое может быть?
Пакет должен как-то ссылаться на другие пакеты.
Оказывается, зависимости прописаны в файле `/setup`:

```bash
$ head result/setup
export SHELL=/nix/store/zmd4jk4db5lgxb8l93mhkvr3x92g2sx2-bash-4.3-p39/bin/bash
initialPath="/nix/store/a457ywa1haa0sgr9g7a1pgldrg3s798d-coreutils-8.24 ..."
defaultNativeBuildInputs="/nix/store/sgwq15xg00xnm435gjicspm048rqg9y6-patchelf-0.8 ..."
```

## Файл `/setup`

Помните наш обобщённый скрипт `builder.sh` из [восьмой пилюли](08-generic-builders.md)?
Он инициализировал переменную `PATH`, распаковывал исходники и запускал для нас команды `autotools`.

[Файл `/setup` из `stdenv`](https://github.com/NixOS/nixpkgs/blob/master/pkgs/stdenv/generic/setup.sh) — точно такой же.
Он инициализирует несколько переменных окружения, таких как `PATH` и создаёт несколько вспомогательных функций `bash` для сборки пакетов.
Давайте его прочитаем.

Чтобы было удобно запускать команды, в файле `/setup` для наборов инструментов и утилит прописаны переменные окружения.
Подобный трюк мы проворачивали с `baseInputs` и `buildInputs`, когда создавали универсальный скрипт сборки.

Сборка в `stdenv` разбита на фазы: `unpackPhase`, `configurePhase`, `buildPhase`, `checkPhase`, `installPhase`, `fixupPhase`.
Список фаз по умолчанию находится в функции `genericBuild`.

Функция запускает эти фазы одну за другой.
Все фазы — это, на самом деле, функции `bash`, так что вы легко можете их прочитать.

У каждой фазы есть хуки для запуска команд до или после фазы.
Фазы можно переопределить, переставить местами, да и в принципе сделать с ними что угодно, так как это просто код на `bash`.

Как использовать этот файл?
Как наш старый скрипт сборки.
Чтобы проверить его, сделаем фиктивную пустую деривацию, запустим `setup` из `stdenv` с помощью `source`, распакуем исходники `hello` и скомпилируем их:

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

*Я очистил `PATH`, чтобы ещё раз показать, что в `stdenv` есть всё, что нужно для сборки пакетов `autotools`, не имеющих других зависимостей.*

Мы запустили функции `configurePhase` и `buildPhase`, и они отработали.
Эти функции `bash` имеют «говорящие» имена, их код вы можете найти в файле `setup`.

## Из чего состоит `setup`

До сих пор мы работали с обычными скриптами `bash`.
А что насчёт Nix?
В репозитории `nixpkgs` есть полезная функция, похожая на ту, которую мы написали в нашем старом скрипте.
Она вызывает функцию деривации, подтягивая `stdenv` и запуская `genericBuild`.
Это [stdenv.mkDerivation](https://github.com/NixOS/nixpkgs/blob/master/pkgs/stdenv/generic/make-derivation.nix).

Обратите внимание, что `stdenv` — это не только деривация, но и набор атрибутов, содержащий, в том числе, `mkDerivation`.
Это сделано для удобства.

Давайте напишем выражение `hello.nix`, используя `stdenv`:

```nix
with import <nixpkgs> { };
stdenv.mkDerivation {
  name = "hello";
  src = ./hello-2.10.tar.gz;
}
```

Не пугайтесь выражения `with`.
Оно подгружает репозиторий `nixpkgs` в область видимости, чтобы мы могли напрямую использовать `stdenv`.
Это очень похоже на выражение `hello` из [пилюли 8](08-generic-builders.md).

Программа отлично собирается и работает:

```bash
$ nix-build hello.nix
...
/nix/store/6y0mzdarm5qxfafvn2zm9nr01d1j0a72-hello
$ result/bin/hello
Hello, world!
```

## Сборщик `stdenv.mkDerivation`

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

Сравним его с нашей старой обёрткой над деривациями из предыдущих пилюль.
Здесь пакет собирается с помощью `bash` (переменная `shell`), аргументом которого является `default-builder.sh`.
В набор атрибутов, уже присутствующему в деривации `stdenv`, мы добавляем переменную окружения `$stdenv`.

Вы можете открыть [default-builder.sh](https://github.com/NixOS/nixpkgs/blob/master/pkgs/stdenv/generic/default-builder.sh) и посмотреть, что он делает:

```bash
source $stdenv/setup
genericBuild
```

То же самое мы делали в [десятой пилюлей](10-developing-with-nix-shell.md) чтобы из `nix-shell` было удобно работать с деривациями.
При входе в оболочку, файл `setup` настраивает окружение, но не запускает сборку.
А при запуске `nix-build` он действительно запускает процесс сборки.

Чтобы получить ясное представление о переменных окружения, загляните в файл `hello.drv`:

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

Он настолько короткий, что я решил вставить его полностью.
Программа сборки — это `bash` с аргументами `-d default-builder.sh`.
В файле вы видите переменные окружения `src` и `stdenv`.

Последняя фаза, с которой мы пока не сталкивались — это `unpackPhase`.
В `setup` она используется для распаковки исходных кодов и перехода в каталог.
Здесь снова всё точно также, как в наших старых скриптах сборки.

## Заключение

Деривация `stdenv` — это ядро репозитория `nixpkgs`.
Все пакеты используют обёртку `stdenv.mkDerivation` вместо прямого вызова дериваций.
Она выполняет разные операции и создаёт удобное окружение для сборки.

Процесс в целом прост:

-   `nix-build`
-   `bash -e default-builder.sh`
-   `source $stdenv/setup`
-   `genericBuild`


И это всё.
То, что вам надо знать о фазах `stdenv` есть в [файле `setup`](https://github.com/NixOS/nixpkgs/blob/master/pkgs/stdenv/generic/setup.sh).

Серьёзно, найдите время его прочитать.
И не забывайте, что в [руководстве по `nixpkgs`](http://nixos.org/nixpkgs/manual/#chap-stdenv) тоже есть важные документы.

## В следующей пилюле

Мы поговорим о том, как добавить зависимости в наши пакеты с помощью `buildInputs` и `propagatedBuildInputs`, и о том, как влиять на зависимые сборки с помощью хуков настройки и окружения.
Эти концепции очень важны для понимания, как `nixpkgs` собирает пакеты.
