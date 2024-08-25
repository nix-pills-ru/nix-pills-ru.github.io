> # Developing with `nix-shell`

# Разработка с помощью `nix-shell`

> Welcome to the 10th Nix pill.
In the previous [9th pill](#automatic-runtime-dependencies) we saw one of the  powerful features of Nix: automatic discovery of runtime dependencies.
> We also finalized the GNU `hello` package.

Добро пожаловать на десятую пилюлю Nix.
В предыдущей [девятой пилюле](09-automatic-runtime.md) мы познакомились с одной из мощных возможностей Nix: автоматическим определением зависимостей времени выполнения.
Заодно мы завершили разработку пакета `hello`.

> In this pill, we will introduce the `nix-shell` tool and use it to hack on the GNU `hello` program.
> We will see how `nix-shell` gives us an isolated environment while we modify the source files of the project, similar to how `nix-build` gave us an isolated environment while building the derivation.

В этой пилюле мы познакомимся с утилитой `nix-shell` и используем её, чтобы взломать программу `hello`.
Мы увидим, как создать изолированное окружение с помощью `nix-shell` одновременно редактируя исходные файлы проекта тем же способом, как `nix-build` предоставляет нам изолированное окружение по время сборки деривации.

> Finally, we will modify our builder to work more ergonomically with a `nix-shell`-focused workflow.

В конце концов мы сделаем наш скипт сборки более эгрономичным, ориентируясь на возможности `nix-shell`

> ## What is `nix-shell`?

## Что такое `nix-shell`?

> The [nix-shell](https://nixos.org/manual/nix/stable/command-ref/nix-shell.html) tool drops us in a shell after setting up the environment variables necessary to hack on a derivation.
> It does not build the derivation; it only serves as a preparation so that we can run the build steps manually.

Утилита [nix-shell](https://nixos.org/manual/nix/stable/command-ref/nix-shell.html) помещает нас в командную оболочку с настроенными переменными окружения, необходимыми для «взлома» деривации.
Она не собирает деривацию; она служит только в качестве подготовки, чтобы можно было собрать пакет пошагово. 

> Recall that in a nix environment, we don't have access to libraries or programs unless they have been installed with `nix-env`.
> However, installing libraries with `nix-env` is not good practice.
> We prefer to have isolated environments for development, which `nix-shell` provides for us.

Давайте вспомним, что в Nix у нас нет доступа к библиотекам или программам, пока они не установлены с помощью `nix-env`.
Однако, установка библиотек через `nix-env` не является хорошей практикой.
Мы предпочитаем изолированное окружение для разработки, которое нам предоставляет `nix-shell`.

> We can call `nix-shell` on any Nix expression which returns a derivation, but the resulting `bash` shell's `PATH` does not have the utilities we want:

Мы можем передать в `nix-shell` любое выражение Nix, возвращающее деривацию, но в переменной `PATH`, которая в конечном итоге попадёт в `bash`, не будет нужных нам утилит.

```bash
$ nix-shell hello.nix
[nix-shell]$ make
bash: make: command not found
[nix-shell]$ echo $baseInputs
/nix/store/jff4a6zqi0yrladx3kwy4v6844s3swpc-gnutar-1.27.1 [...]
```

> This shell is rather useless.
> It would be reasonable to expect that the GNU `hello` build inputs are available in `PATH`, including GNU `make`, but this is not the case.

Такая оболочка довольно бесполезна.
Было бы разумно ожидать, что программы, описанные в `$buildInputs`, попадают в `PATH` (в том числе и программа GNU `make`), но только не в этом случае.

> However, we do have the environment variables that we set in the derivation, like `$baseInputs`, `$buildInputs`, `$src`, and so on.

Однако, у нас есть переменные окружения, которые мы установили в деривации, в частнотси `$baseInputs`, `$buildInputs`, `$src` и др.

> This means that we can `source` our `builder.sh`, and it will build the derivation.
> You may get an error in the installation phase, because your user may not have the permission to write to `/nix/store`:

Это значит, что мы можем запустить наш `builder.sh` и он построит деривацию.
На фазе установки у вас может возникнуть ошибка, потому что ваш пользователь не имеет права писать в `/nix/store`:

```bash
[nix-shell]$ source builder.sh
...
```

> The derivation didn't install, but it did build. Note the following:

Деривация не установилась, но она была построена. Обратите внимание на следующие моменты:

> - We sourced `builder.sh` and it ran all of the build steps, including setting up the `PATH` for us.
> - The working directory is no longer a temp directory created by `nix-build`, but is instead the directory in which we entered the shell.
>   Therefore, `hello-2.10` has been unpacked in the current directory.

- Мы запустили `builder.sh` и он выполнил все шаги сборки, включая настройку `PATH`.
- Рабочий каталог — больше не временный каталог, созданный `nix-build`, а каталог, в котором мы запустили оболочку.
- Таким образом, `hello-2.10` был распокован в текущий каталог.

> We are able to `cd` into `hello-2.10` and type `make`, because `make` is now available.

Мы можем войти в каталог `hello-2.10` и запустить `make`, поскольку `make` теперь доступен.

> The take-away is that `nix-shell` drops us in a shell with the same (or very similar) environment used to run the builder.

Вывод в том, что `nix-shell` помещает нас в оболочку с таким же (или очень похожим) окружением, как и во время сборки.

> ## A builder for nix-shell

## Сборщик для `nix-shell`

> The previous steps require some manual commands to be run and are not optimized for a workflow centered on `nix-shell`.
> We will now improve our builder to be more `nix-shell` friendly.

Предыдущие шаги требуют ручного запуска команд и не оптимизированы для работы с `nix-shell`.
Сейчас мы улучшим наш сборщик, чтобы он был более `nix-shell`-дружественным.

> There are a few things that we would like to change.

Есть несколько вещей, которые нам надо изменить.

> First, when we `source`d the `builder.sh` file, we obtained the file in the current directory.
> What we really wanted was the `builder.sh` that is stored in the nix store, as this is the file that would be used by `nix-build`.
> To achieve this, the correct technique is to pass an environment variable through the derivation.
> (Note that `$builder` is already defined, but it points to the bash executable rather than our `builder.sh`.
> Our `builder.sh` is passed as an argument to bash.)

Во-первых, когда мы загружаем `builder.sh`, мы загружаем его в текущий каталог.
Что мы действительно хотим, так это поместить `builder.sh` в хранилище Nix, поскольку именно этот файл используется утилитой `nix-build`.
Правильный способ заключается в том, чтобы передать в деривацию нужную переменную окружения.
(Обратите внимание, что переменная `$builder` уже определена, но она указывает на исполняемый файл `bash` вместо `builder.sh`.
Наш `builder.sh` передаётся в `bash` как аргумент.)

> Second, we don't want to run the whole builder: we only want to setup the necessary environment for manually building the project.
> Thus, we can break `builder.sh` into two files: a `setup.sh` for setting up the environment, and the real `builder.sh` that `nix-build` expects.

Во-вторых, мы не хотим запускать сборку полностью, мы всего лишь хотим настроить необходимое окружение для ручной сборки проекта.
Поэтому мы можем разбить `builder.sh` на два файла: `setup.sh` для настройки окружения и настоящий `builder.sh`, который и ожидает `nix-build`.

> During our refactoring, we will wrap the build phases in functions to give more structure to our design.
> Additionally, we'll move the `set -e` to the builder file instead of the setup file.
> The `set -e` is annoying in `nix-shell`, as it will terminate the shell if an error is encountered (such as a mistyped command.)

В процессе рефакторинга мы завернём фазы сборки в функции, чтобы придать больше структуры нашему дизайну.
Дополнительно, мы перенесём `set -e` из файла настройки в файл сборки.
Команда `set -e` в `nix-shell` раздражает, поскольку она завершает работу оболочки, при возникновении ошибки.

> Here is our modified `autotools.nix`.
> Noteworthy is the `setup = ./setup.sh;` attribute in the derivation, which adds `setup.sh` to the nix store and correspondingly adds a `$setup` environment variable in the builder.

Вот наш исправленный `autotools.nix`.
Примечательным является атрибут `setup = ./setup.sh` в деривации, который добавляет `setup.sh` в хранилище Nix и соответственно инициализирует переменную окружения `$setup` в сборщике.

```nix
pkgs: attrs:
let
  defaultAttrs = {
    builder = "${pkgs.bash}/bin/bash";
    args = [ ./builder.sh ];
    setup = ./setup.sh;
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
      patchelf
      findutils
    ];
    buildInputs = [ ];
    system = builtins.currentSystem;
  };
in
derivation (defaultAttrs // attrs)
```

> Thanks to that, we can split `builder.sh` into `setup.sh` and `builder.sh`.
> What `builder.sh` does is `source` `$setup` and call the `genericBuild` function.
> Everything else is just some changes to the bash script.

Благодаря этому мы можем разделить `builder.sh` на `setup.sh` и `builder.sh`.
Задача `builder.sh` заключается в том, чтобы загрузить `$setup` и вызвать функцию `genericBuild`.
Всё остальное — небольшие изменения в скрипте `bash`.

> Here is the modified `builder.sh`:

Вот исправленная версия `builder.sh`:

```bash
set -e
source $setup
genericBuild
```

> Here is the newly added `setup.sh`:

Вот новая добавленная версия `setup.sh`:

```bash
unset PATH
for p in $baseInputs $buildInputs; do
    export PATH=$p/bin${PATH:+:}$PATH
done

function unpackPhase() {
    tar -xzf $src

    for d in *; do
    if [ -d "$d" ]; then
        cd "$d"
        break
    fi
    done
}

function configurePhase() {
    ./configure --prefix=$out
}

function buildPhase() {
    make
}

function installPhase() {
    make install
}

function fixupPhase() {
    find $out -type f -exec patchelf --shrink-rpath '{}' \; -exec strip '{}' \; 2>/dev/null
}

function genericBuild() {
    unpackPhase
    configurePhase
    buildPhase
    installPhase
    fixupPhase
}
```

> Finally, here is `hello.nix`:

Наконец, вот `hello.nix`:

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

> Now back to nix-shell:

Возвращаемся в `nix-shell`:

```nix
$ nix-shell hello.nix
[nix-shell]$ source $setup
[nix-shell]$
```

> Now, for example, you can run `unpackPhase` which unpacks `$src` and enters the directory.
And you can run commands like `./configure`, `make`, and so forth manually, or run phases with their respective functions.

Теперь, например, вы можете запустить `unpackPhase`, которая распакует `$src` и зайдёт в каталог.
И вы можете запускать такие команды, как `./configure`, `make` и т.д. вручную, или запускать фазы через их соответствующие функции.

> The process is that straightforward. `nix-shell` builds the `.drv` file and its input dependencies, then drops into a shell by setting up the environment variables necessary to build the `.drv`.
> In particular, the environment variables in the shell match those passed to the `derivation` function.

Процесс именно такой простой. `nix-shell` собирает файл `.drv` и все его входные зависимости, затем запускает командную оболочку с настроенными переменными окружения, необходимыми для сборки `.drv`.
В частности, переменные окружения в оболочке совпадают с теми, которые передаются в фукнкцию `derivation`.

> ## Conclusion

## Заключение

> With `nix-shell` we are able to drop into an isolated environment suitable for developing a project.
> This environment provides the necessary dependencies for the development shell, similar to how `nix-build` provides the necessary dependencies to a builder.
> Additionally, we can build and debug the project manually, executing step-by-step like we would in any other operating system.
> Note that we never installed tools such `gcc` or `make` system-wide; these tools and libraries are isolated and available per-build.

С помощью `nix-shell` мы можем попасть в изолированное окружение, подходящее для разработки проекта.
Это окружение предоставляет необходимые зависимости для оболочки разработчика, подобно тому, как `nix-build` предоставляет необходимые зависимости сборщику.
Дополнительно, мы можем собирать и отлаживать проект вручную, выполняя его пошагово, как мы делали бы в любой другой операционной системе.
Заметьте, что мы никогда не устанавливаем такие инструменты, как `gcc` или `make` в систему; эти инструменты и библиотеки изолированы и доступны попроектно.

> ## Next pill

## В следующей пилюле

> In the next pill, we will clean up the nix store.
> We have written and built derivations which add to the nix store, but until now we haven't worried about cleaning up the used space in the store.

В следующей пилюле мы очистим хранилище Nix.
Мы написали и построили деривации, которые добавили в хранилище Nix, но до этого момента мы не беспокоились о том, чтобы чистить использованное пространство в хранилище.