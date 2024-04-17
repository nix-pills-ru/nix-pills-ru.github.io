> # Generic Builders

# Универсальные скрипты сборки

> Welcome to the 8th Nix pill.
> In the previous [7th pill](07-working-derivation.md) we successfully built a derivation.
> We wrote a builder script that compiled a C file and installed the binary under the nix store.

Добро пожаловать на восьмую пилюлю Nix.
В предыдущей [седьмой пилюле](07-working-derivation.md) мы успешно построили порождение.
Мы написали скрипт сборки, который скомпилировал программу на языке C и установил бинарный образ в хранилище Nix.

> In this post, we will generalize the builder script, write a Nix expression for [GNU hello world](https://www.gnu.org/software/hello/) and create a wrapper around the derivation built-in function.

В этом посте мы обобщим скрипт построения, напишем выражение Nix для [GNU hello world](https://www.gnu.org/software/hello/) и создадим обёртку над встроенной функцией порождения.

> ## Packaging GNU hello world

## Упаковываем GNU hello world

> In the previous pill we packaged a simple .c file, which was being compiled with a raw gcc call.
> That's not a good example of a project.
> Many use autotools, and since we're going to generalize our builder, it would be better to do it with the most used build system.

В предыдущей пилюле мы упаковали простой файл .c, который был скомпилирован с помощью обычного вызова gcc.
Это не очень хороший пример проекта.
Многие используют autotools и, поскольку мы хотим обобщить наш построить, было бы лучше сделать это с самой используемой системой сборки.

> [GNU hello world](https://www.gnu.org/software/hello/), despite its name, is a simple yet complete project which uses autotools.
> Fetch the latest tarball here: <https://ftp.gnu.org/gnu/hello/hello-2.12.1.tar.gz>.

[GNU hello world](https://www.gnu.org/software/hello/) не смотря на своё название, это всё ещё простой, но полный проект, который использует autotools.
Загрузите последний архив отсюда: <https://ftp.gnu.org/gnu/hello/hello-2.12.1.tar.gz>.

> Let's create a builder script for GNU hello world, hello_builder.sh:

Давайте создадим скрипт сборки для GNU hello world, hello_builder.sh:

```bash
export PATH="$gnutar/bin:$gcc/bin:$gnumake/bin:$coreutils/bin:$gawk/bin:$gzip/bin:$gnugrep/bin:$gnused/bin:$bintools/bin"
tar -xzf $src
cd hello-2.12.1
./configure --prefix=$out
make
make install
```

> And the derivation hello.nix:

И порождение hello.nix:

```nix
let
  pkgs = import <nixpkgs> { };
in
derivation {
  name = "hello";
  builder = "${pkgs.bash}/bin/bash";
  args = [ ./hello_builder.sh ];
  inherit (pkgs)
    gnutar
    gzip
    gnumake
    gcc
    coreutils
    gawk
    gnused
    gnugrep
    ;
  bintools = pkgs.binutils.bintools;
  src = ./hello-2.12.1.tar.gz;
  system = builtins.currentSystem;
}
```

> > ### Nix on darwin
> > Darwin (i.e. macOS) builds typically use `clang` rather than `gcc` for a C compiler.
> > We can adapt this early example for darwin by using this modified version of `hello.nix`:
> ### Nix в Darwin 
> Сборка в Darwin (т.е. macOS) традиционно использует `clang` вместо `gcc` в качестве компилятора C.
> Мы можем адаптировать этот ранний пример для Darwin, используя такую модифицированную версию `hello.nix`:
> ```nix
> let
>   pkgs = import <nixpkgs> { };
> in
> derivation {
>   name = "hello";
>   builder = "${pkgs.bash}/bin/bash";
>   args = [ ./hello_builder.sh ];
>   inherit (pkgs)
>     gnutar
>     gzip
>     gnumake
>     coreutils
>     gawk
>     gnused
>     gnugrep
>     ;
>   gcc = pkgs.clang;
>   bintools = pkgs.clang.bintools.bintools_bin;
>   src = ./hello-2.12.1.tar.gz;
>   system = builtins.currentSystem;
> }
> ```
> 
> > Later, we will show how Nix can automatically handle these differences.
> > For now, please be just aware that changes similar to the above may be needed in what follows.
>
> Позже мы покажем как Nix может автоматически обрабатывать эти различия.
> Сейчас просто имейте в виде, что в других скриптах также могут потребоваться изменения, подобные описанным выше.

> Now build it with `nix-build hello.nix` and you can launch `result/bin/hello`.
> Nothing easier, but do we have to create a builder.sh for each package?
> Do we always have to pass the dependencies to the `derivation` function?

Теперь соберём программу, запустив `nix build hello.nix` и вы можете запустить `result/bin/hello`.
Нет ничего проще, но надо ли нам писать builder.sh для каждого пакета?
Должны ли мы всегда передавать зависимости в функцию `derivation`?

> Please note the `--prefix=$out` we were talking about in the [previous pill](#07-working-derivation.md).

Пожалуйста, обратите внимание на `--prefix=$out`, который мы обсуждали в [предыдущей пилюле](07-working-derivation.md).

> ## A generic builder

## Обобщённый скрипт

> Let's create a generic `builder.sh` for autotools projects:

Давайте напишем обобщённую версию `builder.sh` для проектов autotools:

```bash
set -e
unset PATH
for p in $buildInputs; do
    export PATH=$p/bin${PATH:+:}$PATH
done

tar -xf $src

for d in *; do
    if [ -d "$d" ]; then
        cd "$d"
        break
    fi
done

./configure --prefix=$out
make
make install
```

> What do we do here?

Что мы здесь делаем?

> 1. Exit the build on any error with `set -e`.
> 2. First `unset PATH`, because it's initially set to a non-existent path.
> 3. We'll see this below in detail, however for each path in `$buildInputs`, we append `bin` to `PATH`.
> 4. Unpack the source.
> 5. Find a directory where the source has been unpacked and `cd` into it.
> 6. Once we're set up, compile and install.

1. С помощью `set -e` мы говорим оболочке прерывать выполнение скрипта в случае любой ошибки.
2. Первым делом очищаем `PATH`` (`unset PATH`), потому что в начале переменная содержит несуществующие пути.
3. Для каждого пути в `$buildInputs` добавляем к нему `bin` и всё вместе добавляем к `PATH`.
   Подробности обсуждим чуть позже.
4. Распоковываем исходники.
5. Ищем каталог, куда были распакованы исходники и переходим в него, выполнив команду `cd`.
6. Наконец, конфигурируем, компилируем и устанавливаем проект.

> As you can see, there's no reference to "hello" in the builder anymore.
> It still makes several assumptions, but it's certainly more generic.

Как вы видите, в скрипте сборки больше нет никаких ссылок на "hello".
Он, по прежнему, опирается на несколько соглашений, но безусловно, это более универсальная версия.

> Now let's rewrite `hello.nix`:

Теперь давайте перепишем `hello.nix`:

```nix
let
  pkgs = import <nixpkgs> { };
in
derivation {
  name = "hello";
  builder = "${pkgs.bash}/bin/bash";
  args = [ ./builder.sh ];
  buildInputs = with pkgs; [
    gnutar
    gzip
    gnumake
    gcc
    coreutils
    gawk
    gnused
    gnugrep
    binutils.bintools
  ];
  src = ./hello-2.12.1.tar.gz;
  system = builtins.currentSystem;
}```

> All clear, except that buildInputs.
> However it's easier than any black magic you are thinking of at this moment.

Тут всё ясно, за исключением `buildInputs`.
Тем не менее, это проще любой чёрной магии, о которой вы думаете в данный момент.

> Nix is able to convert a list to a string.
> It first converts the elements to strings, and then concatenates them separated by a space:

Nix умеет конвертировать список в строку.
Сначала он конвертирует элементы в строки, а затем соединяет их, разделяя пробелом:

```text
nix-repl> builtins.toString 123
"123"

nix-repl> builtins.toString [ 123 456 ]
"123 456"
```

> Recall that derivations can be converted to a string, hence:

Вспомним, что порождения могут быть сконвертированы в строку, поэтому:

```text
nix-repl> :l <nixpkgs>
Added 3950 variables.

nix-repl> builtins.toString gnugrep
"/nix/store/g5gdylclfh6d224kqh9sja290pk186xd-gnugrep-2.14"

nix-repl> builtins.toString [ gnugrep gnused ]
"/nix/store/g5gdylclfh6d224kqh9sja290pk186xd-gnugrep-2.14 /nix/store/krgdc4sknzpw8iyk9p20lhqfd52kjmg0-gnused-4.2.2"
```

> Simple!
> The buildInputs variable is a string with out paths separated by space, perfect for bash usage in a for loop.

Всё просто!
Переменная `buildInputs` это строка с нашими путями, разделёнными пробелом, идеально для использования в bash в цикле for.

> ## A more convenient derivation function

## Более удобная функция derivation

> We managed to write a builder that can be used for multiple autotools projects.
> But in the hello.nix expression we are specifying tools that are common to more projects; we don't want to pass them every time.

Нам удалось написать сборщик, который может быть использован для множества проектов autotools.
Но в выражении hello.nix мы определяем все программы, которые могут потребоваться, даже те, которые не нужны для сборки конкретного проекта.

> A natural approach would be to create a function that accepts an attribute set, similar to the one used by the derivation function, and merge it with another attribute set containing values common to many projects.

Естественным подходом было бы создать функцию, которая принимала бы набор атрибутов, подобно тому, как это делает функция
derivation, и сливать его с другим набором атрибуотов, содержащим значения, общие для многих проектов.

> Create `autotools.nix`:

Создадим `autotools.nix`:

```nix
pkgs: attrs:
let
  defaultAttrs = {
    builder = "${pkgs.bash}/bin/bash";
    args = [ ./builder.sh ];
    baseInputs = with pkgs; [
      gnutar
      gzip
      gnumake
      gcc
      coreutils
      gawk
      gnused
      gnugrep
      binutils.bintools
    ];
    buildInputs = [ ];
    system = builtins.currentSystem;
  };
in
derivation (defaultAttrs // attrs)
```

> Ok now we have to remember a little about [Nix functions](#functions-and-imports).
> The whole nix expression of this `autotools.nix` file will evaluate to a function.
> This function accepts a parameter `pkgs`, then returns a function which accepts a parameter `attrs`.

Итак, сейчас нам надо кое-что вспомнить о [фукнциях Nix](05-functions-and-imports.md).
Всё выржаение Nix из файла `autotools.nix` превратится в функцию.
Эта функция принимает параметр `pkgs` и возвращает функцию, которая принимает параметр `attrs`.

> The body of the function is simple, yet at first sight it might be hard to grasp:

Тело функции простое, но при первом знакомстве её может быть трудно понять:

> 1. First drop in the scope the magic `pkgs` attribute set.
> 2. Within a let expression we define a helper variable, `defaultAttrs`, which serves as a set of common attributes used in derivations.
> 3. Finally we create the derivation with that strange expression, (`defaultAttrs // attrs`).

1. Сначала добавляем в обласить видимость магический набор атрибутов `pkgs`.
2. С помощью выражения `let` определяем вспомогательную переменную `defaultAttrs`, куда складываем несколько атрибутов, нужных для порождения.
3. В конце создаём вызываем `derivation`, передавая в качестве параметра странное выражение (`defaultAttrs // attrs`).

> The [// operator](https://nixos.org/manual/nix/stable/expressions/language-operators.html) is an operator between two sets.
> The result is the union of the two sets.
> In case of conflicts between attribute names, the value on the right set is preferred.

[Оператор //](https://nixos.org/manual/nix/stable/expressions/language-operators.html) --- это оператор над двумя наборами.
Результатом является объединение двух наборов.
В случае конфликта имён атрибутов, используется значение из правого набора.

> So we use `defaultAttrs` as base set, and add (or override) the attributes from `attrs`.

Так что мы используем `defaultAttrs` как базовый набор, и добавляем в него (или переопределяем в нём) атрибуты из `attrs`.

> A couple of examples ought to be enough to clear out the behavior of the operator:

Пара примеров прояснит поведение оператора: 

```text
nix-repl> { a = "b"; } // { c = "d"; }
{ a = "b"; c = "d"; }

nix-repl> { a = "b"; } // { a = "c"; }
{ a = "c"; }
```

> **Exercise:** Complete the new `builder.sh` by adding `$baseInputs` in the `for` loop together with `$buildInputs`.

**Упражнение:** Завершите новый скрипт `builder.sh` добавив `$baseInputs` в цикл `for` вместе с `$buildInputs`.

> As you noticed, we passed that new variable in the derivation.
> Instead of merging buildInputs with the base ones, we prefer to preserve buildInputs as seen by the caller, so we keep them separated.
> Just a matter of choice.

Вы могли заметить, что мы передаём эту новую переменную в функцию derivation.
Вместо слияния `buildInputs` с базовыми значениями, мы предпочитаем сохранить `buildInputs` в том виде, в котором их передала вызывающая сторона, так что мы оставляем их разделёнными.
Всего лишь вопрос выбора.

> Then we rewrite `hello.nix` as follows:

Теперь перепишем `hello.nix` как ниже:


```nix
let
  pkgs = import <nixpkgs> { };
  mkDerivation = import ./autotools.nix pkgs;
in
mkDerivation {
  name = "hello";
  src = ./hello-2.12.1.tar.gz;
}
```

> Finally!
> We got a very simple description of a package!
> Below are a couple of remarks that you may find useful as you're continuing to understand the nix language:

Финал!
Мы получили очень простое описание пакета!
Ниже пара комментариев, которые вы можете счесть полезными, пока продолжаете разбираться с языком Nix.

> - We assigned to pkgs the import that we did in the previous expressions in the "with".
>   Don't be afraid, it's that straightforward.
> - The mkDerivation variable is a nice example of partial application, look at it as (`import ./autotools.nix`) `pkgs`.
>   First we import the expression, then we apply the `pkgs` parameter.
>   That will give us a function that accepts the attribute set `attrs`.
> - We create the derivation specifying only name and src.
>   If the project eventually needed other dependencies to be in PATH, then we would simply add those to buildInputs (not specified in hello.nix because empty).

- Мы помещаем в переменную `pkgs` импорт, который в предыдущих выражениях помещали в оператор "with".
  Не бойтесь, это просто.
- Переменная `mkDerivation` это прекрасный пример частичного применения, посмотрите на неё как на '(import ./autotools.nix) pkgs'.
- Вначале мы импортируем выражение, затем применяем его к параметру `pkgs`. <!-- вызов функции в функциональных языках часто называют применением -->
- Это даст нам функцию, которая принимает набор атрибутов `attrs`.
- Мы создаём порождение, указывая только атрибуты name и src.
  Если проект в конечном итоге нуждаетя в других знависимостях в PATH, тогда
  мы можем просто добавить их в `buildInputs` (не указан в hello.nix потому что пустой).

> Note we didn't use any other library.
> Special C flags may be needed to find include files of other libraries at compile time, and ld flags at link time.

Обратие внимание, мы не используюем никаких других библиотек.
Специальный флаги комплиятора C могут потребоваться, чтобы искать включаемые файлы других библиотек на этапе компиляции, также, как и флаги компоновщика на этапе компоновки.

> ## Conclusion

## Заключение

> Nix gives us the bare metal tools for creating derivations, setting up a build environment and storing the result in the nix store.

Nix даёт нам базовые инструменты для создания порождений, подготовки окружения для сборки и сохранения результата в хранилище Nix.

> Out of this pill we managed to create a generic builder for autotools projects, and a function `mkDerivation` that composes by default the common components used in autotools projects instead of repeating them in all the packages we would write.

В этой пилюле нам удалсь создать универсальный скрипт сборки для проектов autotools, и функцию `mkDerivation`, которая собирает основные компоненты используемые в проектах autotools с настройками по умолчанию, вместо повторения их во всех пакетах, которые мы могли бы написать.

> We are familiarizing ourselves with the way a Nix system grows up: it's about creating and composing derivations with the Nix language.

Мы знакомим себя со способом, по которому система Nix расширяется: это делается путём создания и композиции порождений с помощью языка Nix.

> *Analogy*: in C you create objects in the heap, and then you compose them inside new objects.
> Pointers are used to refer to other objects.

*Аналогия*: в C вы создаёте объекты в куче, и затем вы объединяете их внутри новых объектов.
Для ссылки на другие объекты используются указатели.

> In Nix you create derivations stored in the nix store, and then you compose them by creating new derivations. Store paths are used to refer to other derivations.

В Nix вы создаёте порождении, хранимые в хранилище Nix, и затем вы объединяете их, создавая новые порождения. Пути в хранилище используются для ссылки на другие порождения.

> ## Next pill

## В следующей пилюле

> ...we will talk a little about runtime dependencies.
> Is the GNU hello world package self-contained?
> What are its runtime dependencies?
> We only specified build dependencies by means of using other derivations in the "hello" derivation.

Мы немного поговорим про зависимости от среды выполнения.
Является ли пакет GNU hello world автономным?
Каковы зависимости его среды выполнения?
Пока что мы всего лишь определили зависимости для сборки, посредством использования других порождений в порождении "hello".