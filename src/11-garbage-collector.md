> # The Garbage Collector {#garbage-collector}

# Сборщик мусора

> Welcome to the 11th Nix pill.
> In the previous [10th pill](#developing-with-nix-shell), we drew a parallel between the isolated build environment provided by `nix-build` and the isolated development shell provided by `nix-shell`.
> Using `nix-shell` allowed us to debug, modify, and manually build software using an environment that is almost identical to the one provided by `nix-build`.

Добро пожаловать на одиннадцатую пилюлю Nix.
В предыдущей [десятой пилюле](10-developing-with-nix-shell.md), мы провели параллель между изолированной средой сборки, предоставлямой `nix-build` и изолированной оболочкой разработки, предоставляемой `nix-shell`.

> Today, we will stop focusing on packaging and instead look at a critical component of Nix: the garbage collector.
> When we use Nix tools, we are often building derivations.
> This includes `.drv` files as well as out paths.
> These artifacts go in the Nix store and take up space in our storage.
> Eventually we may wish to free up some space by removing derivations we no longer need.
> This is the focus of the 11th pill.
> By default, Nix takes a relatively conservative approach when automatically deciding which derivations are \"needed\".
> In this pill, we will also see a technique to conduct more destructive upgrade and deletion operations.

Сегодня мы перестанем фокусироваться на создании пакетов и вместо этого взгляем на критический компонент Nix: сборщик мусора.
Когда мы используем инструменты Nix, мы часто строим деривации.
Это включает файлы `.drv` так же, как и выходные пути.
Эти артефакты попадают в хранилище Nix и занимают место на нашем носителе.
Так что мы можем захотеть освободить немного места, удалив деривации, которые нам больше не нужны.
На этом вопросе мы сфокусируемся в 11-й пилюле.
По умолчанию, Nix придерживается консервативной стратегии, когда автоматически решает, какие из дериваций являются "нужными".
В этой пилюле мы также познакомимся с техникой выполнения более деструктивных операций обновления и удаления.

> ## How does garbage collection work?

## Как работает сборщик мусора?

> Programming languages with garbage collectors use the concept of a set of \"garbage collector (or \'GC\') roots\" to keep track of \"live\" objects.
> A GC root is an object that is always considered \"live\" (unless explicitly removed as GC root).
> The garbage collection process starts from the GC roots and proceeds by recursively marking object references as \"live\".
> All other objects can be collected and deleted.

Языки программирования со сборщиком мусора используют концепцию множества корневых элементов (корней сборщика мусора или СМ-корней) для отслеживания "живых" объектов.
СМ-корень — это объект, который всегда считается "живым" (пока не будет явным образом удалён из списка СМ-корней).
Процесс сборки мусора начинается с СМ-корней, и рекурсивно помечает объекты, докоторых может добраться, как "живые".
Все остальные объекты могут быть собраны и удалены.


> Instead of objects, Nix\'s garbage collection operates on store paths, [with the GC roots themselves being store paths](https://nixos.org/manual/nix/stable/package-management/garbage-collector-roots.html).
> This approach is much more principled than traditional package managers such as `dpkg` or `rpm`, which may leave around unused packages or dangling files.

Вместо объектов, сборщик мусора Nix оперирует путями в хранилище, [причём СМ-корни также являются путями в хранилище](https://nixos.org/manual/nix/stable/package-management/garbage-collector-roots.html).
Этот подход гораздо более принципиален, чем у традиционных менеджеров пакетов, таких как `dpkg` или `rpm`, которые могут оставлять неиспользуемые пакеты или висящие файлы.

> The implementation is very simple and transparent to the user.
> The primary GC roots are stored under `/nix/var/nix/gcroots`.
> If there is a symlink to a store path, then the linked store path is a GC root.

Реализация очень проста и прозрачна для пользователя.
Первичные СМ-корни хранятся в `/nix/var/nix/gcroots`.
Если существует символическая ссылка на путь в хранилище, то связанный путь в хранилище является корнем СМ.

> Nix allows this directory to have subdirectories: it will simply recursively traverse the subdirectories in search of symlinks to store paths.
> When a symlink is encountered, its target is added to the list of live store paths.

Nix позволяет этому каталогу иметь подкаталоги: он просто рекурсивно перебирает подкаталоги в поиске символических ссылок на пути в хранилище.
При обнаружении символической ссылки, её цель добавляется в список живых путей к хранилищу.

> In summary, Nix maintains a list of GC roots.
> These roots can then be used to compute a list of all live store paths.
> Any other store paths are considered dead.
> Deleting these paths is now straightforward.
> Nix first moves dead store paths to `/nix/store/trash`, which is an atomic operation.
> Afterwards, the trash is emptied.

Подводя итог, можно сказать, что Nix ведёт список корней СМ.
Эти корни могут быть использованы для вычисления списка всех живых путей в хранилище.
Любой другой путь в хранилище считается мёртвым.
Удалить эти путь сейчас просто.
Сначала Nix переносит мёртвые пути в каталог `/nix/store/trash`, что является атомарной операцией.
После этого корзица чистится.

> ## Playing with the GC

## Эксмериментируем со сборщиком мусора

> Before we begin we first run the [nix garbage collector](https://nixos.org/manual/nix/stable/command-ref/nix-collect-garbage.html) so that we have a clean setup for our experiments:

Прежде чем начать, запустим [сборщик мусора](https://nixos.org/manual/nix/stable/command-ref/nix-collect-garbage.html), так что у нас будет чистая установка для наших эксперментов:

```bash
$ nix-collect-garbage
finding garbage collector roots...
[...]
deleting unused links...
note: currently hard linking saves -0.00 MiB
1169 store paths deleted, 228.43 MiB freed
```

> If we run the garbage collector again it won\'t find anything new to delete, as we expect.
> After running the garbage collector, the nix store only contains paths with references from the GC roots.

Если мы снова запустим сборщик мусора, он, как мы и ожидаем, не найдёт объектов для удаления.
После завершения сборки мусора, хранилище Nix содержит  только путь с ссылками из корней СМ.

> We now install a new program, `bsd-games`, inspect its store path, and examine its GC root.
> The `nix-store -q --roots` command is used to query the GC roots that refer to a given derivation.
> In this case, our current user environment refers to `bsd-games`:

Теперь мы установим новую программу, `bsd-games`, исследуем её путь в хранилище, и убедимся, что он является СМ-корнем.
Команда `nix-store -q --roots` используется, чтобы запросить корни СМ, которые ссылаются на заданную деривацию.
В нашем случае, нашое окружение текущего пользователя ссылается на `bsd-games`:

```bash
$ nix-collect-garbage
finding garbage collector roots...
[...]
deleting unused links...
note: currently hard linking saves -0.00 MiB
1169 store paths deleted, 228.43 MiB freed
```

> Now we remove it and run the garbage collector, and note that `bsd-games` is still in the nix store:

Теперь мы удалим его, запустим сборщик мусора и убедимся,что `bsd-games` всё ешё находится в хранилище Nix.

```bash
$ rm /nix/var/nix/profiles/default-9-link
$ nix-env --list-generations
[...]
   8   2014-07-28 10:23:24
  10   2014-08-20 12:47:16   (current)
$ nix-collect-garbage
[...]
$ ls /nix/store/b3lxx3d3ggxcggvjw5n0m1ya1gcrmbyn-bsd-games-2.17
ls: cannot access /nix/store/b3lxx3d3ggxcggvjw5n0m1ya1gcrmbyn-bsd-games-2.17: No such file or directory
```

> The old generation is still in the nix store because it is a GC root.
> As we will see below, all profiles and their generations are automatically GC roots.

Старое поколение всё ещё в хранилище Nix, потому что это корент СМ.
Как вы увидим ниже, все рофили и их поколения автоматически становятся корнями СМ.

> Removing a GC root is simple.
> In our case, we delete the generation that refers to `bsd-games`, run the garbage collector, and note that `bsd-games` is no longer in the nix store:

Удаление корня СМ — это просто.
В нашем случае мы удаляем поколение, которое ссылается на `bsd-games`, запускаем сборщик мусора и убеждаемся, что `bsd-games` больше недоступен в хранилище Nix:

> [Note]{.underline}: `nix-env --list-generations` does not rely on any particular metadata. It is able to list generations based solely on the file names under the profiles directory.

Обратите внимание: `nix-env --list-generations` не опирается на какие-либо метаданные. Он может перечислять поколения, основываясь исключительно на именах файлов в каталоге профилей.

> Note that we removed the link from `/nix/var/nix/profiles`, not from `/nix/var/nix/gcroots`.
> In addition to the latter, Nix treats `/nix/var/nix/profiles` as a GC root.
> This is useful because it means that any profile and its generations are GC roots.
> Other paths are considered GC roots as well; for example, `/run/booted-system` on NixOS.
> The command `nix-store --gc --print-roots` prints all paths considered as GC roots when running the garbage collector.

Обратите внимание, что мы удалили ссылку из каталога `/nix/var/nix/profiles`, не из `/nix/var/nix/gcroots`.
Nix трактует `/nix/var/nix/profiles` как корень СМ, помимо прочего.
Это полезно, поскольку это значит, что любой профиль и его поколения являются корнями СМ.
Другие пути также считаются корнями СМ, например, `/run/booted-system` в NixOS.
Команда `nix-store --gc --print-roots` печатает все пути, которые считаются корнями СМ при запуске сборщика мусора.

> ## Indirect roots

## Косвенные корни

> Recall that building the GNU `hello` package with `nix-build` produces a `result` symlink in the current directory.
> Despite the garbage collection done above, the `hello` program is still working.
> Therefore, it has not been garbage collected.
> Since there is no other derivation that depends upon the GNU `hello` package, it must be a GC root.

Вспомните, что построение пакета GNU `hello` с помощью `nix-build` создаёт символическую ссылку `result` в текущем каталоге.
Несмотря на сделанную выше сборку мусора, программа `hello` всё ещё работает.
Следовательно, она не была собрана.
Так как не существует других дериваций, которые зависят от пакета GNU `hello`, он обязан быть корнем СМ.

> In fact, `nix-build` automatically adds the `result` symlink as a GC root.
> Note that this is not the built derivation, but the symlink itself.
> These GC roots are added under `/nix/var/nix/gcroots/auto`.

Фактически, `nix-build` автоматически делает символическую ссылку `result` корнем СМ.
Обратите внимание, что речь идёт не о собранной деривации, а только о символической ссылке.
Эти корни СМ добавляются в каталог `/nix/var/nix/gcroots/auto`.

```bash
$ ls -l /nix/var/nix/gcroots/auto/
total 8
drwxr-xr-x 2 nix nix 4096 Aug 20 10:24 ./
drwxr-xr-x 3 nix nix 4096 Jul 24 10:38 ../
lrwxrwxrwx 1 nix nix   16 Jul 31 10:51 xlgz5x2ppa0m72z5qfc78b8wlciwvgiz -> /home/nix/result/
```

> The name of the GC root symlink is not important to us at this time.
> What is important is that such a symlink exists and points to `/home/nix/result`.
> This is called an **indirect GC root**.
> A GC root is considered indirect if its specification is outside of `/nix/var/nix/gcroots`.
> In this case, this means that the target of the `result` symlink will not be garbage collected.

Нам пока не важно, какое имя носит символическая ссылка, ставшая корнем СМ.
Значение имеет то, что она существует и указывает на `/home/nix/result`.
Это называется **косвенным корнем СМ**.
Корень СМ считается косвенным, если он определён где-то вне каталога `/nix/var/nix/gcroots`.
В нашем случае это значит, что цель символической ссылки `result` не будет удаляться сборщиком мусора.

> To remove a derivation considered \"live\" by an indirect GC root, there are two possibilities:

Есть два способа удалить деривацию, считающуюся «живой» из-за косвенного корня СМ:

> -   Remove the indirect GC root from `/nix/var/nix/gcroots/auto`.
> -   Remove the `result` symlink.

- Удалить косвенный корень GC из `/nix/var/nix/gcroots/auto`.
- Удалить символическую ссылку `result`.

> In the first case, the derivation will be deleted from the nix store during garbage collection, and `result` becomes a dangling symlink.
> In the second case, the derivation is removed as well as the indirect root in `/nix/var/nix/gcroots/auto`.

В первом случае деривация будет удалена из хранилища Nix в течение сборки мусора и `result` станет висячей символической ссылкой.
Во втором случае деривация удаляется вместе с косвенным корнем в `/nix/var/nix/gcroots/auto`.

> Running `nix-collect-garbage` after deleting the GC root or the indirect GC root will remove the derivation from the store.

Запуск `nix-collect-garbage` после удаления корня СМ или косвенного корня СМ удалит деривацию из хранилища.

> ## Cleanup everything

## Чистим всё

> The main source of software duplication in the nix store comes from GC roots, due to `nix-build` and profile generations.
> Running `nix-build` results in a GC root for the build that refers to a specific version of specific libraries, such as glibc.
> After an upgrade, we must delete the previous build if we want the garbage collector to remove the corresponding derivation, as well as if we want old dependencies cleaned up.

Главный источник дублирования программ в хранилище Nix возникает из-за корней СМ, связанных с `nix-build` и поколениями профиля.
Запуск `nix-build` приводит к созданию корня СМ для сборке, который ссылается на определённные версии определённых библиотек, например, `glibc`.
После обновления, мы должны удалить предыдущую сборку, если мы хотим, чтобы сборщик мусора удалил соответствующие деривации, а так же, если мы хотим очистить старые зависимости.

> The same holds for profiles.
> Manipulating the `nix-env` profile will create further generations.
> Old generations refer to old software, thus increasing duplication in the nix store after an upgrade.
То же самое касается и профилей.
Манипулирование профилем `nix-env` создаёт дополнительные поколения.
Старые поколения ссылаются на старые программы, увеличивая таким образом дублирование после обновления в хранилище Nix.

> Other systems typically \"forget\" everything about their previous state after an upgrade.
> With Nix, we can perform this type of upgrade (having Nix remove all old derivations, including old generations), but we do so manually.
> There are four steps to doing this:

Обычно другие системы «забывают» всё о своём предыдущем состоянии сразу после обновления.
В Nix, мы можем выполнить такой тип обновления (заставив Nix удалить старые деривации, включая старые поколения), но это надо сделать вручную.
Для этого нужно четыре шага:

> -   First, we download a new version of the nixpkgs channel, which holds the description of all the software.
>     This is done via `nix-channel --update`.
> -   Then we upgrade our installed packages with `nix-env -u`.
>     This will bring us into a new generation with updated software.
> -   Then we remove all the indirect roots generated by `nix-build`: beware, as this will result in dangling symlinks.
>     A smarter strategy would also remove the target of those symlinks.
> -   Finally, the `-d` option of `nix-collect-garbage` is used to delete old generations of all profiles, then collect garbage.
>     After this, you lose the ability to rollback to any previous generation.
>     It is important to ensure the new generation is working well before running this command.

- Во-первых, мы скачиваем новую версию канала nixpkgs, который содержит описание всех программ.
  Это делается с помощью `nix-channel --update`.
- Затем мы обновляем установленные пакеты, запустив `nix-env -u`.
  Так у нас появится новое поколение с обновлёнными программами.
- Затем мы удаляем все косвенные корни, созданные `nix-build`: будьте осторожны, потому что это приводит к появлением висячих системных ссылок.
  Более умной стратегией будет одновременное удаление целей этих символических ссылок.
- Наконец, опция `-d` при запуске `nix-collect-garbage` используется для удаления старых поколений всех профилей с последующей сборкой мусора.
  После этого, вы теряете возможность вернуться к предыдущим поколениям.
  Важно убедиться, что новое поколение корректно работает, прежде чем запускать эту команду.

> The four steps are shown below:

Эти четыре шага показаны ниже:

```bash
$ nix-channel --update
$ nix-env -u --always
$ rm /nix/var/nix/gcroots/auto/*
$ nix-collect-garbage -d
```

> ## Conclusion

## Заключение

> Garbage collection in Nix is a powerful mechanism to clean up your system.
> The `nix-store` commands allow us to know why a certain derivation is present in the nix store, and whether or not it is eligible for garbage collection.
> We also saw how to conduct more destructive deletion and upgrade operations.

Сборка мусора в Nix — это мощный механизм для очистки вашей системы.
Команды `nix-store` позволяют нам узнать, почему определённая деривация присутствует в хранилище Nix, независимо от того, подлежит ли она сборке, или нет.
Мы также узнали, как выполнить более деструктивные операции удаления и обновления.

> ## Next pill

## В следующей пилюле

> In the next pill, we will package another project and introduce the \"inputs\" design pattern.
> We\'ve only played with a single derivation until now; however we\'d like to start organizing a small repository of software.
> The \"inputs\" pattern is widely used in nixpkgs; it allows us to decouple derivations from the repository itself and increase customization opportunities.

В следующей пилюле мы запакуем другое проект и обхясним паттер проектирования «входы».
До сих пор мы работали только с одной деривацией; теперь нам предстоит организовать маленький репозиторий для программ.
Паттерн «входы» широко используется в nixpkgs; он позволяет нам разделять деривации из одного репозитория и увеличивает возможности настройки.