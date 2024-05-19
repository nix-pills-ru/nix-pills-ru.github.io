> # Automatic Runtime Dependencies

# Автоматические зависимости времени выполнения

> Welcome to the 9th Nix pill.
> In the previous [8th pill](#generic-builders) we wrote a generic builder for autotools projects.
> We fed in build dependencies and a source tarball, and we received a Nix derivation as a result.

Добро пожаловать на девятую пилюлю Nix.
В предыдущей [восьмой пилюле](08-generic-builders.md) мы написали универсальный скрипт сборки для проектов `autotools`.
Мы загрузили зависимости сборки и исходники, и получили в качестве результата деривацию Nix.

> Today we stop by the GNU `hello` program to analyze build and runtime dependencies, and we enhance our builder to eliminate unnecessary runtime dependencies.

Сегодня мы обратимся к программе GNU `hello`, чтобы проанализировать зависимости сборки и времени выполнения, а также усовершенствуем наш скрипт, чтобы исключить ненужные зависимости времени выполнения.

> ## Build dependencies

## Зависимости сборки

> Let's start analyzing build dependencies for our GNU `hello` package:

Давайте начнём анализ зависимостей сборки для пакета GNU `hello`:

```text
$ nix-instantiate hello.nix
/nix/store/z77vn965a59irqnrrjvbspiyl2rph0jp-hello.drv
$ nix-store -q --references /nix/store/z77vn965a59irqnrrjvbspiyl2rph0jp-hello.drv
/nix/store/0q6pfasdma4as22kyaknk4kwx4h58480-hello-2.10.tar.gz
/nix/store/1zcs1y4n27lqs0gw4v038i303pb89rw6-coreutils-8.21.drv
/nix/store/2h4b30hlfw4fhqx10wwi71mpim4wr877-gnused-4.2.2.drv
/nix/store/39bgdjissw9gyi4y5j9wanf4dbjpbl07-gnutar-1.27.1.drv
/nix/store/7qa70nay0if4x291rsjr7h9lfl6pl7b1-builder.sh
/nix/store/g6a0shr58qvx2vi6815acgp9lnfh9yy8-gnugrep-2.14.drv
/nix/store/jdggv3q1sb15140qdx0apvyrps41m4lr-bash-4.2-p45.drv
/nix/store/pglhiyp1zdbmax4cglkpz98nspfgbnwr-gnumake-3.82.drv
/nix/store/q9l257jn9lndbi3r9ksnvf4dr8cwxzk7-gawk-4.1.0.drv
/nix/store/rgyrqxz1ilv90r01zxl0sq5nq0cq7v3v-binutils-2.23.1.drv
/nix/store/qzxhby795niy6wlagfpbja27dgsz43xk-gcc-wrapper-4.8.3.drv
/nix/store/sk590g7fv53m3zp0ycnxsc41snc2kdhp-gzip-1.6.drv
```

> It has precisely the derivations referenced in the `derivation` function; nothing more, nothing less.
> Of course, we may not use some of them at all.
> However, given that our generic `mkDerivation` function always pulls such dependencies (think of it like [build-essential](https://packages.debian.org/unstable/build-essential) from Debian), we will already have these packages in the nix store for any future packages that need them.

Список содержит ровно ссылки на деривации в функции `derivation`; ни больше, ни меньше.
Конечно, вы можете не использовать некоторые из них вообще.
Однако, учитывая, что наша универсальная функция `mkDerivation` всегда извлекает такие зависимости (сравните с [пакетом build-essential](https://packages.debian.org/unstable/build-essential) из Debian), мы уже будем иметь эти пакеты в хранилище Nix для всех других пакетов, которым они потребуются.

> Why are we looking at `.drv` files?
> Because the `hello.drv` file is the representation of the build action that builds the `hello` out path.
> As such, it contains the input derivations needed before building `hello`.

Почему мы видим файлы `.drv`?
Потому что файл `hello.drv` представляет собой действие, которое собирает программу `hello` по выходному пути.
Как таковой, он содержит входные деривации, нужные для сборки `hello`.

> ## Digression about NAR files

## Отступление про файлы NAR

> The `NAR` format is the "Nix ARchive".
> This format was designed due to existing archive formats, such as `tar`, being insufficient.
> Nix benefits from deterministic build tools, but commonly used archivers lack this property: they add padding, they do not sort files, they add timestamps, and so on.
> This can result in directories containing bit-identical files turning into non-bit-identical archives, which leads to different hashes.

Формат `NAR` это "Nix ARchive".
Этот формат был разработан, потому что существующие форматы архивов, такие как `tar`, оказались недостаточны.
Nix получает бонусы от детерменированных инструментов сборки, но широко используемые архиваторы игнорируют это свойство: один добавляют дополнения <!-- выравнивания -->, они не сортируют файлы, они добавляют метки времени и так далее.
Результатом этого является то, что директории, содержащие побитово-идентичные файлы превращаются в побитово-неидентичные архивы, что приводит к различным хэшам.

> Thus the `NAR` format was developed as a simple, deterministic archive format.
> `NAR`s are used extensively within Nix, as we will see below.

Таким образом, формат `NAR` был разработан как простой детерминированный формат архива.
`NAR` широко используется в Nix, как мы увидим ниже.

> For more rationale and implementation details behind `NAR` see [Dolstra's PhD Thesis](http://nixos.org/~eelco/pubs/phd-thesis.pdf).

Более подробное обоснование и детали реализации `NAR` см. в [докторской диссертации Долстры](http://nixos.org/~eelco/pubs/phd-thesis.pdf).

> To create NAR archives from store paths, we can use `nix-store --dump` and `nix-store --restore`.

Чтобы создавать архивы NAR из путей хранилища, мы можем использовать `nix-store --dump` и `nix-store --restore`.

> ## Runtime dependencies

## Зависимости времени выполнения

> We now note that Nix automatically recognized build dependencies once our `derivation` call referred to them, but we never specified the runtime dependencies.

Теперь заметим, что Nix автоматически распознаёт зависимости сборки, как только вызов функции `derivation` ссылается на них, но мы никогда не указывали зависимости времени выполнения.

> Nix handles runtime dependencies for us automatically.
> The technique it uses to do so may seem fragile at first glance, but it works so well that the NixOS operating system is built off of it.
> The underlying mechanism relies on the hash of the store paths.
> It proceeds in three steps:

Nix обрабатывает зависимости времени выполнения для нас автоматически.
Техника, которую он использует, чтобы это делать, может показаться хрупкой на первый взгляд, но она работает так хорошо, что на ней построена операционная система NixOS.
Базовый механизм построен на хэшах путей в хранилище.
Он состоит из трёх шагов. 

> 1. Dump the derivation as a NAR.
>    Recall that this is a serialization of the derivation output -- meaning this works fine whether the output is a single file or a directory.
> 2. For each build dependency `.drv` and its relative out path, search    the contents of the NAR for this out path.
> 3. If the path is found, then it's a runtime dependency.

1. Создание архива NAR из деривации.
   Помните, что этот шаг сериализует вывод деривации — он хорошо работает и тогда, когда деривация это один файл, и тогда, когда это целый каталог.
2. Для каждого файла `.drv`, от которого зависит сборка и её относительного выходного пути, искать архив NAR по этому пути.
3. Если архив найден, то путь является зависимостью времени выполнения.


> The snippet below shows the dependencies for `hello`.

В приведённом ниже фрагменте показаны зависимости `hello`.

```text
$ nix-instantiate hello.nix
/nix/store/z77vn965a59irqnrrjvbspiyl2rph0jp-hello.drv
$ nix-store -r /nix/store/z77vn965a59irqnrrjvbspiyl2rph0jp-hello.drv
/nix/store/a42k52zwv6idmf50r9lps1nzwq9khvpf-hello
$ nix-store -q --references /nix/store/a42k52zwv6idmf50r9lps1nzwq9khvpf-hello
/nix/store/94n64qy99ja0vgbkf675nyk39g9b978n-glibc-2.19
/nix/store/8jm0wksask7cpf85miyakihyfch1y21q-gcc-4.8.3
/nix/store/a42k52zwv6idmf50r9lps1nzwq9khvpf-hello
```

> We see that `glibc` and `gcc` are runtime dependencies.
> Intuitively, `gcc` shouldn't be in this list! Displaying the printable strings in the `hello` binary shows that the out path of `gcc` does indeed appear:

Мы видим, что `glibc` и `gcc` являются зависимостями времени выполнения.
Интуитивно, `gcc` не должен быть в этом списке! Но вывод на экран печатных строк из двоичного файла `hello` показывает, что выходной путь `gcc` действительно появляется:

```text
$ strings result/bin/hello|grep gcc
/nix/store/94n64qy99ja0vgbkf675nyk39g9b978n-glibc-2.19/lib:/nix/store/8jm0wksask7cpf85miyakihyfch1y21q-gcc-4.8.3/lib64
```

> This is why Nix added `gcc`.
> But why is that path present in the first place?
> The answer is that it is the [ld rpath](http://en.wikipedia.org/wiki/Rpath): the list of directories where libraries can be found at runtime.
> In other distributions, this is usually not abused.
> But in Nix, we have to refer to particular versions of libraries, and thus the rpath has an important role.

Вот почему Nix добавил `gcc`.
Но почему этот путь вообще присутствует?
Ответ в том, что он есть в [ld rpath](http://en.wikipedia.org/wiki/Rpath): списке каталогов, где могут быть найдены библиотеки времени выполнения.
В других дистрибутивах этим, обычно, не злоупотребляют.
Но в Nix нам приходится ссылаться на определённые версии библиотек, поэтому `rpath` играет важную роль.


> The build process adds the `gcc` lib path thinking it may be useful at runtime, but this isn't necessary.
> To address issues like these, Nix provides a tool called [patchelf](https://nixos.org/patchelf.html), which reduces the rpath to the paths that are actually used by the binary.

Процесс сборки добавляет путь к библиотекам `gcc`, думаю, что он может быть полезен во время выполнения, но это не обязательно.
Для решения подобных проблем, Nix предоставляет инструмент под названием [patchelf](https://nixos.org/patchelf.html), который сводит `rpath` к путям, которые действительно используются в исполняемом файле.

> Even after reducing the rpath, the `hello` binary would still depend upon `gcc` because of some debugging information.
> This unnecessarily increases the size of our runtime dependencies.
> We'll explore how `strip` can help us with that in the next section.

Даже после сокращения `rpath`, исполняемый файл `hello` по прежнему будет зависеть от `gcc` из-за некоторой отладочной информации.
В следующем разделе мы исследуем, как `strip` может нам в этом помочь.


> ## Another phase in the builder

## Другая фаза в скрипте сборки

> We will add a new phase to our autotools builder.
> The builder has six phases already:

Мы добавим новую фазу в наш скрипт сборки проектов `autotools`.
Сейчас сборщик имеет шесть фаз:

> 1. The "environment setup" phase
> 2. The "unpack phase": we unpack the sources in the current directory (remember, Nix changes to a temporary directory first)
> 3. The "change directory" phase, where we change source root to the directory that has been unpacked
> 4. The "configure" phase: `./configure`
> 5. The "build" phase: `make`
> 6. The "install" phase: `make install`

1. Фаза "настройки окружения"
2. Фаза "распаковки": мы распаковываем исходники в текущий каталог (помните, что сначала Nix переходит во временный каталог)
3. Фаза "смены каталога": где каталог, куда мы распаковали исходники, становится корнем дерева исходников
4. Фаза "конфигурирования": `./configure`
5. Фаза "сборки": `make`
6. Фаза "установки": `make install`

> Now we will add a new phase after the installation phase, which we call the "fixup" phase.
> At the end of the `builder.sh`, we append:

Теперь мы добавим новую фазу после фазы устновки, мы назовём её фазой исправления.
Мы добавляем в конец `builder.sh`:

```text
find $out -type f -exec patchelf --shrink-rpath '{}' \; -exec strip '{}' \; 2>/dev/null
```

> That is, for each file we run `patchelf --shrink-rpath` and `strip`.
> Note that we used two new commands here, `find` and `patchelf`.
> These must be added to our derivation.

То есть, для каждого файла мы выполняем `patchelf --shrink-rpath` и `strip`.
Заметьте, что мы использовали здесь две новые команды, `find` и `patchelf`.
Они должны быть добавлены в нашу деривацию.

> **Exercise:** Add `findutils` and `patchelf` to the `baseInputs` of `autotools.nix`.

**Упражнение:** Добавьте `findutils` и `patchelf` к `baseInputs` скрипта `autotools.nix`.

> Now, we rebuild `hello.nix`...

Теперь мы пересобираем `hello.nix`...

```text
$ nix-build hello.nix
[...]
$ nix-store -q --references result
/nix/store/94n64qy99ja0vgbkf675nyk39g9b978n-glibc-2.19
/nix/store/md4a3zv0ipqzsybhjb8ndjhhga1dj88x-hello
```

> and we see that `glibc` is a runtime dependency.
> This is exactly what we wanted.

и мы видим, что в списке зависимостей времени выполнения осталась только библиотека `glibc`.
Это именно то, чего мы добивались.

> The package is self-contained.
> This means that we can copy its closure onto another machine and we will be able to run it.
> Remember, only a very few components under the `/nix/store` are required to [run nix](#install-on-your-running-system).
> The `hello` binary will use the exact version of `glibc` library and interpreter referred to in the binary, rather than the system one:

Этот пакет самодостаточен (автономен).
Это значит, что мы можем скопировать на другую машину только его и мы сможем его запустить.
Запомните, что только несколько компонентов в каталоге `nix/store` требуют [запуска nix](02-install-on-your-running-system.md).
Исполняемый файл `hello` будет использовать не системные версии библиотеки `glibc` и интерпретатора, а те, которые прописаны в нём.

```text
$ ldd result/bin/hello
 linux-vdso.so.1 (0x00007fff11294000)
 libc.so.6 => /nix/store/94n64qy99ja0vgbkf675nyk39g9b978n-glibc-2.19/lib/libc.so.6 (0x00007f7ab7362000)
 /nix/store/94n64qy99ja0vgbkf675nyk39g9b978n-glibc-2.19/lib/ld-linux-x86-64.so.2 (0x00007f7ab770f000)
```

> Of course, the executable will run fine as long as everything is under the `/nix/store` path.

Конечно, исполняемый файл будет прекрасно работать, пока всё находится в каталоге `/nix/store`.

> ## Conclusion

## Заключение

> We saw some of the tools Nix provides, along with their features.
> In particular, we saw how Nix is able to compute runtime dependencies automatically.
> This is not limited to only shared libraries, but can also reference executables, scripts,
> Python libraries, and so forth.

Мы познакомились с некоторыми инструментами, которые предоставляет Nix, вместе с их возможностями.
В частности, мы увидели, как Nix может автоматически определять зависимости времени выполнения.
Это касатеся не только разделяемых библиотек, но также исполняемых файлов, скриптов, библиотек Python, и так далее.

> Approaching builds in this way makes packages self-contained, ensuring (apart from data and configuration) that copying the runtime closure onto another machine is sufficient to run the program.
> This enables us to run programs without installation using `nix-shell`, and forms the basis for [reliable deployment in the cloud](https://nixos.org/manual/nix/stable/introduction.html).

Такой подход к сборке делает пакеты самодостаточными, гарантируя, что копирование <!-- runtime closure --> на другую машину достаточно для запуска программы.
Это позволяет нам запускать программу без установки, используя `nix-shell` и формирует основу для [надёжного развёртывания в облаке](https://nixos.org/manual/nix/stable/introduction.html).

> ## Next pill

## В следующей пилюле

> The next pill will introduce `nix-shell`.
> With `nix-build`, we've always built derivations from scratch: the source gets unpacked, configured, built, and installed.
> But this can take a long time for large packages.
> What if we want to apply some small changes and compile incrementally instead, yet still want to keep a self-contained environment similar to nix-build`? `nix-shell` enables this.

Следующая пилюля расскажет про `nix-shell`.
С помощью `nix-build` мы всегда строили деривации с нуля: исходные коды распаковывались, конфигурировались, собирались и устанавливались.
Но это может занимать много времени для больших пакетов.
Что если вместо этого мы хотим применять несколько небольших изменений и использовать инкрементальную компиляцию, всё ещё желая оставить самодостаточное окружение, как это сделано в `nix-build`? `nix-shell` разрешает это.
