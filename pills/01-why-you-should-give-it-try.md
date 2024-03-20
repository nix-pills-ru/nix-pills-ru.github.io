> # Why You Should Give it a Try

# Почему вам стоит попробовать Nix

> ## Introduction

## Введение

> Welcome to the first post of the "[Nix](https://nixos.org/nix) in pills" series.
> Nix is a purely functional package manager and deployment system for POSIX.

Добро пожаловать в первую статью из серии "[Nix](https://nixos.org/nix) в пилюлях".

> There's a lot of documentation that describes what Nix, [NixOS](https://nixos.org/nixos) and related projects are.
> But the purpose of this post is to convince you to give Nix a try.
> Installing NixOS is not required, but sometimes I may refer to NixOS as a real world example of Nix usage for building a whole operating system.

Есть много материалов, посвящённых Nix, [NixOS](https::/nixos.org/nixos) и связанным проектам.
Но цель этой статьи — убедить вас попробовать Nix.
Установка NixOS не потребуется, но иногда я буду могу ссылать на NixOS, как на реальный пример построения полноценной операционной системы с помощью Nix.

> ## Rationale for this series

## Для чего написаны эти статьи

> The [Nix](https://nixos.org/manual/nix), [Nixpkgs](https://nixos.org/manual/nixpkgs/), and [NixOS](https://nixos.org/manual/nixos/) manuals along with [the wiki](https://nixos.wiki/) are excellent resources for explaining how Nix/NixOS works, how you can use it, and how cool things are being done with it.
> However, at the beginning you may feel that some of the magic which happens behind the scenes is hard to grasp.

Руководства по [Nix](https://nixos.org/manual/nix), [Nixpkgs](https://nixos.org/manual/nixpkgs/) и [NixOS](https://nixos.org/manual/nixos/) вместе с [вики](https://nixos.wiki/) — великолепные ресурсы, которые объясняют, как устроены Nix/NixOS, как их использовать, и насколько крутые штуки можно делать с их помощью.

> This series aims to complement the existing explanations from the more formal documents.

Цель этих статей — дополнить существующие документы чуть менее формальными объяснениями.

> The following is a description of Nix.
> Just as with pills, I'll try to be as short as possible.

Теперь мы препарируем Nix.
Как и в случае с другими медицинскими процедурами, постараюсь закончить всё как можно быстрее.

> ## Not being purely functional

## Не чистый функцоинальный

> Most, if not all, widely used package managers ([dpkg](https://wiki.debian.org/dpkg), [rpm](http://www.rpm.org/), ...) mutate the global state of the system.
> If a package `foo-1.0` installs a program to `/usr/bin/foo`, you cannot install `foo-1.1` as well, unless you change the installation paths or the binary name.
> But changing the binary names means breaking users of that binary.

Большинство, если не все, широко применяемых пакетных менеджеров ([dpkg](https://wiki.debian.org/dpkg), [rpm](http://www.rpm.org/), ...) изменяют глобальное состояние системы.
Установив пакет `foo-1.0` в каталог `/usr/bin/foo`, вы не сможете установить туда же `foo-1.1`, пока не измените путь установки или имя исполняемого файла.
Но изменение имени файла обманывает ожидания пользователей пакета.

> There are some attempts to mitigate this problem.
> Debian, for example, partially solves the problem with the [alternatives](https://wiki.debian.org/DebianAlternatives) system.

Есть разные способы снять эту проблему. Например, Debian, частично решает её с помощью системы [альтернатив](https://wiki.debian.org/DebianAlternatives).

> So while in theory it's possible with some current systems to install multiple versions of the same package, in practice it's very painful.

Так что в теории мы можем установить несколько версий одного пакета, но на практике этот опыт может быть весьма болезненным.

> Let's say you need an nginx service and also an nginx-openresty service.
> You have to create a new package that changes all the paths to have, for example, an `-openresty` suffix.

Скажем, вам нужен сервис nginx и, кроме него — сервис nginx-openresty.
Вы должны создать новый пакет, и поменять в нём все пути, например, добавив к ним суффикс `-openresty`.

> Or suppose that you want to run two different instances of mysql: 5.2 and 5.5.
> The same thing applies, plus you have to also make sure the two mysqlclient libraries do not collide.

Или представим, вам надо запустить две разные версии mysql: 5.2 и 5.5.
Возникнет та же свистопляска с путями, кроме того, вам надо будет убедиться, что две версии библиотеки mysqlclient не конфликтуют друг с другом.

> This is not impossible but it *is* very inconvenient.
> If you want to install two whole stacks of software like GNOME 3.10 and GNOME 3.12, you can imagine the amount of work.

Это не то, чтобы невозможно, но *очень* неудобно.

> From an administrator's point of view: you can use containers.
> The typical solution nowadays is to create a container per service, especially when different versions are needed.
> That somewhat solves the problem, but at a different level and with other drawbacks.
> For example, needing orchestration tools, setting up a shared cache of packages, and new machines to monitor rather than simple services.

Адмнистраторы скажут: вы можете использовать контейнеры.
Типичный подход в наши дни — создать контейнер для каждого сервиса, особенно, когда нужны разные версии.
Он в какой-то степени решает проблему, но на другом уровне и с другими побочными эффектами.

> From a developer's point of view: you can use virtualenv for python, or jhbuild for gnome, or whatever else.
> But then how do you mix the two stacks?
> How do you avoid recompiling the same thing when it could instead be shared?
> Also you need to set up your development tools to point to the different directories where libraries are installed.
> Not only that, there's the risk that some of the software incorrectly uses system libraries.

Разработчики скажут: вы можете использовать virtualenv для python, или jhbuild для gnome, или что-то ещё.
Но как вы тогда смешаете два разных стека?
Как вам избежать перекомпиляции исходников, когда их надо разделить между проектами?
Помимо прочего, вам придётся настраивать инструменты разработки, чтобы они загружали библиотеки из правильных каталогов.
Наконец, есть риск, что часть программ некорректно работают с системными библиотеками.

> And so on.
> Nix solves all this at the packaging level and solves it well.
> A single tool to rule them all.

И так далее.
Nix решает все эти проблемы на уровне пакетов и решает их хорошо.
Один инструмент — управлять всеми пакетами.

> ## Being purely functional

## Чистый функциональный

> Nix makes no assumptions about the global state of the system.
> This has many advantages, but also some drawbacks of course.
> The core of a Nix system is the Nix store, usually installed under `/nix/store`, and some tools to manipulate the store.
> In Nix there is the notion of a *derivation* rather than a package.
> The difference can be subtle at the beginning, so I will often use the words interchangeably.

Nix не делает никаких предположений относительно глобального состояния системы.
У такого подхода есть много преимуществ, но, конечно, есть и недостатки.
Сердцем системы Nix является хранилище, обычно расположенное в `/nix/store` и кое-какие инструменты для работы с ним.
В Nix вместо понятия пакет существует понятие *порождение* (*derivation*).
Для новичков различия между ними кажутся слишком тонкими, поэтому я буду использовать эти слова, как синонимы.

> Derivations/packages are stored in the Nix store as follows: `/nix/store/hash-name`, where the hash uniquely identifies the derivation (this isn't quite true, it's a little more complex), and the name is the name of the derivation.

Порождения/пакеты хранятся в хранилище Nix в соответствии с форматом `/nix/store/hash-name`, где хэш (hash) однозначено идентифицирует порождение (это не совсем правда, в действтительности всё немного сложнее), а имя (name) — это имя порождения.

> Let's take a bash derivation as an example: `/nix/store/s4zia7hhqkin1di0f187b79sa2srhv6k-bash-4.2-p45/`.
> This is a directory in the Nix store which contains `bin/bash`.

Взглянем, для пример, на порождение bash: `/nix/store/s4zia7hhqkin1di0f187b79sa2srhv6k-bash-4.2-p45/`.
Этот каталог в хранилище Nix содержит утилиту `bin/bash`.

> What that means is that there's no `/bin/bash`, there's only that self-contained build output in the store.
> The same goes for coreutils and everything else.
> To make them convenient to use from the shell, Nix will arrange for binaries to appear in your `PATH` as appropriate.

Фактически это означает, что в системе нет никакой глобальной оболочки, а есть только эта конкретная версия в одном из каталогов хранилища.
То же касается и других утилит, да и вообще всего.
Чтобы утилиты можно было вызывать из командной строки, Nix следит за тем, чтобы в переменной `PATH` были правильные пути.

> What we have is basically a store of all packages (with different versions occupying different locations), and everything in the Nix store is immutable.

В итоге у нас есть хранилище всех пакетов (разные версии пакетов хранятся в разных каталогах), и всё, что там есть — менять нельзя.

> In fact, there's no ldconfig cache either.
> So where does bash find libc?

Фактичски, в системе нет даже кэша ldconfig.
Как, в таком случае, bash находит libc?

> It turns out that when bash was built, it was built against that specific version of glibc in the Nix store, and at runtime it will require exactly that glibc version.

Дело в том, что когда bash был собран, он был собран с конкретной версией glibc из хранилища Nix, и при запуске он загружает именно эту версию glibc.

> Don't be confused by the version in the derivation name: it's only a name for us humans.
> You may end up having two derivations with the same name but different hashes: it's the hash that really matters.

Пусть вас не смущает номер версии в имени порождения: это имя для нас, людей.
Мы можете создать два порождения с одним и тем же именем, но разными хэшами: значение имеет только хэш.

> What does all this mean?
> It means that you could run mysql 5.2 with glibc-2.18, and mysql 5.5 with glibc-2.19.
> You could use your python module with python 2.7 compiled with gcc 4.6 and the same python module
with python 3 compiled with gcc 4.8, all in the same system.

Для чего все эти сложности?
Для того, что теперь вы можете запускать mysql 5.2 с glibc-2.18 и mysql 5.5 с glibc-2.19.
Вы можете использовать ваш модуль c python 2.7, собранным gcc 4.6 и тот же самый модуль — с python 3, собранным gcc 4.8, в той же самой системе.

> In other words: no dependency hell, not even a dependency resolution algorithm.
> Straight dependencies from derivations to other derivations.

Другими словами: никакого больше ада зависимостей (dependency hell) и даже никакого алгоритма разрешения зависимостей.
Прямые зависимости порождений от других порождений.

> From an administrator's point of view: if you want an old PHP version for one application, but want to upgrade the rest of the system, that's not painful any more.

С точки зрения администратора: если вам нужна старая версия PHP для одного приложения, но вы хотите обновить всю остальную систему, это можно сделать безболезненно.

> From a developer's point of view: if you want to develop webkit with llvm 3.4 and 3.3, that's not painful any more.

С точки зрения разработчика: если вы хотите разрабатывать webkit и llvm 3.4 и с llvm 3.3, это можно сделать безболезненно.

> ## Mutable vs. immutable

## Изменяемое против неизменного

> When upgrading a library, most package managers replace it in-place.
> All new applications run afterwards with the new library without being recompiled.
> After all, they all refer dynamically to `libc6.so`.

При обновлении библиотеки большинство пакетных менеджеров просто перезаписывают файл в каталоге.
После этого все приложения запускаются с новой версией библиотеки без перекомпиляции.
В конце концов, все они динамически ссылаются на `libc6.so`.

> Since Nix derivations are immutable, upgrading a library like glibc means recompiling all applications, because the glibc path to the Nix store has been hardcoded.

Поскольку порождения Nix неизменны (иммубельны), обновление библиотеки наподобие glibc требует перекомпиляции всех приложений, потому что путь к glibc в хранилище Nix зависит от версии.

> So how do we deal with security updates?
> In Nix we have some tricks (still pure) to solve this problem, but that's another story.

Как же нам быть с обновлениями безопасности?
У нас в Nix есть несколько трюков (всё ещё чистых), чтобы справиться с этой проблемой, но это уже другая история.

> Another problem is that unless software has in mind a pure functional model, or can be adapted to it, it can be hard to compose applications at runtime.

Другая проблема заключается в том, что если программное обеспечение не разрабатывалось с расчётом на запуск из каталога только для чтения, или его няльзя адаптировать к такому запуску, может быть сложно заставить такие приложения работать.

> Let's take Firefox for example.
> On most systems, you install flash, and it starts working in Firefox because Firefox looks in a global path for plugins.

Возьмём для примера Firefox.
В большинстве системы вы устанавливаете flash, и он просто начинает работать, потому что Firefox ищет плагины по глобальному пути.

> In Nix, there's no such global path for plugins.
> Firefox therefore must know explicitly about the path to flash.
> The way we handle this problem is to wrap the Firefox binary so that we can setup the necessary environment to make it find flash in the nix store.
> That will produce a new Firefox derivation: be aware that it takes a few seconds, and it makes composition harder at runtime.

В Nix не существует никого глобального пути для плагинов.
Таким образом, Firefox должен точно значть о пути к flash.
Мы справляемся с этой проблемой, создавая для Firefox особое окружение, позволяющее найти flash в хранилище Nix.
В результате будет создано новое порождение Firefox: это займёт всего несколько секунд и сделает работу приложения более сложной.

> There are no upgrade/downgrade scripts for your data.
> It doesn't make sense with this approach, because there's no real derivation to be upgraded.
> With Nix you switch to using other software with its own stack of dependencies, but there's no formal notion of upgrade or downgrade when doing so.

Вам больше не нужны скрипты обновления/удаления ваших данных.
В этом просто нет смысла, потому что не бывает порождений, которые можно было бы обновлять.
В Nix вы переключаетесь на другой софт со своим собственным набором зависимостей, но при этом нет никаких обновлений или удалений.

> If there is a data format change, then migrating to the new data format remains your own responsibility.

Если изменился формат данных, то миграция на новый формат — это ваша отвественность.

> ## Conclusion

## Заключение

> Nix lets you compose software at build time with maximum flexibility, and with builds being as reproducible as possible.
> Not only that, due to its nature deploying systems in the cloud is so easy, consistent, and reliable that in the Nix world all existing self-containment and orchestration tools are deprecated by [NixOps](http://nixos.org/nixops/).

Nix позволяет вам с максимальной гибкостью управлять компоновкой программ, оставляя сборки настолько воспроизводимыми, насколько это возможно.
Более того, из-за природы Nix, разворачивание систем в облаке настолько просто, согласовано и надёжно, что в мире Nix все существующие инструменты контейнеризации и оркестрации безнадёжно устарели по сравнению с [NixOps](http://nixos.org/nixops/).

> It however *currently* falls short when working with dynamic composition at runtime or replacing low level libraries, due to the need to rebuild dependencies.

Тем не менее, он *пока* не справляется с динамической компоновкой при работе программ, или с заменой низкоуровневых библиотек, из-за того, что всё это требует перекомпиляции.

> That may sound scary, however after running NixOS on both a server and a laptop desktop, I'm very satisfied so far.
> Some of the architectural problems just need some man-power, other design problems still need to be solved as a community.

Это может звучать пугающе, однако после запуска NixOS и на сервере, и на десктопе, я очень доволен.
Некоторые проблемы архитектуры просто нуждаются в человеческих ресурсах, другие проблемы по-прежнему требуют решения со стороны сообщества разработчиков.

> Considering [Nixpkgs](https://nixos.org/nixpkgs/) ([github link](https://github.com/NixOS/nixpkgs)) is a completely new repository of all the existing software, with a completely fresh concept, and with few core developers but overall year-over-year increasing contributions, the current state is more than acceptable and beyond the experimental stage.
> In other words, it's worth your investment.

Взглянув на [Nixpkgs](https://nixos.org/nixpkgs) ([ссылка на github](https://github.com/NixOS/nixpkgs)) — совершенно новый репозиторий всего существующего софта, с совершенно свежим подходом, с небольшим количеством основных разработчиков, но с растущим год от года вкладом сообщества, мы должны признать, что он вышел из стадии эксперимента, и его состояние более чем удовлетворительно.
Другими словами, он стоит потраченного на него времени.

> ## Next pill...

## В следующей пилюле...

> ...we will install Nix on top of your current system (I assume GNU/Linux, but we also have OSX users) and start inspecting the installed software.

... мы установим Nix в вашу систему (предположительно GNU/Linux, но подойдёт и OSX), и начнём его изучать. 