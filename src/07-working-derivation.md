> # Working Derivation

# Работающее порождение

> ## Introduction

## Введение

> Welcome to the seventh nix pill. In the previous [sixth pill](#our-first-derivation) we introduced the notion of derivation in the Nix language — how to define a raw derivation and how to (try to) build it.

Добро пожаловать на седьмую пилюлю Nix. В предыдущей [шестой пилюле](06-our-first-derivation.md) вы начали разбираться с порождениями в языке Nix, выяснили, как описать пустое порождение и к (попытаться) его построить. 

> In this post we continue along the path, by creating a derivation that actually builds something.
> Then, we try to package a real program: we compile a simple C file and create a derivation out of it, given a blessed toolchain.

В этом посте мы продолжим двигаться по тому же пути, создав порождение, которое действительно что-то строит.
Затем мы попытаемся запаковать реальную программы: скомпилируем простой файл .C и созданим порождение с ним, используя благословенный набор инструментов.

> I remind you how to enter the Nix environment: `source ~/.nix-profile/etc/profile.d/nix.sh`

Напоминаю, как запускать окружение Nix: `source ~/.nix-profile/etc/profile.d/nix.sh`

> ## Using a script as a builder

## Использование скрипта в качестве построителя

> What's the easiest way to run a sequence of commands for building something?
> A bash script.
> We write a custom bash script, and we want it to be our builder.
> Given a `builder.sh`, we want the derivation to run `bash builder.sh`.

Каков наипростейший путь выполнить цепочку команд, чтобы что-нибудь построить?
Скрипт.
Мы пишем собственный скрипт, и хотим, чтобы он был нашим построителем.
Имея `builder.sh`, мы хотим, чтобы порождение запускало `bash builder.sh`.

> We don't use hash bangs in `builder.sh`, because at the time we are writing it we do not know the path to bash in the nix store.
> Yes, even bash is in the nix store, everything is there.

Мы не используем шебанг[^1] в `builder.sh`, поскольку во время написания мы не знаем путь к `bash` в хранилище Nix.
Да, не смотря на то, что `bash` есть в хранилище, ведь там есть всё.

> We don't even use /usr/bin/env, because then we lose the cool stateless property of Nix.
> Not to mention that `PATH` gets cleared when building, so it wouldn't find bash anyway.

Мы даже не ссылаемся на /usr/bin/env, потому что тогда мы такое крутое свойство Nix, как отсутствие состояния.
Не говоря уже о том, что `PATH` очищается во время построения, так что `bash` всё равно не будет найден.

> In summary, we want the builder to be bash, and pass it an argument, `builder.sh`.
> Turns out the `derivation` function accepts an optional `args` attribute which is used to pass arguments to the builder executable.

В итоге, мы хотим, чтобы bash был построителем, и передаём ему аргумент `builder.sh`.
Оказыватся, функция `derivation` принимает опционаьный атрибут `args`, который используется для передачи аргументов исполняемому файлу построителя.

> First of all, let's write our `builder.sh` in the current directory:

Прежде всего, запишем наш `builder.sh` в текущий каталог:

```text
declare -xp
echo foo > $out
```

> The command `declare -xp` lists exported variables (`declare` is a builtin bash function).
> As we covered in the previous pill, Nix computes the output path of the derivation.
> The resulting `.drv` file contains a list of environment variables passed to the builder.
> One of these is `$out`.

Команда `declare -xp` выводит список эскпортированных переменных (`declare` это встроенная функция `bash`).
Как мы рассмотрели в предыдущей пилюле, Nix вычисляет выходной путь порождения.
Получившийся файл `.drv` содержит список переменных окружения, передаваемых построителю.

> What we have to do is create something in the path `$out`, be it a file or a directory.
> In this case we are creating a file.

Что мы должны сделать, так это создать что-то по пути `$out`, будь то файл или каталог.
В этом случае мы создаём файл.

> In addition, we print out the environment variables during the build process.
> We cannot use env for this, because env is part of coreutils and we don't have a dependency to it yet.
> We only have bash for now.

Дополнительно, мы выводим переменных окружения в процессе построения.
Мы не можем использовать `env` для этого, потому что `env` является частью `coreutils`, и у нас пока нет этой зависимости.
Сейчас у нас есть только `bash`.

> Like for coreutils in the previous pill, we get a blessed bash for free from our magic nixpkgs stuff:

Как и с `coreutils` в предыдущей пилюле, мы получаем благословенный `bash` бесплатно <!-- без усилий --> из нашей волшебной "груды всего", которая называется `nixpkgs`.

```text
nix-repl> :l <nixpkgs>
Added 3950 variables.
nix-repl> "${bash}"
"/nix/store/ihmkc7z2wqk3bbipfnlh0yjrlfkkgnv6-bash-4.2-p45"
```

> So with the usual trick, we can refer to bin/bash and create our derivation:

Так что с помощью обычного трюка, мы можем сослаться на bin/bash и создать наше порождение:

```text
nix-repl> d = derivation { name = "foo"; builder = "${bash}/bin/bash"; args = [ ./builder.sh ]; system = builtins.currentSystem; }
nix-repl> :b d
[1 built, 0.0 MiB DL]

this derivation produced the following outputs:
  out -> /nix/store/gczb4qrag22harvv693wwnflqy7lx5pb-foo
```

> We did it!
> The contents of `/nix/store/w024zci0x1hh1wj6gjq0jagkc1sgrf5r-foo` is really foo.
> We've built our first derivation.

У нас получилось!
Содержимым файла `/nix/store/w024zci0x1hh1wj6gjq0jagkc1sgrf5r-foo` является как раз строка "foo".
Мы построили наше первое порождение.

> Note that we used `./builder.sh` and not `"./builder.sh"`.
> This way, it is parsed as a path, and Nix performs some magic which we will cover later.
> Try using the string version and you will find that it cannot find `builder.sh`.
> This is because it tries to find it relative to the temporary build directory.

Обратите внимание, что мы использовали `./builder.sh`, а не `"./builder.sh"`.
Таким образом, он анализируется, как путь, и Nix выполняет кое-какую маги, которую мы обсудим позже.
Попробуйте использовать строковую версию и вы увидите, что Nix не может найти `builder.sh`.
Это потому, что он пытается найти скрипт по относительному пути от временного каталога, где идёт сборка.

> ## The builder environment

Окружение построителя

> We can use `nix-store --read-log` to see the logs our builder produced:

Мы можем использовать `nix-store --read-log`, чтобы посмотреть, какие логи произвёл наш построитель:

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

> Let's inspect those environment variables printed during the build process.

Давайте проинспектируем эти переменные окружения, напечатннные в процессе построения.

> - `$HOME` is not your home directory, and `/homeless-shelter` doesn't exist at all.
>   We force packages not to depend on `$HOME` during the build process.
> - `$PATH` plays the same game as `$HOME`
> - `$NIX_BUILD_CORES` and `$NIX_STORE` are [nix configuration options](https://nixos.org/manual/nix/stable/command-ref/conf-file.html)
> - `$PWD` and `$TMP` clearly show that nix created a temporary build directory
> - Then `$builder`, `$name`, `$out`, and `$system` are variables set due to the .drv file's contents.

- `$HOME` --- это не ваш домашний каталог, и `/homeless-shelter` вообще не существует.
  Мы заставляем пакеты не зависеть от `$HOME` в процессе построения.
- `$PATH` играет такую же роль, как и `$HOME`.
- `$NIX_BUILD_CORES` и `$NIX_STORE` --- это [конфигурационные опции Nix](https://nixos.org/manual/nix/stable/command-ref/conf-file.html).
- `$PWD` и `$TMP` ясно показываеют, что Nix создал временный каталог для построения.
- `$builder`, `$name`, `$out` и `$system` -- переменные, установленные в соответствии с содержимым файла `.drv`.

> And that's how we were able to use `$out` in our derivation and put stuff in it.
> It's like Nix reserved a slot in the nix store for us, and we must fill it.

И вот как мы могли бы использовать `$out` в нашем порождении и сохранить там что-нибудь.
Как будто Nix зарезервировал для нас слот в хранилище, и мы должны его заполнить.

> In terms of autotools, `$out` will be the `--prefix` path.
> Yes, not the make `DESTDIR`, but the `--prefix`.
> That's the essence of stateless packaging.
> You don't install the package in a global common path under `/`, you install it in a local isolated path under your nix store slot.

В терминах средства сборки `autotools`, `$out` -- то же самое, что и путь `--prefix`.
Да, создавать не `DESTDIR`, а именно `--prefix`.
Вот суть создания пакетов без состояния.
Вы не устанавливаете пакет по глобальному пути относительно `/`, вы устанавливаете его по локальному изолированному пути в слот вашего хранилища Nix.

> ## The .drv contents

## Содержимое .drv

> We added something else to the derivation this time: the args attribute.
> Let's see how this changed the .drv compared to the previous pill:

Мы добавили кое-что новое в порождение сейчас: атрибут `args`.
Давайте взглянем, как это изменило .drv по сравнению с предыдущей пилюлей.

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

> Much like the usual .drv, except that there's a list of arguments in there passed to the builder (bash) with `builder.sh`…
> In the nix store..?
> Nix automatically copies files or directories needed for the build into the store to ensure that they are not changed during the build process and that the deployment is stateless and independent of the building machine.
> `builder.sh` is not only in the arguments passed to the builder, it's also in the input derivations.

Очень похоже на обычный .drv, за исключением того, что здесь есть список аргументов, который передаётся в построитель (bash) с помощью `builder.sh`...
В хранилище Nix..?
Nix автоматически копирует файлы или каталоги, нужные для построения в хранилище, чтобы гарантировать, что они не будут изменены в процессе построения, и что развёртывание не имеет состояния и независимо от машины, на которой происходит сборка.
`builder.sh` не только в аргументах, передаваемых построителю, он также во входных порождениях.


> Given that `builder.sh` is a plain file, it has no .drv associated with it.
> The store path is computed based on the filename and on the hash of its contents.
> Store paths are covered in detail in [a later pill](#nix-store-paths).

Учитывая, что `builder.sh` --- это обычный файл, он не имеет никаких .drv ассоциированных с кним.
Путь в хранилище вычисляется на основании имени файла и хэша его содержимого.
Путь в хранилище подробно обсуждаются в [одной из следующих пилюль](18-nix-store-paths.md).

> ## Packaging a simple C program

## Создаём пакет из простой программы на C

> Start off by writing a simple C program called `simple.c`:

Начнём с написания простой программы на C, поместив её в файл `simple.c`:

```text
void main() {
  puts("Simple!");
}
```

> And its `simple_builder.sh`:

А вот её `simple_builder.sh`:

```text
export PATH="$coreutils/bin:$gcc/bin"
mkdir $out
gcc -o $out/simple $src
```

> Don't worry too much about where those variables come from yet; let's write the derivation and build it:

Пока не бескойтесь слишком сильно о том, откуда возьмутся эти переменные; давайте напишем порождение и построим его:

```text
nix-repl> :l <nixpkgs>
nix-repl> simple = derivation { name = "simple"; builder = "${bash}/bin/bash"; args = [ ./simple_builder.sh ]; gcc = gcc; coreutils = coreutils; src = ./simple.c; system = builtins.currentSystem; }
nix-repl> :b simple
this derivation produced the following outputs:

  out -> /nix/store/ni66p4jfqksbmsl616llx3fbs1d232d4-simple
```

> Now you can run `/nix/store/ni66p4jfqksbmsl616llx3fbs1d232d4-simple/simple` in your shell.

Теперь вы можете запустить `/nix/store/ni66p4jfqksbmsl616llx3fbs1d232d4-simple/simple` в своей оболочке.

> ## Explanation

## Объяснение

> We added two new attributes to the derivation call, `gcc` and `coreutils`.
> In `gcc = gcc;`, the name on the left is the name in the derivation set, and the name on the right refers to the gcc derivation from nixpkgs.
> The same applies for coreutils.

Мы добавили два новых атрибута к вызову построения, `gcc` и `coreutils`.
В `gcc = gcc;`, слева находится имя в наборе построения, а справа --- ссылка на порождение gcc из nixpkgs.
То же касается и coreutils.

> We also added the `src` attribute, nothing magical — it's just a name, to which the path `./simple.c` is assigned. 
> Like `simple-builder.sh`, `simple.c` will be added to the store.

Мы также добавили атрибут `src`, ничего магического --- это всего лишь имя, которому присвоен путь `./simple.c`.
Так же, как и `simple-builder.sh`, `simple.c` будет добавлен в хранилище.

> The trick: every attribute in the set passed to `derivation` will be converted to a string and passed to the builder as an environment variable.
> This is how the builder gains access to coreutils and gcc: when converted to strings, the derivations evaluate to their output paths, and appending `/bin` to these leads us to their binaries.

Трюк: каждый атрибут в наборе, переданный в `derivation` будет сконвертирован в строку и передан в построитель как переменная окружения.
Это то, как построитель получает доступ к coreutils и gcc: во время конвертации, порождения превращаются в их выходные пути и добавление к ним `/bin` ведёт нас к их исполняемым файлам.

> The same goes for the `src` variable. `$src` is the path to `simple.c` in the nix store.
> As an exercise, pretty print the .drv file.
> You'll see `simple_builder.sh` and `simple.c` listed in the input derivations, along with bash, gcc and coreutils .drv files.
> The newly added environment variables described above will also appear.

То же касается и переменной `src`. `$src` -- это путь к `simple.c` в хранилище Nix.
В качестве упражнение выведите .drv в читаемом виде.
Вы увидите `simple_builder.sh` и `simple.c` среди входных построений, наряду с файлами .drv для bash, gcc и coreutils.
Новые добавленные переменные окружения, описанные выше, также появятся.

> In `simple_builder.sh` we set the `PATH` for gcc and coreutils binaries, so that our build script can find the necessary utilities like mkdir and gcc.

В `simple_builder.sh` мы установили `PATH` для исполняемых файлов gcc и coreutils, так что скрипт построения может найти нужные утилиты вроде mkdir и gcc.

> We then create `$out` as a directory and place the binary inside it.
> Note that gcc is found via the `PATH` environment variable, but it could equivalently be referenced explicitly using `$gcc/bin/gcc`.

Затем мы создали `$out` как каталог и разместили бинарные файлы внутри него.
Обратите внимание, что gcc найден через переменную окружения `PATH`, но на него точно также можно было бы сослаться явно, использьуя `$gcc/bin/gcc`.

> ## Enough of `nix repl`

## Достаточно `nix repl`

> Drop out of nix repl and write a file `simple.nix`:

Избавимся от nix repl и напишем файл `simple.nix`:

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

> Now you can build it with `nix-build simple.nix`.
> This will create a symlink `result` in the current directory, pointing to the out path of the derivation.

Теперь вы можете выполнить его с помощью `nix-build simple.nix`.
Этот вызов создаст символическую ссылку `result` в текущем каталоге, указывающую на выходной путь порождения.

> nix-build does two jobs:

nix-buils выполняет две задачи:

> - [ nix-instantiate ](https://nixos.org/manual/nix/stable/command-ref/nix-instantiate.html):\\
>   parse and evaluate `simple.nix` and return the .drv file corresponding to the parsed derivation set
> - [ `nix-store -r` ](https://nixos.org/manual/nix/stable/command-ref/nix-store.html#operation---realise):
>   realise the .drv file, which actually builds it.

- [ nix-instantiate ](https://nixos.org/manual/nix/stable/command-ref/nix-instantiate.html):\\
  разбирает и выполняет `simple.nix` и возвращает файл .drv, относящийся к разобранному набору порождения
- [ `nix-store -r` ](https://nixos.org/manual/nix/stable/command-ref/nix-store.html#operation---realise):
  реализует файл .drc, что в действительности строит порождение.

> Finally, it creates the symlink.

В конце он создаёт символическую ссылку.

> In the second line of `simple.nix`, we have an `import` function call.
> Recall that `import` accepts one argument, a nix file to load.
> In this case, the contents of the file evaluate to a function.

Во второй строке `simple.nix` у нас есть вызов функкции `import`.
Вспомните, что `import` принимает один аргумент, файл .nix для загрузки.
В этом случае, содержимое файла выполняется, как будто это функция.

> Afterwards, we call the function with the empty set.
> We saw this already in [the fifth pill](#functions-and-imports).
> To reiterate: `import <nixpkgs> {}` is calling two functions, not one.
> Reading it as `(import <nixpkgs>) {}` makes this clearer.

После мы вызываем функцию с пустым набором.
Мы видили подобный вызов в [пятой пилюле](05-functions-and-imports.md).
Чтобы повторить: `import <nixpkgs> {}` вызывает две функции, не одну.
Чтение этого выражение как `(import <nixpkgs>) {}` сделаем этот момент ясным.

> The value returned by the nixpkgs function is a set; more specifically, it's a set of derivations.
> Calling `import <nixpkgs> {}` into a `let`-expression creates the local variable `pkgs` and brings it into scope.
> This has an effect similar to the `:l <nixpkgs>` we used in nix repl, in that it allows us to easily access derivations such as `bash`, `gcc`, and `coreutils`, but those derivations will have to be explicitly referred to as members of the `pkgs` set (e.g., `pkgs.bash` instead of just `bash`).

Значение, возвращаемое функцией nixpkgs это набор; более точно, это набор порождений.
Вызов `import<nixpkgs> {}` в выражении `let` создаёт локальную переменную pkgs и вводит её в область видимости.
Это имеет тот же эффект, что и `:l <nixpkgs>`, который мы использовали в nix repl, что позволяет нам легко обращаться к порождениям, таким, как `bash`, `gcc` и `coreutils`, но мы должны ссылаться на эти порождения, как на члены набора `pkgs` (т.е. `pkgs.bach` вместо `bash`).

> Below is a revised version of the `simple.nix` file, using the `inherit` keyword:

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

> Here we also take the opportunity to introduce the [`inherit` keyword](https://nixos.org/manual/nix/stable/expressions/language-constructs.html#inheriting-attributes).
> `inherit foo;` is equivalent to `foo = foo;`.
> Similarly, `inherit gcc coreutils;` is equivalent to ` gcc = gcc; coreutils = coreutils;`.
> Lastly, `inherit (pkgs) gcc coreutils;` is equivalent to ` gcc = pkgs.gcc; coreutils = pkgs.coreutils;`.

Здесь у нас таже есть возможность ввести [ключевое слово `inherit`](https://nixos.org/manual/nix/stable/expressions/language-constructs.html#inheriting-attributes).
`inherit foo;` является эквивалентом `foo = foo;`.
Похожим образом, `inherit gcc coretutils;` --- это эквивалент для `gcc = gcc; coreutils = coreutils;`.
Наконец, `inherit (pkgs) gcc coreutils;` --- эквивалент для `gcc = pkgs.gcc; coreutils = pkgs.coreutils;`.

> This syntax only makes sense inside sets.
> There's no magic involved, it's simply a convenience to avoid repeating the same name for both the attribute name and the value in scope.

Этот синтаксис имеет смысл только внутри наборов.
Здесь нет никакой магии, это просто удобный способ избежать повторения одного и того же имени и для имени атрибута и для значения в области видимости.

> ## Next pill

## В следующей пилюле

> We will generalize the builder.
> You may have noticed that we wrote two separate `builder.sh` scripts in this post.
> We would like to have a generic builder script instead, especially since each build script goes in the nix store: a bit of a waste.

Мы обобщим построители.
Вы должно быть заметили, что мы написали два разных скрипта `builder.sh` в этом посте.
Было бы лучше, если бы у нас был общий скрипт-построитель, особенно, учитывая, что каждый скрипт сохраняется в хранилище Nix: это затратно.

> *Is it really that hard to package stuff in Nix? No*, here we're studying the fundamentals of Nix.

*Это действительно так трудно делать пакеты в Nix? Нет*, мы просто изучаем основы Nix.

[^1]: *Шебанг* ровно так и записан в русской википедии.
      *Shebang* или *hash-bang*, как в оригинале у Люка --- символы "#!", которые идут в начале любого скрипта и указывают пусть к интепретатору.