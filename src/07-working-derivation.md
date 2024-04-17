# Работающая деривация

## Введение

Добро пожаловать на седьмую пилюлю Nix. В предыдущей [шестой пилюле](06-our-first-derivation.md) мы начали разбираться с деривациями в языке Nix, выяснили, как определить пустую деривацию и (попытаться) её собрать. 

В этом посте мы продолжим двигаться тем же курсом, создав деривацию, которая действительно что-то собирает.
Затем мы попытаемся создать пакет с реальной программой: скомпилируем простой файл `.c` и созданим деривацию, используя классический инструментарий.

Напоминаю, как запускать окружение Nix: `source ~/.nix-profile/etc/profile.d/nix.sh`

## Использование скрипта для сборки

Каков простейший способ выполнить цепочку команд, чтобы что-нибудь собрать?
Скрипт.
Напишем свой скрипт `builder.sh`, и запустим его при сборке деривации: `bash builder.sh`.

Мы не можем использовать шебанг[^1] в `builder.sh`, поскольку во время написания мы не знаем путь к `bash` в хранилище Nix.
Не смотря на то, что `bash` конечно же есть в хранилище, ведь там есть всё!

Мы даже не можем сослаться на `/usr/bin/env`, потому что тогда мы потеряем такое крутое свойство Nix, как отсутствие состояния.
Это не говоря о том, что `PATH` очищается во время сборки, так что `bash` всё равно не будет найден.

В итоге, чтобы собрать деривацию с помощью `bash`, мы должны вызывать его, передав аргумент `builder.sh`.
Оказыватся, функция `derivation` принимает опциональный атрибут `args`, который используется для передачи аргументов исполняемому файлу сборки.

Для начала, запишем наш `builder.sh` в текущий каталог:

```text
declare -xp
echo foo > $out
```

Команда `declare -xp` выводит список эскпортированных переменных (`declare` — встроенная функция `bash`).
В предыдущей пилюле мы выяснили, что Nix вычисляет выходной путь деривации на основании её атрибутов.
Получившийся файл `.drv` содержит список переменных окружения, передаваемых в скрипт сборки.
И одна из этих переменных — `$out`.

Что мы должны сделать, так это создать что-то по пути `$out` — файл или каталог.
В нашем примере мы создаём файл.

Дополнительно, мы выводим переменные среды в процессе построения.
Мы не можем использовать `env` для этого, потому что `env` является частью `coreutils`, и этой зависимости у нас пока нет.
Сейчас у нас есть только `bash`.

Как и в случае с `coreutils` из предыдущей пилюли, мы без усилий получаем доступ к `bash`, благодаря нашей волшебной "груде всего", которая называется `nixpkgs`.

```text
nix-repl> :l <nixpkgs>
Added 3950 variables.
nix-repl> "${bash}"
"/nix/store/ihmkc7z2wqk3bbipfnlh0yjrlfkkgnv6-bash-4.2-p45"
```

Так что с помощью небольшого трюка, мы можем сослаться на `bin/bash` и создать нашу деривацию:

```text
nix-repl> d = derivation { name = "foo"; builder = "${bash}/bin/bash"; args = [ ./builder.sh ]; system = builtins.currentSystem; }
nix-repl> :b d
[1 built, 0.0 MiB DL]

this derivation produced the following outputs:
  out -> /nix/store/gczb4qrag22harvv693wwnflqy7lx5pb-foo
```

У нас получилось!
Содержимым файла `/nix/store/w024zci0x1hh1wj6gjq0jagkc1sgrf5r-foo` является как раз строка "foo".
Мы собрали нашу первую деривацию.

Обратите внимание, что мы использовали `./builder.sh`, а не `"./builder.sh"`.
Благодаря этому Nix понимает, что речь идёт о пути, и может выполнить кое-какую магию, которую мы обсудим позже.
Попробуйте строковую версию и вы увидите, что Nix не сможет найти `builder.sh`.
Это потому, что он пытается найти скрипт по относительному пути от временного каталога, где идёт сборка.

## Окружение скрипта сборки

Мы можем использовать `nix-store --read-log`, чтобы посмотреть, какие логи записал наш скрипт:

```text
$ nix-store --read-log /nix/store/gczb4qrag22harvv693wwnflqy7lx5pb-foo
declare -x HOME="/homeless-shelter"
declare -x NIX_BUILD_CORES="4"
declare -x NIX_BUILD_TOP="/tmp/nix-build-foo.drv-0"
declare -x NIX_LOG_FD="2"
declare -x NIX_STORE="/nix/store"
declare -x OLDPWD
declare -x PATH="/path-not-set"
declare -x PWD="/tmp/nix-build-foo.drv-0"
declare -x SHLVL="1"
declare -x TEMP="/tmp/nix-build-foo.drv-0"
declare -x TEMPDIR="/tmp/nix-build-foo.drv-0"
declare -x TMP="/tmp/nix-build-foo.drv-0"
declare -x TMPDIR="/tmp/nix-build-foo.drv-0"
declare -x builder="/nix/store/q1g0rl8zfmz7r371fp5p42p4acmv297d-bash-4.4-p19/bin/bash"
declare -x name="foo"
declare -x out="/nix/store/gczb4qrag22harvv693wwnflqy7lx5pb-foo"
declare -x system="x86_64-linux"
```

Давайте исследуем эти переменные среды, напечатаннные в процессе сборки.

- `$HOME` — это не ваш домашний каталог, и `/homeless-shelter` вообще не существует.
  Мы заставляем пакеты не зависеть от `$HOME` в процессе построения.
- `$PATH` также, как и `$HOME` не содержит реальных путей.
- `$NIX_BUILD_CORES` и `$NIX_STORE` — это [конфигурационные опции Nix](https://nixos.org/manual/nix/stable/command-ref/conf-file.html).
- `$PWD` и `$TMP` указывают на временный каталог, который Nix создал для сборки.
- `$builder`, `$name`, `$out` и `$system` — переменные, получившие значения из файла `.drv`.

Переменная `$out` содержит путь к деривации, куда нам надо что-нибудь сохранить.
Выглядит так, будто Nix зарезервировал для нас слот в хранилище, который мы должны заполнить.

В терминах `autotools`, `$out` — то же самое, что и путь `--prefix`.
Да, не переменная `DESTDIR` из `make`, а именно `--prefix`.
Вот суть создания пакетов без состояния.
Вы не устанавливаете пакет по глобальному пути относительно `/`, вы устанавливаете его по локальному изолированному пути в слот вашего хранилища Nix.

## Содержимое файлов .drv

> We added something else to the derivation this time: the args attribute.
> Let's see how this changed the .drv compared to the previous pill:

В примере выше мы добавили в деривацию атрибут `args`.
Как это изменило файл .drv по сравнению с примером из предыдущей пилюли?

```text
$ nix derivation show /nix/store/i76pr1cz0za3i9r6xq518bqqvd2raspw-foo.drv
{
  "/nix/store/i76pr1cz0za3i9r6xq518bqqvd2raspw-foo.drv": {
    "outputs": {
      "out": {
        "path": "/nix/store/gczb4qrag22harvv693wwnflqy7lx5pb-foo"
      }
    },
    "inputSrcs": [
      "/nix/store/lb0n38r2b20r8rl1k45a7s4pj6ny22f7-builder.sh"
    ],
    "inputDrvs": {
      "/nix/store/hcgwbx42mcxr7ksnv0i1fg7kw6jvxshb-bash-4.4-p19.drv": [
        "out"
      ]
    },
    "platform": "x86_64-linux",
    "builder": "/nix/store/q1g0rl8zfmz7r371fp5p42p4acmv297d-bash-4.4-p19/bin/bash",
    "args": [
      "/nix/store/lb0n38r2b20r8rl1k45a7s4pj6ny22f7-builder.sh"
    ],
    "env": {
      "builder": "/nix/store/q1g0rl8zfmz7r371fp5p42p4acmv297d-bash-4.4-p19/bin/bash",
      "name": "foo",
      "out": "/nix/store/gczb4qrag22harvv693wwnflqy7lx5pb-foo",
      "system": "x86_64-linux"
    }
  }
}
```

Похоже на старый добрый файл `.drv`, за исключением списка аргументов, который передаётся в `bash`.
Это спосок на самом деле содержит один скрипт `builder.sh`, который каким-то образом оказался в хранилище Nix.
Дело в том, что Nix автоматически копирует в хранилище файлы и каталоги, нужные для сборки.
Это гарантирует, что они не изменятся в процессе сборки, и что при развёртывании не может быть никаких состояний или зависимостей от текущей машины.

Поскольку `builder.sh` — обычный файл, с ним не могут быть ассцииорованы никакие файлы `.drv`.
Путь в хранилище вычисляется на основании имени файла и хэша его содержимого.
Путь в хранилище подробно обсуждаются в [одной из следующих пилюль](18-nix-store-paths.md).

## Создаём пакет из простой программы на C

Напишем простую программу на C, поместив её в файл `simple.c`:

```text
void main() {
  puts("Simple!");
}
```

А вот скрипт сборки `simple_builder.sh`:

```text
export PATH="$coreutils/bin:$gcc/bin"
mkdir $out
gcc -o $out/simple $src
```

Не бескойтесь о том, откуда возьмутся эти переменные; давайте пока опишем деривацию и соберём её:

```text
nix-repl> :l <nixpkgs>
nix-repl> simple = derivation { name = "simple"; builder = "${bash}/bin/bash"; args = [ ./simple_builder.sh ]; gcc = gcc; coreutils = coreutils; src = ./simple.c; system = builtins.currentSystem; }
nix-repl> :b simple
this derivation produced the following outputs:

  out -> /nix/store/ni66p4jfqksbmsl616llx3fbs1d232d4-simple
```

Теперь можно запустить `/nix/store/ni66p4jfqksbmsl616llx3fbs1d232d4-simple/simple` в вашем `bash`.

## Объяснение

Мы добавили два новых атрибута к вызову `derivation`: `gcc` и `coreutils`.
В `gcc = gcc;`, слева находится имя в наборе деривации, а справа — ссылка на деривацию `gcc` из `nixpkgs`.
То же касается и `coreutils`.

Мы также добавили атрибут `src`.
Здесь ничего магического — атрибуту присвоен путь `./simple.c`.
Так же, как и `simple-builder.sh`, `simple.c` будет добавлен в хранилище.

Трюк: каждый атрибут в наборе, переданный в `derivation` будет сконвертирован в строку и передан в скрипт сборки как переменная окружения.
Именно так скрипт получает доступ к `coreutils` и `gcc`: при конвертации в строку, деривации превращаются в их выходные пути, а добавление к ним `/bin` ведёт нас к их исполняемым файлам.

То же касается и переменной `src`.
`$src` — это путь к `simple.c` в хранилище Nix.
В качестве упражнение выведите `.drv` в читаемом виде.
Вы увидите среди входных дериваций `simple_builder.sh` и `simple.c`, наряду с файлами `.drv`, относящимися к `bash`, `gcc` и `coreutils`.
Кроме того, вы увидите новые добавленные переменные окружения, описанные выше.

В `simple_builder.sh` мы установили `PATH` для исполняемых файлов `gcc` и `coreutils`, так что скрипт сборки может найти нужные утилиты вроде `mkdir` и `gcc`.

Затем мы создали `$out` как каталог и разместили бинарные файлы внутри него.
Обратите внимание, что `gcc` найден через переменную окружения `PATH`, но на него точно также можно было бы сослаться явно, используя `$gcc/bin/gcc`.

## Избавляемся от `nix repl`

Попробуем повторить те же шаги, избавившись от `nix repl`, и написав выражение Nix в файле `simple.nix`:

```text
let
  pkgs = import <nixpkgs> { };
in
pkgs.stdenv.mkDerivation {
  name = "simple";
  builder = "${pkgs.bash}/bin/bash";
  args = [ ./simple_builder.sh ];
  gcc = pkgs.gcc;
  coreutils = pkgs.coreutils;
  src = ./simple.c;
  system = builtins.currentSystem;
}
```

Собрать деривацию можно с помощью команды `nix-build simple.nix`.
Она создаст в текущем каталоге символическую ссылку `result`, указывающую на выходной путь деривации.

`nix-buils` выполняет две задачи:

- [ nix-instantiate ](https://nixos.org/manual/nix/stable/command-ref/nix-instantiate.html):
  разбирает и выполняет `simple.nix` и возвращает файл `.drv`, относящийся к разобранному набору деривации
- [ `nix-store -r` ](https://nixos.org/manual/nix/stable/command-ref/nix-store.html#operation---realise):
  исполняет файл .drv, что в действительности строит деривацию.

В конце концов он создаёт символическую ссылку.

Во второй строке `simple.nix` у нас есть вызов функкции `import`.
Вспомните, что `import` принимает один аргумент — файл `.nix` для загрузки.
Содержимое файла выполняется, как будто это функция.

Мы вызываем эту функцию с пустым набором.
Подобный вызов мы видили в [пятой пилюле](05-functions-and-imports.md).
Ещё раз обратите внимание: конструкция `import <nixpkgs> { }` вызывает две функции, не одну.
Чтобы стало яснее, прочитайте выражение как `(import <nixpkgs>) { }`.

Значение, возвращаемое функцией `nixpkgs` это набор; более точно, это набор дериваций.
Вызов `import<nixpkgs> { }` в выражении `let` создаёт локальную переменную `pkgs` и вводит её в область видимости.
Тот же эффект воникает и при выполнении инструкции `:l <nixpkgs>`, который мы использовали в `nix repl`.
Набор `pkgs`, позволяет нам легко обращаться к таким деривациям, как `bash`, `gcc` и `coreutils`.
На эти деривации надо ссылаться, как на атрибуты набора `pkgs`, т.е. писать `pkgs.bach` вместо `bash`.

Ниже представлена исправленная версия `simple.nix`, использующая ключевое слово `inherit`:

```text
let
  pkgs = import <nixpkgs> { };
in
pkgs.stdenv.mkDerivation {
  name = "simple";
  builder = "${pkgs.bash}/bin/bash";
  args = [ ./simple_builder.sh ];
  inherit (pkgs) gcc coreutils;
  src = ./simple.c;
  system = builtins.currentSystem;
}
```

Для чего используется [ключевое слово `inherit`](https://nixos.org/manual/nix/stable/expressions/language-constructs.html#inheriting-attributes)?
`inherit foo;` является эквивалентом `foo = foo;`.
Точно также `inherit gcc coretutils;` — эквивалент для `gcc = gcc; coreutils = coreutils;`.
Наконец, `inherit (pkgs) gcc coreutils;` — эквивалент для `gcc = pkgs.gcc; coreutils = pkgs.coreutils;`.

Этот синтаксис имеет смысл только внутри наборов.
Здесь нет никакой магии, это просто удобный способ избежать повторения одного и того же имени и для атрибута, и для значения в области видимости.

## В следующей пилюле

Мы напишем универсальный скрипт сборки.
Наверное вы заметили, что мы в этом посте написали два разных скрипта `builder.sh`.
Было бы лучше, если бы у нас был универсальный скрипт, особенно, учитывая, что каждый скрипт сохраняется в хранилище Nix, а это затратно.

> *Is it really that hard to package stuff in Nix? No*, here we're studying the fundamentals of Nix.

*Это и правда так трудно — делать пакеты в Nix? Нет*, мы просто изучаем основы.

[^1]: *Шебанг* ровно так и записан в русской википедии.
      *Shebang* или *hash-bang*, как в оригинале у Люка — символы "#!", которые идут в начале любого скрипта и указывают пусть к интепретатору.
