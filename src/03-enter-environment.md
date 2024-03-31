> # Enter the Environment

# Погружаемся в среду

> Welcome to the third Nix pill. In the [second pill](#install-on-your-running-system) we installed Nix on our running system.
> Now we can finally play with it a little, these things also apply to NixOS users.

Добро пожаловать в третью Nix-пилюлю. Во [второй пилюле](02-install-on-your-running.md) мы установили Nix на свою систему.
Сейчас мы, наконец, можем немного с ним поиграться, и эти инструкции также подходят для пользователей NixOS.

> ## Enter the environment

## Погружаемся в среду

> **If you're using NixOS, you can skip to the [next](#install-something) step.**

**Если вы используете NixOS, вы можете пропустить [следующий](#install-something) шаг.

> In the previous article we created a Nix user, so let's start by switching to it with `su - nix`.
> If your `~/.profile` got evaluated, then you should now be able to run commands like `nix-env` and
`nix-store`.

В предыдущей статье мы создали пользователя Nix, так что давайте начнём, переключившись на него с помощью команды `su - nix`.
Если ваш `~/.profile` уже отработал, вам должны быть доступны команды наподобие `nix-env` и `nix-store`.

> If that's not the case:

Если же нет:

```text
$ source ~/.nix-profile/etc/profile.d/nix.sh
```

> To remind you, `~/.nix-profile/etc` points to the `nix-2.1.3` derivation.
> At this point, we are in our Nix user profile.

Напоминаю, что `~/.nix-profile/etc` указывает на порождение `nix-2.1.3`.

> ## Install something

## Устанавливаем что-нибудь

> Finally something practical!
> Installation into the Nix environment is an interesting process.
> Let's install `hello`, a simple CLI tool which prints `Hello world` and is mainly used to test compilers and package installations.

Конечно, что-нибудь практическое!
Установка в окружение Nix — интересный процесс.
Давайте установим `hello` — простую командную утилиту, которая печатает `Hello world` и обычно используется для тестирования компиляторов и установки пакетов.

> Back to the installation:

Возвращаемся к установке:

```text
$ nix-env -i hello
installing 'hello-2.10'
[...]
building '/nix/store/0vqw0ssmh6y5zj48yg34gc6macr883xk-user-environment.drv'...
created 36 symlinks in user environment
```

> Now you can run `hello`.
> Things to notice:

Теперь вы можете запустить `hello`.
Вот на что нужно обратить внимание:

> - We installed software as a user, and only for the Nix user.
> - It created a new user environment.
>   That's a new generation of our Nix user profile.
> - The [nix-env](https://nixos.org/manual/nix/stable/command-ref/nix-env.html) tool manages environments, profiles and their generations.
> - We installed `hello` by derivation name minus the version.
>   I repeat: we specified the **derivation name** (minus the version) to install it.

- Мы установили программу с правами пользователя, и только для пользователя Nix.
- Из-за этого появилось новое окружение (среда) пользователя.
  Это новое поколение профиля нашего пользователя Nix.
- Утилита [nix-env](https://nixos.org/manual/nix/stable/command-ref/nix-env.html) управляет окружениями, профилями и их поколенями.
- Мы установили `hello` по имени порождения без указания версии.
  Повторяю: мы указали только **имя порождения** (без версии) для установки.

> We can list generations without walking through the `/nix` hierarchy:

Мы можем просмотреть список поколений без блужданий по каталогу `/nix`:

```text
$ nix-env --list-generations
   1   2014-07-24 09:23:30
   2   2014-07-25 08:45:01   (current)
```

> Listing installed derivations:

Просмотреть установленные порождения:

```text
$ nix-env -q
nix-2.1.3
hello-2.10
```

> So, where did `hello` really get installed? `which hello` is `~/.nix-profile/bin/hello` which points to the store.
> We can also list the derivation paths with `nix-env -q --out-path`.
> So that's what those derivation paths are called: the **output** of a build.

Так что, мы действительно установили `hello`? `which hello` показывает `~/.nix-profile/bin/hello`, который указывает на хранилище.
Мы также можем просмотреть пути порождения с помощью команды `nix-env -q --out-path`.
Эти пути порождения называются **выходом** сборки.

> ## Path merging

## Слияние путей

> At this point you probably want to run `man` to get some documentation.
> Even if you already have man system-wide outside of the Nix environment, you can install and use it within Nix with `nix-env -i man-db`.
> As usual, a new generation will be created, and `~/.nix-profile` will point to it.

Сейчас вы, возможно, хотите запустить `man` чтобы получить кое-какую информацию.
Даже если у вас уже есть утилита `man` в вашей системе, вы можете установить её в окружение Nix с помощью команды `nix-env -i man-db`.

> Let's inspect the [profile](https://nixos.org/manual/nix/stable/package-management/profiles.html) a bit:

Давайте немного исследуем [профиль](https://nixos.org/manual/nix/stable/package-management/profiles.html):

```text
$ ls -l ~/.nix-profile/
dr-xr-xr-x 2 nix nix 4096 Jan  1  1970 bin
lrwxrwxrwx 1 nix nix   55 Jan  1  1970 etc -> /nix/store/ig31y9gfpp8pf3szdd7d4sf29zr7igbr-nix-2.1.3/etc
[...]
```

> Now that's interesting.
> When only `nix-2.1.3` was installed, `bin` was a symlink to `nix-2.1.3`.
> Now that we've actually installed some things (`man`, `hello`), it's a real directory, not a symlink.

Вот кое-что интересное.
Когда было установлено только порождение `nix-2.1.3`, `bin` был симлинком на `nix-2.1.3`.
Теперь, когда мы дополнительно установили несколько программ (`man,` `hello`), он стал реальным каталогом, не симлинком.

```text
$ ls -l ~/.nix-profile/bin/
[...]
man -> /nix/store/83cn9ing5sc6644h50dqzzfxcs07r2jn-man-1.6g/bin/man
[...]
nix-env -> /nix/store/ig31y9gfpp8pf3szdd7d4sf29zr7igbr-nix-2.1.3/bin/nix-env
[...]
hello -> /nix/store/58r35bqb4f3lxbnbabq718svq9i2pda3-hello-2.10/bin/hello
[...]
```

> Okay, that's clearer now. `nix-env` merged the paths from the installed derivations.
> `which man` points to the Nix profile, rather than the system `man`, because `~/.nix-profile/bin` is at the head of `$PATH`.

Хорошо, кое-что стало проясняться. `nix-env` слил пути из установленных порождений.
`which man` указывает на профиль Nix, вместо того, чтобы указывать на системный `max`, потому что `~/.nix-profile/bin` находится в начале `$PATH`.

> ## Rolling back and switching generation

## Откат и переключение поколений

> The last command installed `man`.
> We should be at generation 3, unless you changed something in the middle.
> Let's say we want to rollback to the old generation:

Последняя установленная команда это `man`.
Мы должны быть в третьем поколении, если вы ничего не меняли на предыдущих шагах.
Представим, что нам надо откатиться к предыдущему поколению:

```text
$ nix-env --rollback
switching from generation 3 to 2
```

> Now `nix-env -q` does not list `man` anymore.
> `` ls -l `which man` `` should now be your system copy.

Теперь `nix-env -q` не показывет `man`.
`` ls -l `which man` `` должен вывести содержимое каталога основной системы.

> Enough with the rollback, let's go back to the most recent generation:

Откатившись, давайте вернёмся назад к поколению 3:

```text
$ nix-env -G 3
switching from generation 2 to 3
```

> I invite you to read the manpage of `nix-env`.
> `nix-env` requires an operation to perform, then there are common options for all operations, as well as options specific to each operation.

Я приглашаю вас ознакомиться со справкой на `nix-env`.
Для запуска `nix-env` необходимо указать операцию, есть опции, общие для всех операций, и такие, которые специфичны для отдельных операций.

> You can of course also [uninstall](https://nixos.org/manual/nix/stable/command-ref/nix-env.html#operation---uninstall) and [upgrade](https://nixos.org/manual/nix/stable/command-ref/nix-env.html#operation---upgrade) packages.

Конечно, вы можете [удалить](https://nixos.org/manual/nix/stable/command-ref/nix-env.html#operation---uninstall) и [обновить](https://nixos.org/manual/nix/stable/command-ref/nix-env.html#operation---upgrade) пакеты.

> ## Querying the store

## Запросы в хранилище

> So far we learned how to query and manipulate the environment.
> But all of the environment components point to the store.

К текущему моменту мы научились исследовать и изменять окружение.
Однако все компоненты окружения указывают на хранилище.

> To query and manipulate the store, there's the `nix-store` command.
> We can do some interesting things, but we'll only see some queries for now.

Чтобы исследовать и изменять хранилище, есть команда `nix-store`.
Нам доступны разные интересные штуки, но пока мы рассмотрим только несколько запросов.

> To show the direct runtime dependencies of `hello`:

Просмотреть непосредственные зависимости программы `hello`:

```text
$ nix-store -q --references `which hello`
/nix/store/fg4yq8i8wd08xg3fy58l6q73cjy8hjr2-glibc-2.27
/nix/store/58r35bqb4f3lxbnbabq718svq9i2pda3-hello-2.10
```

> The argument to `nix-store` can be anything as long as it points to the Nix store.
> It will follow symlinks.

Аргумент `nix-store` может быть любым, если он указывает на хранилище Nix.
Утилита будет следовать симлинкам.

> It may not make sense to you right now, but let's print reverse dependencies of `hello`:

Сейчас это может показаться не важным, но давайте просмотрим порождения, зависящие от `hello`:

```text
$ nix-store -q --referrers `which hello`
/nix/store/58r35bqb4f3lxbnbabq718svq9i2pda3-hello-2.10
/nix/store/fhvy2550cpmjgcjcx5rzz328i0kfv3z3-env-manifest.nix
/nix/store/yzdk0xvr0b8dcwhi2nns6d75k2ha5208-env-manifest.nix
/nix/store/mp987abm20c70pl8p31ljw1r5by4xwfw-user-environment
/nix/store/ppr3qbq7fk2m2pa49i2z3i32cvfhsv7p-user-environment
```

> Was it what you expected?
> It turns out that our environments depend upon `hello`.
> Yes, that means that the environments are in the store, and since they contain symlinks to `hello`, therefore the environment depends upon `hello`.

Это было то, что вы ожидали?
Оказывается, наши окружения зависят от `hello`.
Да, это значит, что окружения также находятся в хранилище, и поскольку они содержат симлинки на `hello`, каждое окружение зависит от `hello`.

> Two environments were listed, generation 2 and generation 3, since these are the ones that had `hello` installed in them.

Мы видим в списке два окужения, относящиеся к поколениям 2 и 3, так как именно они содержат установленную программу `hello`.

> The `manifest.nix` file contains metadata about the environment, such as which derivations are installed.
> So that `nix-env` can list, upgrade or remove them.
> And yet again, the current `manifest.nix` can be found at `~/.nix-profile/manifest.nix`.

Файл `manifest.nix` содержит метаданные, относящиеся к окружению, например, какие порождения установлены.
Поэтому `nix-env` может выводить, обновлять и удалять их.
И снова — текущий `manifest.nix` находится по пути `~/.nix-profile/manifest.nix`.

> ## Closures

## Замыкания

> The closures of a derivation is a list of all its dependencies, recursively, including absolutely everything necessary to use that derivation.

Замыкание порождения — это рекурсивный список всех его зависимостей, который включает абсолютно всё, что нужно для использования данного порождения.

```text
$ nix-store -qR `which man`
[...]
```

> Copying all those derivations to the Nix store of another machine makes you able to run `man` out of the box on that other machine.
> That's the base of deployment using Nix, and you can already foresee the potential when deploying software in the cloud (hint: `nix-copy-closures` and `nix-store --export`).

Копирование всех этих порождений в хранилище Nix на другой машине позволяет запускать там утилиту `man` прямо из коробки.
Это и есть основа развёртывания с помощю Nix, и вы уже можете предвидеть потенциал такого развёртывания софта в облака.

> A nicer view of the closure:

Просмотр замыкания в удобном виде:

```text
$ nix-store -q --tree `which man`
[...]
```

> With the above command, you can find out exactly why a *runtime* dependency, be it direct or indirect, exists for a given derivation.

С помощью приведённой выше команды вы можете точно выяснитиь, почему у данного порождения существует зависимость времени выполнения, прямая или косвенная.

> The same applies to environments.
> As an exercise, run `nix-store -q --tree ~/.nix-profile`, and see that the first children are direct dependencies of the user environment: the installed derivations, and the `manifest.nix`.

То же самое относится и окружениям.

> ## Dependency resolution

## Разрешение зависимостей

> There isn't anything like `apt` which solves a SAT problem in order to satisfy dependencies with lower and upper bounds on versions.
> There's no need for this because all the dependencies are static: if a derivation X depends on a derivation Y, then it always depends on it.
> A version of X which depended on Z would be a different derivation.

В Nix нет ничего похожего на `apt` которая решает задачу выполнимости булевых формул (SAT problem), чтобы подобрать зависимости с учётом верхних и нижних границ версий.
Это не требуется, потому что все зависимости статичны: если порождение X зависиот от порождения Y, оно зависит от него всегда.
Порождение X, которое зависит от Z, должно быть другим порождением.

> ## Recovering the hard way

## Восстановление вручную

```text
$ nix-env -e '*'
uninstalling 'hello-2.10'
uninstalling 'nix-2.1.3'
[...]
```

> Oops, that uninstalled all derivations from the environment, including Nix.
> That means we can't even run `nix-env`, what now?

Ой, команда удалила все порождения из окружения, включая Nix.
Это значит, что мы не можем даже запустить `nix-env`, и что делать в этом случае?

> Previously we got `nix-env` from the environment.
> Environments are a convenience for the user, but Nix is still there in the store!

Раньше мы получали доступ к `nix-env` из окружения.
Окружения удобны для пользователей, но Nix всё ещё находится в хранилище!

> First, pick one `nix-2.1.3` derivation: `ls /nix/store/*nix-2.1.3`, say `/nix/store/ig31y9gfpp8pf3szdd7d4sf29zr7igbr-nix-2.1.3`.

Сначала, выберем одно из порождений `nix-2.1.3`: `ls /nix/store/*nix-2.1.3`, пусть это будет `/nix/store/ig31y9gfpp8pf3szdd7d4sf29zr7igbr-nix-2.1.3`.

> The first option is to rollback:

Первый способ — это откат:

```text
$ /nix/store/ig31y9gfpp8pf3szdd7d4sf29zr7igbr-nix-2.1.3/bin/nix-env --rollback
```

> The second option is to install Nix, thus creating a new generation:

Второй способ — установить Nix, тем самым создав новое поколение:

```text
$ /nix/store/ig31y9gfpp8pf3szdd7d4sf29zr7igbr-nix-2.1.3/bin/nix-env -i /nix/store/ig31y9gfpp8pf3szdd7d4sf29zr7igbr-nix-2.1.3/bin/nix-env
```

> ## Channels

## Каналы

> So where are we getting packages from?
> We said something about this already in the [second article](#install-on-your-running-system).
> There's a list of channels from which we get packages, although usually we use a single channel.
> The tool to manage channels is [nix-channel](https://nixos.org/manual/nix/stable/command-ref/nix-channel.html).

Ну, а откуда мы берём пакеты?
Мы уже говорили об этом во [второй статье](02-install-on-your-running-system.md).
Существует список каналов, откуда мы получаем пакеты, хотя обычно мы используем только один канал.
Есть утилита для управления каналами — [nix-channel](https://nixos.org/manual/nix/stable/command-ref/nix-channel.html).

```text
$ nix-channel --list
nixpkgs http://nixos.org/channels/nixpkgs-unstable
```

> If you're using NixOS, you may not see any output from the above command (if you're using the default), or you may see a channel whose name begins with "nixos-" instead of "nixpkgs".

Если вы используете NixOS, вы можете не увидеть вывода этой команды (если используете настройки по умолчанию), либо вы можете увидеть канал, чьё имя начинается с "nixos-" вместо "nixpkgs".

> That's essentially the contents of `~/.nix-channels`.

По сути, это содержимое файла `~/.nix-channels`.

> > [!NOTE]
> > `~/.nix-channels` is not a symlink to the nix store!

> [!NOTE]
> `~/.nix-channels` — это не символическая ссылка на хранилище!

> To update the channel run `nix-channel --update`.
> That will download the new Nix expressions (descriptions of the packages), create a new generation of the channels profile and unpack it under `~/.nix-defexpr/channels`.

Для обновления канала запустите `nix-channel --update`.
Эта команда скачает новые выражения Nix (описания пакетов), создаст новое поколнение профиля каналов и распакует его в `~/.nix-defexpr/channels`.

> This is quite similar to `apt-get update`.
> (See [this table](https://nixos.wiki/wiki/Cheatsheet) for a rough mapping between Ubuntu and NixOS package management.)

Она немного похожа на `apt-get update`.
(См. [таблицу](https://nixos.wiki/wiki/Cheatsheet), где достаточно грубо сравниваются средства управления пакетами Ubuntu и NixOS).

> ## Conclusion

## Заключение

> We learned how to query the user environment and to manipulate it by installing and uninstalling software.
> Upgrading software is also straightforward, as you can read in [the manual](https://nixos.org/manual/nix/stable/command-ref/nix-env.html#operation---upgrade) (`nix-env -u` will upgrade all packages in the environment).

Мы узнали, как исследовать окружение пользователя и изменять его, устанавливая и удаляя программы.
Обновление программ тоже просто, как вы можете прочитать в [руководстве](https://nixos.org/manual/nix/stable/command-ref/nix-env.html#operation---upgrade) (`nix-env -u` обновит все пакеты в окружении).

> Every time we change the environment, a new generation is created.
> Switching between generations is easy and immediate.

Всякий раз, когда мы меняем среду, создаётся новое поколение.
Переключение между поколениями выполняется просто и мгновенно.

> Then we learned how to query the store.
> We inspected the dependencies and reverse dependencies of store paths.

Затем мы узнали как исследовать хранилище.
Мы изучили зависимости и обратные зависимости путей в хранилище.

> We saw how symlinks are used to compose paths from the Nix store, a useful trick.

Вы увидели, как силлинки используются для слияния путей из хранилища Nix, полезная уловка.

> A quick analogy with programming languages: you have the heap with all the objects, that corresponds to the Nix store.
> You have objects that point to other objects, those correspond to derivations.
> This is a suggestive metaphor, but will it be the right path?

Быстрая аналогия с языками программирования: у вас есть куча со объектами, что соответствует хранилищу Nix.
У вас есть обекты, которые указывают на другие объекты, что соответствует порождениям.
Это всего лишь наводящая метафора, но поможет ли она нам двигаться в правильном направлении?

> ## Next pill...

## В следующая пилюле...

> ...we will learn the basics of the Nix language.
> The Nix language is used to describe how to build derivations, and it's the basis for everything else, including NixOS.
> Therefore it's very important to understand both the syntax and the semantics of the language.

...мы изучим основы языка Nix.
Язык Nix используется для описания того, как строить порождения, и он является основой для всего остального, включая NixOS.
Таким образом, очень важно понимать как синтаксис, так и семантику языка.
