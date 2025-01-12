> # Nixpkgs Parameters

# Параметры nixpkgs

> Welcome to the 16th Nix pill. In the previous [15th](#nix-search-paths) pill we\'ve realized how nix finds expressions with the angular brackets syntax, so that we finally know where `<nixpkgs>` is located on our system.

Добро пожаловать на шестнадцатую пилюлю Nix.
В предыдущей [пятнадцатой пилюле](15-nix-search-paths.md) мы узнали, как Nix ищет выражения в угловых скобках, так что теперь мы знаем, где в вашей системе находится `<nixpkgs>`.


> We can start diving into the [nixpkgs repository](https://github.com/NixOS/nixpkgs), through all the various tools and design patterns.
> Please note that also `nixpkgs` has its own manual, underlying the difference between the general `nix` language and the `nixpkgs` repository.

Теперь мы готовы к погружение в [репозиторий nixpkgs](https://github.com/NixOS/nixpkgs) сквозь все разнообразные инструменты и паттерны проектирования.
Пожалуйста обратите также внимание, что у `nixpkgs` есть собственное руководство, что подчёркивает различие между языком Nix "вообще" и конкретным репозиторием `nixpkgs`.

> ## The default.nix expression

# Выражение по умолчанию default.nix

> We will not start inspecting packages at the beginning, rather the general structure of `nixpkgs`.

Мы не станем исследовать пакеты сразу, а сначала разберёмся с общей структурой `nixpkgs`.

> In our custom repository we created a `default.nix` which composed the expressions of the various packages.

В нашем собственном репозитории вы создали `default.nix`, который объединяет выражения нескольких пакетов.

> Also `nixpkgs` has its own [default.nix](https://github.com/NixOS/nixpkgs/blob/master/default.nix), which is the one being loaded when referring to `<nixpkgs>`.
> It does a simple thing: check whether the `nix` version is at least 1.7 (at the time of writing this blog post).
> Then import [pkgs/top-level/all-packages.nix](https://github.com/NixOS/nixpkgs/blob/master/pkgs/top-level/all-packages.nix).
> From now on, we will refer to this set of packages as **pkgs**.

Точно также и у `nixpkgs` есть собственный [default.nix](https://github.com/NixOS/nixpkgs/blob/master/default.nix), который загружается, когда кто-то ссылается на `<nixpkgs>`.
Он проверяет, что версия Nix не меньше, чем 1.7 (на момент написания оригинального поста).
Затем ипортирует [pkgs/top-level/all-packages.nix](https://github.com/NixOS/nixpkgs/blob/master/pkgs/top-level/all-packages.nix).
Начиная с этого момента, мы будем называть этот набор пакетов **pkgs**.

> The `all-packages.nix` is then the file that composes all the packages.
> Note the `pkgs/` subdirectory, while nixos is in the `nixos/` subdirectory.

Настоящий файл, который объединяет все пакеты, это `all-packages.nix`.
Обратите внимание, что пакеты хранятся в подкаталоге `pkgs/`, в то время, как сама NixOS — в подкаталоге `nixos/`.

> The `all-packages.nix` is a bit contrived.
> First of all, it\'s a function.
> It accepts a couple of interesting parameters:

Содерижмое `all-packages.nix` весьма интересно.
Во-первых, это функция.
Она принимает пару интересных параметров:

> -   `system`: defaults to the current system
> -   `config`: defaults to null
> -   others\...

- `system`: по умолчанию — текущая система
- `config`: по умолчанию `null`
- прочее...

> The **system** parameter, as per comment in the expression, it\'s the system for which the packages will be built.
> It allows for example to install i686 packages on amd64 machines.

Параметр **system**, если верить комментарию в выражении — это система, для которой будут собираться пакеты.
Он, например, позволяет устанавливать пакеты i686 на машины amd64.

> The **config** parameter is a simple attribute set.
> Packages can read some of its values and change the behavior of some derivations.

Параметр **config** — это просто набор атрибутов.
Пакеты могут читать эти значения и изменять поведение некоторых дериваций.

> ## The system parameter

## Параметр system

> You will find this parameter in many other .nix expressions (e.g. release expressions).
> The reason is that, given pkgs accepts a system parameter, then whenever you want to import pkgs you also want to pass through the value of system.
> E.g.:

Вы обнаружите этот параметр во многих выражениях `.nix`, скажем, в выражениях выпусков (releases).
Причина в том, что `pkgs` принимает параметр `system`, так что, импортируя `pkgs`, вы можете передать в него нужное значение.

`myrelease.nix`:

```nix
{ system ? builtins.currentSystem }:

let pkgs = import <nixpkgs> { inherit system; };
...
```

> Why is it useful?
> With this parameter it\'s very easy to select a set of packages for a particular system.
> For example:

Для чего это может понадобиться?
С этим параметром достаточно опросто выбрать набор пакетов конкретной системы.
Например:

```bash
nix-build -A psmisc --argstr system i686-linux
```

> This will build the psmisc derivation for i686-linux instead of x86_64-linux.
> This concept is very similar to multi-arch of Debian.

Этот вызов соберёт деривацию `psmisc` для i686-linux вместо x86_67-linux.
Эта концепция очень похожа на мультиархивы Debian.

> The setup for cross compiling is also in `nixpkgs`, however it\'s a little contrived to talk about it and I don\'t know much of it either.

Настройка кросс-компиляции есть и в `nixpkgs`, но мы не будем погружаться в изучение этого вопроса, поскольку я в нём не разбираюсь.

> ## The config parameter

## Параметр config

> I\'m sure on the wiki or other manuals you\'ve read about `~/.config/nixpkgs/config.nix` (previously `~/.nixpkgs/config.nix`) and I\'m sure you\'ve wondered whether that\'s hardcoded in nix.
> It\'s not, it\'s in [nixpkgs](https://github.com/NixOS/nixpkgs/blob/32c523914fdb8bf9cc7912b1eba023a8daaae2e8/pkgs/top-level/impure.nix#L28).

Я уверен, вы читали в wiki или других руководствах о `~/.config/nixpkgs/config.nix` (в прошлом `~/.nixpkgs/config.nix`), и я уверен, вы задавались вопросом, почему этот путь "вшит" в Nix.
На самом деле, он вшит не в Nix, а в [nixpkgs](https://github.com/NixOS/nixpkgs/blob/32c523914fdb8bf9cc7912b1eba023a8daaae2e8/pkgs/top-level/impure.nix#L28).

> The `all-packages.nix` expression accepts the `config` parameter.
> If it\'s `null`, then it reads the `NIXPKGS_CONFIG` environment variable.
> If not specified, `nixpkgs` will pick `$HOME/.config/nixpkgs/config.nix`.

Выражение `all-packages.nix` принимает параметр `config`.
Если там содержится `null, функция ищет переменную окружения `NIXPKGS_CONFIG`.
Если переменная не найдена, `nixpkgs` обращается к `$HOME/.config/nixpkgs/config.nix`.

> After determining `config.nix`, it will be imported as a nix expression, and that will be the value of `config` (in case it hasn\'t been passed as parameter to import `<nixpkgs>`).

При обнаружении `config.nix`, он будет импортирован как выражение Nix, которое и станет значением `config` (конечно, в том случае, если параметр не был явно передан при импорте `<nixpkgs>`).

> The `config` is available in the resulting repository:

Значение `config` доступно после импорта репозитория:

```bash
$ nix repl
nix-repl> pkgs = import <nixpkgs> {}
nix-repl> pkgs.config
{ }
nix-repl> pkgs = import <nixpkgs> { config = { foo = "bar"; }; }
nix-repl> pkgs.config
{ foo = "bar"; }
```

> What attributes go in `config` is a matter of convenience and conventions.

Какие атрибуты указывать в `config` — вопрос удобства и соглашений.

> For example, `config.allowUnfree` is an attribute that forbids building packages that have an unfree license by default.
> The `config.pulseaudio` setting tells whether to build packages with pulseaudio support or not where applicable and when the derivation obeys to the setting.

Например, `config.allowUnfree` — это атрибут, запрещающий сборку пакетов, чья лицензия по умолчанию не является свободной.
Настройка `config.pulseaudio` подсказыват, собирать ли пакеты с поддержкой PulseAudio, если это возможно и если деривация это умеет.

> ## About .nix functions

## О функциях .nix

> A `.nix` file contains a nix expression.
> Thus it can also be a function.
> I remind you that `nix-build` expects the expression to return a derivation.
> Therefore it\'s natural to return straight a derivation from a `.nix` file.
> However, it\'s also very natural for the `.nix` file to accept some parameters, in order to tweak the derivation being returned.

Файл `.nix` содержит выражение Nix.
Которое может быть функцией.
Я напоминаю, что `nix-build` ожидает, что выражение возвращает деривацию.
Так что, вполне естественно возвращать деривацию напрямую из файла `.nix`.
Кроме того, вполне естественно, что файл `.nix` принимает несколько параметров, помогающих настроить возвращаемую деривацию.

> In this case, nix does a trick:

Для таких случае в Nix есть трюк:

> -   If the expression is a derivation, build it.
> -   If the expression is a function, call it and build the resulting derivation.

- Если выражение является деривацией, собрать её.
- Если выражение является функцией, вызвать её и собрать деривацию, которую она вернула.

> For example you can nix-build the `.nix` file below:

Например, с помощью `nix-build` вы можете собрать такой файл `.nix`:

```nix
{ pkgs ? import <nixpkgs> {} }:

pkgs.psmisc
```

> Nix is able to call the function because the pkgs parameter has a default value.
> This allows you to pass a different value for pkgs using the `--arg` option.

Nix может вызывать функцию, поскольку у параметра `pkgs` есть значение по умолчанию.
Вы можете передать другое значение `pkgs`, используя опцию **--arg**.

> Does it work if you have a function returning a function that returns a derivation?
> No, Nix only calls the function it encounters once.

Работает ли эта схема, если вы получаете функцию, которая возвращает функцию, которая возвращает деривацию?
Нет, Nix вызывает найденную функцию только один раз.

> ## Conclusion

## Заключение

> We\'ve unleashed the `<nixpkgs>` repository.
> It\'s a function that accepts some parameters, and returns the set of all packages.
> Due to laziness, only the accessed derivations will be built.

Мы выяснили, что из себя представляет репозиторий `<nixpkgs>`.
Это функция, которая принимает несколько параметров и возвращает набор всех пакетов.
Из-за ленивых вычислений, собраны будут только те деривации, к которым вы обращались.

> You can use this repository to build your own packages as we\'ve seen in the previous pill when creating our own repository.

Вы можете использовать этот репозиторий, чтобы собирать собственные пакеты, как мы видели в предыдущей пилюле, создавая наш собственный репозиторий.

> Lately I\'m a little busy with the NixOS 14.11 release and other stuff, and I\'m also looking toward migrating from blogger to a more coder-oriented blogging platform.
> So sorry for the delayed and shorter pills :)

В последнее время я немного занят выпуском NixOS 14.11 и другими вещами, а также думаю о переходе с Blogger на другую блог-платформу, больше оринетированную на программистов.
Поэтому прошу прощения, что пишу редко и мало.


> ## Next pill

## В следующей пилюле

> \...we will talk about overriding packages in the `nixpkgs` repository.
> What if you want to change some options of a library and let all other packages pick the new library?
> One possibility is to use, like described above, the `config` parameter when applicable.
> The other possibility is to override derivations.

...мы поговорим о переопределении пакетов в репозитории `nixpkgs`.
Что, если вам бы хотелось изменить несколько параметров библиотеки и позволить другим пакетам использовать новую библиотеку?
Одним из способов является, когда это возможно, использование параметра `config`.
Другой способ — переопределение дериваций.