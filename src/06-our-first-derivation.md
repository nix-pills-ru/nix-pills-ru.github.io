> # Our First Derivation

# Наше первое порождение

> Welcome to the sixth Nix pill.
> In the previous [fifth pill](#functions-and-imports) we introduced functions and imports.
> Functions and imports are very simple concepts that allow for building complex abstractions and composition of modules to build a flexible Nix system.

Добро пожалвать в шестую пилюлю.
В предыдущей [пятой пилюле](05-functions-and-imports.md) мы познакомились с функциями и импортом.
Функции и импорт — очень простые концепции, которые позволяют строить сложные абстракции и композицию модулей, чтобы строить гибкую систему Nix.

> In this post we finally arrived to writing a derivation.
> Derivations are the building blocks of a Nix system, from a file system view point.
> The Nix language is used to describe such derivations.

В этом посте мы наконец добрались до написания порождения.
Порождения это строительные блоки систему Nix, с точки зрения файловой системы.
Язык Nix используется для описания таких порождений.

> I remind you how to enter the Nix environment: `source ~/.nix-profile/etc/profile.d/nix.sh`

Напоминаю, как входить в окружение Nix: `source ~/.nix-profile/etc/profile.d/nix.sh`

> ## The derivation function

## Функция порождения

> The [derivation built-in function](https://nixos.org/manual/nix/stable/expressions/derivations.html) is used to create derivations.
> I invite you to read the link in the Nix manual about the derivation built-in.
> A derivation from a Nix language view point is simply a set, with some attributes.
> Therefore you can pass the derivation around with variables like anything else.

[Встроенная функция derivation](https://nixos.org/manual/nix/stable/expressions/derivations.html) используется для создания порождений.
Я приглашаю вас прочитать сслыку в руководстве по Nix про встроенную <!-- функцию --> derivation.
Порождение с точки зрения языка Nix — это всего лишь набор с несколькими атрибутами.
Так что вы можете передавать порождение с помощью переменных, как и всё остальное <!-- как и в других случаях в языке Nix -->.

> That's where the real power comes in.

Вот где приходит реальная мощь.

> The `derivation` function receives a set as its first argument.
> This set requires at least the following three attributes:

Функция `derivation` принимает набор в качестве первого агрумента.
Этот набор требует по меньшей мере следующих трёх атрибутов:

> - name: the name of the derivation.
>   In the nix store the format is hash-name, that's the name.
> - system: is the name of the system in which the derivation can be built.
>   For example, x86_64-linux.
> - builder: is the binary program that builds the derivation.

- name: название порождения.
  В хранилище Nix порождения хранятся в формате хэш-название, и это то самое название.
- system: название системы, в которой порождение может быть собрано.
  Например, x86_64-linux.
- builder: двоичная программа, которая собирает порождение.

> First of all, what's the name of our system as seen by nix?

Прежде всего, <!-- давайте выясним -->, какое название у нашей системы с точки зрения Nix?

```text
nix-repl> builtins.currentSystem
"x86_64-linux"
```

> Let's try to fake the name of the system:

Давайте попробуем использовать какое-нибудь несуществующее название системы:

```text
nix-repl> d = derivation { name = "myname"; builder = "mybuilder"; system = "mysystem"; }
nix-repl> d
«derivation /nix/store/z3hhlxbckx4g3n9sw91nnvlkjvyw754p-myname.drv»
```

> Oh oh, what's that?
> Did it build the derivation?
> No it didn't, but it **did create the .drv file**.
> `nix repl` does not build derivations unless you tell it to do so.

Так, так, что это?
Это <!-- этот вызов, эта команда -->  построило порождение?
Нет, не построило, но **создало файл .drv**.
`nix repl` не строит порождений, пока вы не попросите его сделать это.

> ## Digression about .drv files

## Отсутпление про файлы .drv

> What's that `.drv` file?
> It is the specification of how to build the derivation, without all the Nix language fuzz.

Что такое файлы `.drv`?
Это спецификация, как строить порождение, без всякого языкового шума <!-- языка --> Nix.

> Before continuing, some analogies with the C language:

Перед тем, как продолжить, несколько аналогий с языком C:

> - `.nix` files are like `.c` files.
> - `.drv` files are intermediate files like `.o` files.
>   The `.drv` describes how to build a derivation; it's the bare minimum information.
> - out paths are then the product of the build.

- Файлы `.nix` похожи на файлы `.c`.
- Файлы `.drv` являются промежуточными файлами наподобие <!-- файлов --> `.o`.
  <!-- Файл --> `.drv` пописывает, как строить порождение: это самый минимум информации.
- В конечном итоге результатом построения являются выходные пути.

> Both drv paths and out paths are stored in the nix store as you can see.

Как вы можете видеть, выходные пути и пути порождения хранятся в хранилище Nix.

> What's in that `.drv` file?
> You can read it, but it's better to pretty print it:

Так что внутри файла `.drv`?
Вы можете прочитать его, но лучше распечатать его содержимое в понятном виде:

> > [!NOTE]
> > If your version of nix doesn't have `nix derivation show`, use `nix show-derivation` instead.

> [!NOTE]
> Если в вашей версия Nix нет команды `nix derivation show`, используйте вместо неё `nix show-derivation`.

```text
$ nix derivation show /nix/store/z3hhlxbckx4g3n9sw91nnvlkjvyw754p-myname.drv
{
  "/nix/store/z3hhlxbckx4g3n9sw91nnvlkjvyw754p-myname.drv": {
    "outputs": {
      "out": {
        "path": "/nix/store/40s0qmrfb45vlh6610rk29ym318dswdr-myname"
      }
    },
    "inputSrcs": [],
    "inputDrvs": {},
    "platform": "mysystem",
    "builder": "mybuilder",
    "args": [],
    "env": {
      "builder": "mybuilder",
      "name": "myname",
      "out": "/nix/store/40s0qmrfb45vlh6610rk29ym318dswdr-myname",
      "system": "mysystem"
    }
  }
}
```

> Ok, we can see there's an out path, but it does not exist yet.
> We never told Nix to build it, but we know beforehand where the build output will be.
> Why?

Так, мы можем видеть здесь выходной путь, но он ещё не существует.
МЫ никогда не просим Nix построить его, но мы заранее знаем, куда будут записаны файлы при строительстве.
Почему?

> Think, if Nix ever built the derivation just because we accessed it in Nix, we would have to wait a long time if it was, say, Firefox.
> That's why Nix let us know the path beforehand and kept evaluating the Nix expressions, but it's still empty because no build was ever made.

Подумайте, если бы мы строили порождение только потому, что обратились к нему в Nix, мы должны были бы ждать долгое время, если это бы это был, например, Firefox.
Вот почему Nix заранее даёт нам знать путь и вычисляет значения выражений Nix, но <!-- путь --> продолжает быть пустым, потому что ещё не было выполнено ни одного построения.

> *Important*: the hash of the out path is based solely on the input derivations in the current version of Nix, not on the contents of the build product.
> It's possible however to have [content-addressable](https://en.wikipedia.org/wiki/Content-addressable_storage) derivations for e.g. tarballs as we'll see later on.

*Важно*: хэш в выходном пути в текущей версии Nix вычисляется исключительно на основе входных порождений, без учёта содержания <!-- строящегося продукта --> текущего порождения.

> Many things are empty in that `.drv`, however I'll write a summary of the [.drv format](http://nixos.org/~eelco/pubs/phd-thesis.pdf) for you:

Многие вещи <!-- поля --> являются пустыми в `.drv`, однако я наскоро опишу [формат .drv](http://nixos.org/~eelco/pubs/phd-thesis.pdf) для вас:

> 1. The output paths (there can be multiple ones).
>    By default nix creates one out path called "out".
> 2. The list of input derivations.
>    It's empty because we are not referring to any other derivation.
>    Otherwise, there would be a list of other .drv files.
> 3. The system and the builder executable (yes, it's a fake one).
> 4. Then a list of environment variables passed to the builder.

1. Выходные пути (их может быть несколько).
   По умолчанию Nix создаёт один выходной путь, называемый "out".
2. Список входных порождений.
   Он пуст, поскольку мы не ссылаемся на любые другие порождения.
   В противном случае, здесь был бы список других файлов .drv.
3. Название системы (system) и путь к построителю (да, сейчас это ненастоящая программа).
4. Напоследок, список переменных окружения, передаваемых в построитель.

> That's it, the minimum necessary information to build our derivation.

Вот и я вся минимальная информация, необходимая для построения порождения.

> *Important note*: the environment variables passed to the builder are just those you see in the .drv plus some other Nix related configuration (number of cores, temp dir, ...).
> The builder will not inherit any variable from your running shell, otherwise builds would suffer from [non-determinism](https://wiki.debian.org/ReproducibleBuilds).

*Важное замечание*: переменные окружения, передаваемые в построитель — это те, которые вы видите в файле .drv плюс некоторые другие, относящиеся к конфигурации Nix (количество ядер, временный каталог и др.).
Построитель не наследут никаких переменных из вашей оболочки, иначе сборки бы страдали от [нон-детерминизма](https://wiki.debian.org/ReproducibleBuilds).

> Back to our fake derivation.

Возвращаемся к нашему учебному порождению.

> Let's build our really fake derivation:

Давайте построим наше поистине ненастоящее порождение:

```text
nix-repl> d = derivation { name = "myname"; builder = "mybuilder"; system = "mysystem"; }
nix-repl> :b d
[...]
these derivations will be built:
  /nix/store/z3hhlxbckx4g3n9sw91nnvlkjvyw754p-myname.drv
building path(s) `/nix/store/40s0qmrfb45vlh6610rk29ym318dswdr-myname'
error: a `mysystem' is required to build `/nix/store/z3hhlxbckx4g3n9sw91nnvlkjvyw754p-myname.drv', but I am a `x86_64-linux'
```

> The `:b` is a `nix repl` specific command to build a derivation.
> You can see more commands with `:?`.
> So in the output you can see that it takes the `.drv` as information on how to build the derivation.
> Then it says it's trying to produce our out path.
> Finally the error we were waiting for: that derivation can't be built on our system.

Команда `:b` доступна только в `nix rpl`, она используется для построения порождения.
Вы можете увидеть больше команд, введя `:?`.
Так, в справке вы можете видеть, что она получается `.drv` в качестве информации о то, как строить порождение.
Затем она говорит, что пытается записать результат по нашему выходному пути.
Наконец, возникает ошибка, которую мы и ожидали: порождение не может быть построено в нашей системе.

> We're doing the build inside `nix repl`, but what if we don't want to use `nix repl`?
> You can **realise** a `.drv` with:

Мы выполняем построение внутри `nix repl`, но что если мы не хотим использовать `nix repl`?
Вы можете **реализовать** `.drv` с помощью команды:

```text
\$ nix-store -r /nix/store/z3hhlxbckx4g3n9sw91nnvlkjvyw754p-myname.drv
```

> You will get the same output as before.

Вы получите тот же вывод, что и в предыдущем случаее.

> Let's fix the system attribute:

Давайте исправим атрибут system:

```text
nix-repl> d = derivation { name = "myname"; builder = "mybuilder"; system = builtins.currentSystem; }
nix-repl> :b d
[...]
build error: invalid file name `mybuilder'
```

> A step forward: of course, that `mybuilder` executable does not really exist.
> Stop for a moment.

Заглядывая вперёд: конечно, исполняемого файла `mybuilder` в действительности не существует.
Остановимся <!-- притормозим --> на мгновение.

> ## What's in a derivation set

## Что есть в наборе порождения

> It is useful to start by inspecting the return value from the derivation function.
> In this case, the returned value is a plain set:

Полезно начать, исследуя результат, возвращаемый функцией derivation.
В нашем случае, возвращаемое значение это простой набор:

```text
nix-repl> d = derivation { name = "myname"; builder = "mybuilder"; system = "mysystem"; }
nix-repl> builtins.isAttrs d
true
nix-repl> builtins.attrNames d
[ "all" "builder" "drvAttrs" "drvPath" "name" "out" "outPath" "outputName" "system" "type" ]
```

> You can guess what `builtins.isAttrs` does; it returns true if the argument is a set.
> While `builtins.attrNames` returns a list of keys of the given set.
> Some kind of reflection, you might say.

Вы можете догадаться, что делает `builtins.isAttrs`; она возвращает true, если аргумент является набором.
В то время как `builtins.attrNames` возвращает список ключей из заданного набора.
Вы можете сказать, что это своего рода рефлексия.

> Start from drvAttrs:

Начнём с drvAttrs:

```text
nix-repl> d.drvAttrs
{ builder = "mybuilder"; name = "myname"; system = "mysystem"; }
```

> That's basically the input we gave to the derivation function.
> Also the `d.name`, `d.system` and `d.builder` attributes are exactly the ones we gave as input.

По сути, это входные данные, которые мы передали функции derivation.
Кроме того, `d.name`, `d.system` и `d.builder` — именно те атрибуты, которые мы указали в качестве входных.

```text
nix-repl> (d == d.out)
true
```

> So out is just the derivation itself, it seems weird but the reason is that we only have one output from the derivation.
> That's also the reason why `d.all` is a singleton.
> We'll see multiple outputs later.

Таким образом, out — это и есть порождение, что кажется странным, но причина этого в том, что мы у нам может быть <!-- только --> один out на порождение.
По той же причине `d.all` — это синглтон.
Позже мы увидим множественный вывод.

> The `d.drvPath` is the path of the `.drv` file: `/nix/store/z3hhlxbckx4g3n9sw91nnvlkjvyw754p-myname.drv`.

`d.drvPath` — это пусть к файлу `.drv`: `/nix/store/z3hhlxbckx4g3n9sw91nnvlkjvyw754p-myname.drv`.

> Something interesting is the `type` attribute.
> It's `"derivation"`.
> Nix does add a little of magic to sets with type derivation, but not that much.
> To help you understand, you can create yourself a set with that type, it's a simple set:

Кое-что интересное об атрибует `type`.
Его значение `"derivation"`.
Nix добавляет небольшую магию при работе с наборами типа derivation, но не очень много.
Чтобы помочь вам разобраться, вы можете создать набор с таким типом, и это обычный набор:

```text
nix-repl> { type = "derivation"; }
«derivation ???»
```

> Of course it has no other information, so Nix doesn't know what to say :-)
> But you get it, the `type = "derivation"` is just a convention for Nix and for us to understand the set is a derivation.

Конечно, в нём нет никакой информации, так что Nix не знает, что печатать.
Как понимаете, `type = "derivation"` — всего лишь соглашение для Nix и для нас, чтобы понимали, что набор — это порождение.

> When writing packages, we are interested in the outputs.
> The other metadata is needed for Nix to know how to create the drv path and the out path.

При написании пакетов, нас интересует вывод.
Другие метаданные нужны Nix чтобы знать, как определить путь drv и путь out.

> The `outPath` attribute is the build path in the nix store:

Атрибут `outPath` — это пусть построения в хранилище Nix: `/nix/store/40s0qmrfb45vlh6610rk29ym318dswdr-myname`.

> ## Referring to other derivations

## Ссылки на другие порождения

> Just like dependencies in other package managers, how do we refer to other packages?
> How do we refer to other derivations in terms of files on the disk?
> We use the `outPath`.
> The `outPath` describes the location of the files of that derivation.
> To make it more convenient, Nix is able to do a conversion from a derivation set to a string.

Как и с зависимостями в других пакетных менеджерах, как нам ссылаться на другие пакеты?
Как нам сослаться на другие порождения в терминах файлов на диске?
Мы используем `outPath`.
`outPath` описывает положение файлов нужного порождения.
Чтобы сделать это более удобным, Nix даёт возможность конвертировать набор derivation в строку.

```text
nix-repl> d.outPath
"/nix/store/40s0qmrfb45vlh6610rk29ym318dswdr-myname"
nix-repl> builtins.toString d
"/nix/store/40s0qmrfb45vlh6610rk29ym318dswdr-myname"
```

> Nix does the "set to string conversion" as long as there is the `outPath` attribute (much like a toString method in other languages):

Nix выполняет "конверсию набора в строку", если в наборе есть атрибут `outPath` (похоже на метод toString в других языках):

```text
nix-repl> builtins.toString { outPath = "foo"; }
"foo"
nix-repl> builtins.toString { a = "b"; }
error: cannot coerce a set to a string, at (string):1:1
```

> Say we want to use binaries from coreutils (ignore the nixpkgs etc.):

Скажем, мы хотим использовать бинарные программы из coreutils (игнорируем nixpkgs и пр.):

```text
nix-repl> :l <nixpkgs>
Added 3950 variables.
nix-repl> coreutils
«derivation /nix/store/1zcs1y4n27lqs0gw4v038i303pb89rw6-coreutils-8.21.drv»
nix-repl> builtins.toString coreutils
"/nix/store/8w4cbiy7wqvaqsnsnb3zvabq1cp2zhyz-coreutils-8.21"
```

> Apart from the nixpkgs stuff, just think we added to the scope a series of variables.
> One of them is coreutils.
> It is the derivation of the coreutils package you all know of from other Linux distributions.
> It contains basic binaries for GNU/Linux systems (you may have multiple derivations of coreutils in the nix store, no worries):

Безотносительно nixpkgs, просто представьте, как будто мы добавили в область видимости серию переменных.
Одна из них это coreutils.
Это порождение пакета coreutils, который знаком вам из других дистрибутивов Linux.
Он содержит основные программы для систем GNU/Linux (не беспокойтесь, у вас может быть несколько порождений coreutils в хранилище).

```text
$ ls /nix/store/*coreutils*/bin
[...]
```

> I remind you, inside strings it's possible to interpolate Nix expressions with `${...}`:

Напоминаю что внутри строк можно интерполировать выражения Nix, с помощью `${...}`:

```text
nix-repl> "${d}"
"/nix/store/40s0qmrfb45vlh6610rk29ym318dswdr-myname"
nix-repl> "${coreutils}"
"/nix/store/8w4cbiy7wqvaqsnsnb3zvabq1cp2zhyz-coreutils-8.21"
```

> That's very convenient, because then we could refer to e.g. the bin/true binary like this:

Это очень удобно, потому что мы можем ссылаться, скажем, на бинарник bin/true следующим образом:

```text
nix-repl> "${coreutils}/bin/true"
"/nix/store/8w4cbiy7wqvaqsnsnb3zvabq1cp2zhyz-coreutils-8.21/bin/true"
```

> ## An almost working derivation

## Почти работающее порождение

> In the previous attempt we used a fake builder, `mybuilder` which obviously does not exist.
> But we can use for example bin/true, which always exits with 0 (success).

В предыдущем примере мы использовали ненастоящий построитель, `mybuilder`, которого, очевидно, не существует.
Но мы можем использовать для экспериментов прораму bin/true, которая всегда завершается с кодом 0 (успех).

```text
nix-repl> :l <nixpkgs>
nix-repl> d = derivation { name = "myname"; builder = "${coreutils}/bin/true"; system = builtins.currentSystem; }
nix-repl> :b d
[...]
builder for `/nix/store/qyfrcd53wmc0v22ymhhd5r6sz5xmdc8a-myname.drv' failed to produce output path `/nix/store/ly2k1vswbfmswr33hw0kf0ccilrpisnk-myname'
```

> Another step forward, it executed the builder (bin/true), but the builder did not create the out path of course, it just exited with 0.

Ещё раз заглядывая вперёд, команда запускат построитель (bin/true), но построитель не создаёт выходного пути (конечно), он просто завершается с кодом 0.

> *Obvious note*: every time we change the derivation, a new hash is created.

*Очевидное замечание*: каждый раз, когда мы меняем порождение, создаётся новый хэш.

> Let's examine the new `.drv` now that we referred to another derivation:

Давайте проверим новый `div`, после того как мы сосолались на другое порождение:

```text
$ nix derivation show /nix/store/qyfrcd53wmc0v22ymhhd5r6sz5xmdc8a-myname.drv
{
  "/nix/store/qyfrcd53wmc0v22ymhhd5r6sz5xmdc8a-myname.drv": {
    "outputs": {
      "out": {
        "path": "/nix/store/ly2k1vswbfmswr33hw0kf0ccilrpisnk-myname"
      }
    },
    "inputSrcs": [],
    "inputDrvs": {
      "/nix/store/hixdnzz2wp75x1jy65cysq06yl74vx7q-coreutils-8.29.drv": [
        "out"
      ]
    },
    "platform": "x86_64-linux",
    "builder": "/nix/store/qrxs7sabhqcr3j9ai0j0cp58zfnny0jz-coreutils-8.29/bin/true",
    "args": [],
    "env": {
      "builder": "/nix/store/qrxs7sabhqcr3j9ai0j0cp58zfnny0jz-coreutils-8.29/bin/true",
      "name": "myname",
      "out": "/nix/store/ly2k1vswbfmswr33hw0kf0ccilrpisnk-myname",
      "system": "x86_64-linux"
    }
  }
}
```

> Aha!
> Nix added a dependency to our myname.drv, it's the coreutils.drv.
> Before doing our build, Nix should build the coreutils.drv.
> But since coreutils is already in our nix store, no build is needed, it's already there with out path
`/nix/store/qrxs7sabhqcr3j9ai0j0cp58zfnny0jz-coreutils-8.29`.

Ага!
Nix добавил зависимость в наш myname.drv, то coreutils.drv.
Перед тем, как делать наше построение, Nix должен построить coreutils.drv.
Но поскольку coreutils уже в нашем хранилище, не нужно ничего строить, оно уже доступно по пути `/nix/store/qrxs7sabhqcr3j9ai0j0cp58zfnny0jz-coreutils-8.29`.

> ## When is the derivation built

## Когда строится порождение

> Nix does not build derivations **during evaluation** of Nix expressions.
> In fact, that's why we have to do ":b drv" in `nix repl`, or use nix-store -r in the first place.

Nix не строит порождений **в процессе вычисления** выражений Nix.
Фактически, в первую очередь именно поэтому мы должны вызывать ":b drv" в `nix repl` или использовать `nix-store -r`.

> An important separation is made in Nix:

В Nix существует важное разделение:

> - **Instantiate/Evaluation time**: the Nix expression is parsed, interpreted and finally returns a derivation set.
>   During evaluation, you can refer to other derivations because Nix will create .drv files and we will know out paths beforehand.
>   This is achieved with [nix-instantiate](https://nixos.org/manual/nix/stable/command-ref/nix-instantiate.html).
> - **Realise/Build time**: the .drv from the derivation set is built, first building .drv inputs (build dependencies).
>   This is achieved with [nix-store -r](https://nixos.org/manual/nix/stable/command-ref/nix-store.html#operation---realise).

- **Время Инстанцирования/Вычисления**: выражения Nix анализируются, интерпретируются и в конце концов возвращают набор порождения.
  При вычислении, вы можете ссылаться на другие порождения потому что Nix создаст файлы .drv и нам заранее известны пути out.
  Это достигается с помощью [nix-instantiate](https://nixos.org/manual/nix/stable/command-ref/nix-instantiate.html).
-- **Время Реализации/Построения**: .drv из порождения строится, предварительно построив .drv входных порождений (зависимости).
  Это достигается с помощью [nix-store -r](https://nixos.org/manual/nix/stable/command-ref/nix-store.html#operation---realise).

> Think of it as of compile time and link time like with C/C++ projects.
> You first compile all source files to object files.
> Then link object files in a single executable.

Думайте об этом, как о времени компиляции и времени компоновки в проектах C/C++.
Сначала вы компилируете все исходные файлы в объектные файлы.
А затем компонуете объектные файлы в один исполняемый файл.

> In Nix, first the Nix expression (usually in a .nix file) is compiled to .drv, then each .drv is built and the product is installed in the relative out paths.

В Nix, сначало выражение Nix (обычно из файла .nix) компилируется в .drv, и затем каждый .drv строится и результат инсталлируется по относительным путям out.

> ## Conclusion

## Заключение

> Is it that complicated to create a package for Nix?
> No, it's not.

Сложно ли создать пакет в Nix?
Нет, не сложно.

> We're walking through the fundamentals of Nix derivations, to understand how they work, how they are represented.
> Packaging in Nix is certainly easier than that, but we're not there yet in this post.
> More Nix pills are needed.

Мы рассмотрели основы порождений Nix, чтобы понять, как они работают, как они представлены <!-- хранятся? -->.
Создание пакетов в Nix всё-таки проще, чем мы обсуждали, но у этого поста была другая цель.
Нужны ещё пилюли Nix.

> With the derivation function we provide a set of information on how to build a package, and we get back the information about where the package was built. Nix converts a set to a string when there's an `outPath`; that's very convenient.
> With that, it's easy to refer to other derivations.

С помощью функции `derivation` мы предоставляем информацию,как строить пакет, и мы получаем обратно информацию о том, где пакет был построен. Nix конвертирует набор в строку, когда в наборе есть атрибут `outPath`; это очень удобно.
Благодаря этому, лекго ссылаться на другие порождения.

> When Nix builds a derivation, it first creates a .drv file from a derivation expression, and uses it to build the output.
> It does so recursively for all the dependencies (inputs).
> It "executes" the .drv files like a machine.
> Not much magic after all.

Когда Nix строит порождение, он сначала создаёт файл .drv из порождающего выражения, и использует его для построения выходных файлов.
Он делает это рекурсивно для всех зависимостей (входящих .drv).
Он "выполняет" файлы .drv, как машина.
Не так уж много магии, в конце концов.

> ## Next pill

## В следующей пилюле

> ...we will finally write our first **working** derivation.
> Yes, this post is about "our first derivation", but I never said it was a working one ;)

Мы в конце концов напишем наше первое **работающее** порождение.
Да, этот посто тоже про "наше первое порождение", но я никогад не утверждал, что оно будет работать.
