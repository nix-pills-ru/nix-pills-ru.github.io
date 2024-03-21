> # Install on Your Running System

# Установка в вашей системе

> Welcome to the second Nix pill.
> In the [first](#why-you-should-give-it-a-try) pill we briefly described Nix.

Настало время принимать вторую пилюлю Nix.
[Первая](01-why-you-should-give-it-try.md) кратко рассказала про Nix.

> Now we'll install Nix on our running system and understand what changed in our system after the installation.
> **If you're using NixOS, Nix is already installed; you can skip to the [next](#enter-environment)
pill.**

Теперь мы установим Nix на нашу систему и разбирёмся, что изменилось после установки.
**Если вы используте NixOS, Nix у вас уже установлен, так что мы можете пропустить [следующую](03-enter-environment.md) пилюлю.

> For installation instructions, please refer to the Nix Reference Manual on [Installing
Nix](https://nixos.org/manual/nix/stable/installation/installation.html).

За инструкциями по установки, пожалуйста, обращайтесь к разделу [Установка Nix](https://nixos.org/manual/nix/stable/installation/installation.html) Справочного руководства Nix.

> ## Installation

## Установка

> These articles are not a tutorial on *using* Nix.
> Instead, we're goingto walk through the Nix system to understand the fundamentals.

Эти статьи не являются руководством по *использованию* Nix.
Вместо этого, мы собираемся прогулялся по системе Nix, чтобы понять, как она устроена.

>The first thing to note: derivations in the Nix store refer to other derivations which are themselves in the Nix store.
>They don't use `libc` from our system or anywhere else.
>It's a self-contained store of all the software we need to bootstrap up to any particular package.

Первая важная вещь: порождения в хранилище Nix ссылаются на другие порождения, которые также находятся в хранилище Nix.
Они не используют `libc` из нашей системы или откуда-то ещё.
В хранилище находятся все необходимые библиотеки, которые могут потребоваться, чтобы запустить любой конкретный пакет.

<div class="note">

>In a multi-user installation, such as the one used in NixOS, the store is owned by root and multiple users >can install and build software through a Nix daemon.
>You can read more about multi-user installations here: <https://nixos.org/manual/nix/stable/installation/>installing-binary.html#multi-user-installation>.

При многопользовательской установке (такая как раз применяется в NixOS), хранилищем владеет root, а многочисленные пользователи могут устанавливать или собирать софт при помощи демона Nix.
Больше о многопользовательской установке вы можете прочитать здесь: <https://nixos.org/manual/nix/stable/installation/>installing-binary.html#multi-user-installation>.

</div>

> ## The beginnings of the Nix store

## С чего начинается хранилище в Nix

> Start looking at the output of the install command:

Взглянем, что печатает команда установки:

```text
    copying Nix to /nix/store..........................
```

> That's the `/nix/store` we were talking about in the first article.
> We're copying in the necessary software to bootstrap a Nix system.
> You can see bash, coreutils, the C compiler toolchain, perl libraries, sqlite and Nix itself with its own  tools and libnix.

Речь идёт о каталоге `/nix/store`, который мы обсуждали в первой статье.
Туда мы копируем всё, что необходимо для запуска системы Nix.
Вы можете заметить bash, утилиты ядра, компилятор C, библиотеки Perl, sqlite и сам Nix с его утилитами и libnix.

> You may have noticed that `/nix/store` can contain not only directories, but also files, still always in the form \<hash-name>.

Возможно, вы заметили, что `/nix/store` может содержать не только каталоги, но и файлы, с тем же форматом имени \<hash-name>.

> ## The Nix database

## База данных Nix

> Right after copying the store, the installation process initializes a database:

Сразу после наполнения хранилища, процесс установки инициализирует базу данных:

```text
    initialising Nix database...
```

> Yes, Nix also has a database.
> It's stored under `/nix/var/nix/db`.
> It is a sqlite database that keeps track of the dependencies between derivations.

Да, в Nix есть база данных.
Она находится в `/nix/var/nix/db`.
Это база данных sqlite, куда записываются записываются зависимости между порождениями.

> The schema is very simple: there's a table of valid paths, mapping from an auto increment integer to a store path.

Схема очень простая: есть таблица корректных путей, где каждому пути сопоставлено целое автоинкрементное число.

> Then there's a dependency relation from path to paths upon which they depend.

Далее, есть таблица зависимостей, где записано, от каких путей зависит каждый новый путь.

> You can inspect the database by installing sqlite (`nix-env -iA sqlite -f '<nixpkgs>'`) and then running
`sqlite3 /nix/var/nix/db/db.sqlite`.

Вы можете исследовать эту базу, установив sqlite (`nix-env -iA sqlite -f '<nixpkgs>'`) и выполнив команду `sqlite3 /nix/var/nix/db/db.sqlite`.

<div class="note">

> If this is the first time you're using Nix after the initial installation, remember you must close and open your terminals first, so that your shell environment will be updated.

Сразу после установки Nix не забудьте закрыть и заново открыть терминалы, чтобы обновить настройки оболочки.

</div>

<div class="important">

> Never change `/nix/store` manually.
> If you do, then it will no longer be in sync with the sqlite db, unless you *really* know what you are doing.

Никогда не изменяйте `/nix/store` вручную.
Иначе хранилище больше не будет синхронизировано в базой данных sqlite, если только вы *на самом деле* не знаете, что делаете.

</div>

> ## The first profile

## Первый профиль

> Next in the installation, we encounter the concept of the [profile](https://nixos.org/manual/nix/stable/package-management/profiles.html):

```text
creating /home/nix/.nix-profile
installing 'nix-2.1.3'
building path(s) `/nix/store/a7p1w3z2h8pl00ywvw6icr3g5l9vm5r7-user-environment'
created 7 symlinks in user environment
```

Завершив установку, давайте познакомимся с понятием [профиля](https://nixos.org/manual/nix/stable/package-management/profiles.html):

> A profile in Nix is a general and convenient concept for realizing rollbacks.
> Profiles are used to compose components that are spread among multiple paths under a new unified path.
> Not only that, but profiles are made up of multiple "generations": they are versioned.
> Whenever you change a profile, a new generation is created.

Профиль в Nix — общая и удобная концепция для реализации отката.
Профили используются для объединения компонентов, размещённых по разным путям, в один путь.
Более того, профили могут состоятить из нескольких "поколений": они версионируются.
При изменении профиля появляется новое поколение.

> Generations can be switched and rolled back atomically, which makes them convenient for managing changes to your system.

Поколения можно переключать и отктывать атомарно, что делает управление изменениями в вашей системе особенно удобным.

> Let's take a closer look at our profile:

```bash
$ ls -l ~/.nix-profile/
bin -> /nix/store/ig31y9gfpp8pf3szdd7d4sf29zr7igbr-nix-2.1.3/bin
[...]
manifest.nix -> /nix/store/q8b5238akq07lj9gfb3qb5ycq4dxxiwm-env-manifest.nix
[...]
share -> /nix/store/ig31y9gfpp8pf3szdd7d4sf29zr7igbr-nix-2.1.3/share
```

Рассмотрим наш профиль подробнее:

> That nix-2.1.3 derivation in the Nix store is Nix itself, with binaries and libraries.
> The process of "installing" the derivation in the profile basically reproduces the hierarchy of the nix-2.1.3 store derivation in the profile by means of symbolic links.

Порождение nix-2.1.3 в хранилище Nix — это сам Nix вместе с исполняемыми файлами и библиотеками.
Процесс "установки" порождения в профиль в сущности воспроизводит дерево nix-2.1.3 из хранилища в профиле с помощью символических ссылок.

> The contents of this profile are special, because only one program has been installed in our profile, therefore e.g. the `bin` directory points to the only program which has been installed (Nix itself).

Содержимое профиля особенное, поскольку только одна программа установлена в наш профиль, таким образом, каталог `bin` указывает на единственную программу, которая может быть установлена (сам Nix).

> But that's only the contents of the latest generation of our profile.
> In fact, `~/.nix-profile` itself is a symbolic link to `/nix/var/nix/profiles/default`.

Но это содержимое — всего лишь последнее поколение нашего профиля.
Фактически, `~/.nix-profile` — это символическая ссылка на `/nix/var/nix/profiles/default`.

> In turn, that's a symlink to `default-1-link` in the same directory.
> Yes, that means it's the first generation of the `default` profile.

Которая, в свою очередь тоже является символической ссылкой на `default-1-link` в том же каталоге.
Да, это значит, что речь идёт о первом покололении профиля `default`.

> Finally, `default-1-link` is a symlink to the nix store "user-environment" derivation that you saw printed during the installation process.

В конечном итоге, `default-1-link` — это символическая ссылка на порождение "user-environment", которое печаталось на экране в процесс установки.

> We'll talk about `manifest.nix` more in the next article.

Мы подробнее поговорим о `manifest.nix` в следующей статье.

> ## Nixpkgs expressions

## Выражения Nixpkgs

> More output from the installer:

Больше вывода от установщика:

```text
downloading Nix expressions from `http://releases.nixos.org/nixpkgs/nixpkgs-14.10pre46060.a1a2851/nixexprs.tar.xz'...
unpacking channels...
created 2 symlinks in user environment
modifying /home/nix/.profile...
```

> Nix expressions are written in the [Nix language](https://nix.dev/tutorials/nix-language) and used to describe packages and how to build them.
> [Nixpkgs](https://nixos.org/nixpkgs/) is the repository containing all of the expressions:
<https://github.com/NixOS/nixpkgs>.

Выражения Nix написаны на [языке Nix](https://nix.dev/tutorials/nix-language), они описывают пакеты и то, как их собирать.
[Nixpkgs](https://nixos.org/nixpkgs/) — это репозиторий, содержащий все выражения: <https://github.com/NixOS/nixpkgs>.

> The installer downloaded the package descriptions from commit `a1a2851`.

Установщик скачал описания пакетов, начиная с коммита `a1a2851`.

> The second profile we discover is the channels profile.
> `~/.nix-defexpr/channels` points to `/nix/var/nix/profiles/per-user/nix/channels` which points to
`channels-1-link` which points to a Nix store directory containing the downloaded Nix expressions.

Второй профиль, с которым мы столкнулись — это профиль каналов.
`~/.nix-defexpr/channels` ссылается на `/nix/var/nix/profiles/per-user/nix/channels`, который в свою очередь ссылается на `channels-1-link`, который ссылается на каталог в хранилище со скачанными выражениями Nix.

> Channels are a set of packages and expressions available for download.
> Similar to Debian stable and unstable, there's a stable and unstable channel.
> In this installation, we're tracking `nixpkgs-unstable`.

Каналы — это множество пакетов и выражений, доступных для скачивания.
Подобно стабильным и нестабильным репозиториям в Debian, в Nix есть есть стабильные и нестабильные каналы.

> Don't worry about Nix expressions yet, we'll get to them later.

Пока не беспокойтесь о выражениях Nix, мы вернёмся к ним позже.

> Finally, for your convenience, the installer modified `~/.profile` to automatically enter the Nix environment.
> What `~/.nix-profile/etc/profile.d/nix.sh` really does is simply to add `~/.nix-profile/bin` to `PATH` and `~/.nix-defexpr/channels/nixpkgs` to `NIX_PATH`.
> We'll discuss `NIX_PATH` later.

В конечном итоге для вашего удобства установщик изменил `~/.profile`, чтобы вы автоматически попадали в окружение Nix.
Что делает `~/.nix-profile/etc/profile.d/nix.sh` на самом деле — просто добавляет `~/.nix-profile/bin` в `PATH` и `~/.nix-defexpr/channels/nixpkgs` в `NIX_PATH`.
Переменную `NIX_PATH` мы обсудим позже.

> Read `nix.sh`, it's short.

Прочитайте `nix.sh`, он короткий.

> ## FAQ: Can I change /nix to something else?

## Вопросы и ответы: можно ли заменить /nix на что-то другое?

> You can, but there's a good reason to keep using `/nix` instead of a different directory.
> All the derivations depend on other derivations by using absolute paths.
> We saw in the first article that bash referenced a glibc under a specific absolute path in `/nix/store`.

Да, можно, но есть веская причина использовать именно каталог `/nix` вместо любого другого.
Все порождения завият от других порождений, используя при этом абсолютные пути.
В первой статье мы видели, что bash ссылается на glibc по конкретному абсолютному пути внутри `/nix/store`.

> You can see for yourself, don't worry if you see multiple bash derivations:

Убедитесь сами, и не волнуйтесь, если увидите множество порождений bash:

```bash
$ ldd /nix/store/*bash*/bin/bash
[...]
```

> Keeping the store in `/nix` means we can grab the binary cache from nixos.org (just like you grab packages from debian mirrors) otherwise:

Размещая хранилище в `/nix`, мы можем напрямую использовать бинарные образы с nixos.org (точно также, как и пакеты с зеркал Debian), а в противном случае:

> - glibc would be installed under `/foo/store`
> - Thus bash would need to point to glibc under `/foo/store`, instead of under `/nix/store`
> - So the binary cache can't help, because we need a *different* bash, and so we'd have to recompile everything ourselves.

- glibc будет установлен в `/foo/store`
- После этого bash будет указывать на glibc в `/foo/store` вместо `/nix/store`
- В результате мы не сможем использовать бинарный образ, так как нам нужен *другой* bash, и мы вынуждены перекомпилировать вообще всё.

> After all `/nix` is a sensible place for the store.

Помимо прочег, `/nix` — осмысленное место для хранилища.

> ## Conclusion

## Заключение

> We've installed Nix on our system, fully isolated and owned by the `nix` user as we're still coming to terms with this new system.

Мы установили Nix в нашу систему, он полностью изолирован и принадлежит пользователю `nix`, и мы продолжаем знакомиться с особенностями новой системы.

> We learned some new concepts like profiles and channels.
> In particular, with profiles we're able to manage multiple generations of a composition of packages, while with channels we're able to download binaries from `nixos.org`.

Мы познакомились с новыми концепциями, такими как профили и каналы.
В частности, с помощью профилей мы научились управлять поколениями составных пакетов, а с помощью каналов — загружать бинарные образы с `nixos.org`.

> The installation put everything under `/nix`, and some symlinks in the Nix user home.
> That's because every user is able to install and use software in her own environment.

Установщик помещает всё в каталог `/nix`, создавая несколько символических ссылок в домашнем каталоге пользователя Nix.

> I hope I left nothing uncovered so that you think there's some kind of magic going on behind the scenes.
> It's all about putting components in the store and symlinking these components together.

Я надеюсь, что ничего не оставил необъяснённым, чтобы у вас не возникло ощущения какого-то волшебства за кулисами.
Всё сводится к тому, что компоненты находятся в хранилище и ссылаются друг на друга посредством символических ссылок.

> ## Next pill...

## В следующпей пилюле...

> ...we will enter the Nix environment and learn how to interact with the store.

Мы погрузимся в окружение Nix и научимся взаимодействовать с хранилищем.