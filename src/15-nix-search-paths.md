> # Nix Search Paths

# Поисковые пути Nix

> Welcome to the 15th Nix pill.
> In the previous [14th](#override-design-pattern) pill we have introduced the \"override\" pattern, useful for writing variants of derivations by passing different inputs.

Добро пожаловать на пятнадцатую пилюлю Nix.
В предыдущей [четырнадцатой пилюле](14-override-design-pattern.md) мы познакомились с паттерном Переопределение, полезным при написании вариантов дериваций, путём передачи разнличных входящих параметров.

> Assuming you followed the previous posts, I hope you are now ready to understand `nixpkgs`.
> But we have to find `nixpkgs` in our system first!
> So this is the step: introducing some options and environment variables used by nix tools.

Полагая, что вы усвоили материал предыдущих постов, я надеюсь, что сейчас вы готовы разобраться, как устроен `nixpkgs`.
На сначала мы должны найти `nixpkgs` в нашей системе!
Так что первый шаг: рассказать о некоторых опциях и переменных окружения, используемых утилитами Nix.

> ## The NIX_PATH

## NIX_PATH

> The [NIX_PATH environment variable](https://nixos.org/manual/nix/stable/command-ref/env-common.html) is very important.
> It\'s very similar to the `PATH` environment variable.
> The syntax is similar, several paths are separated by a colon `:`.
> Nix will then search for something in those paths from left to right.

[Переменная окружения NIX_PATH](https://nixos.org/manual/nix/stable/command-ref/env-common.html) очень важна.
Она очень похожа на переменную окружения `PATH`.
Похожий синтаксис: несколько путей, разделённых символом `:`.
Nix ищет что-то по этим путям, начиная с левого и заканчивая правым.

> Who uses `NIX_PATH`?
> The nix expressions!
> Yes, `NIX_PATH` is not of much use by the nix tools themselves, rather it\'s used when writing nix expressions.

Где нужен `NIX_PATH`?
В выражениях Nix!
Да, `NIX_PATH` нужен не столько для инструментария Nix, как такового, а для написания выражений Nix.

> In the shell for example, when you execute the command `ping`, it\'s being searched in the `PATH` directories.
> The first one found is the one being used.

Например, в командной строке, когда вы запускаете `ping`, программа ищется в каталогах, перечисленных в `PATH`.
Первая найденная программа и будет запущена.

> In nix it\'s exactly the same, however the syntax is different.
> Instead of just typing `ping` you have to type `<ping>`.
> Yes, I know\... you are already thinking of `<nixpkgs>`.
> However, don\'t stop reading here, let\'s keep going.

В Nix всё точно также, хотя синтаксис отличается.
Вместо того, чтобы вводить `ping`, вы вводите `<ping>`.
Да, я знаю... вы уже подумали про `<nixpkgs>`.
Что ж, не прекращайте читать, давайте продолжим.

> What\'s `NIX_PATH` good for?
> Nix expressions may refer to an \"abstract\" path such as `<nixpkgs>`, and it\'s possible to override it from the command line.

Для чего же нужна переменная `NIX_PATH`?
Выражения Nix могут ссылаться на "абстрактный" путь, такой как `<nixpkgs>`, и этот путь можно переопределить из комадной строки.

> For ease we will use `nix-instantiate --eval` to do our tests.
> I remind you, [nix-instantiate](https://nixos.org/manual/nix/stable/command-ref/nix-instantiate.html) is used to evaluate nix expressions and generate the .drv files.
> Here we are not interested in building derivations, so evaluation is enough.
> It can be used for one-shot expressions.

Для простоты будем использовать `nix-instantiate --eval`, чтобы провести несколько тестов.
Напоминаю, что [nix-instantiate](https://nixos.org/manual/nix/stable/command-ref/nix-instantiate.html) используется для выполнения выражений и генерации файлов **.drv**.
Здесь нам не потребуется собирать деривации, так что выполнения будет достаточно.
Программу можно использовать для запуска одноразовых выражений.

> ## Fake it a little

## Небольшая подделка

> It\'s useless from a nix view point, but I think it\'s useful for your own understanding.
> Let\'s use `PATH` itself as `NIX_PATH`, and try to locate `ping` (or another binary if you don\'t have it).

Это бессмысленно с точки зрения Nix, но я думаю, что полезно с точки зрения вашего понимания.
Давайте используем переменную `PATH` вместо `NIX_PATH` и попытаемся найти `ping` (или любую другую утилиту, если этой у вас нет).

```bash
$ nix-instantiate --eval -E '<ping>'
error: file `ping' was not found in the Nix search path (add it using $NIX_PATH or -I)
$ NIX_PATH=$PATH nix-instantiate --eval -E '<ping>'
/bin/ping
$ nix-instantiate -I /bin --eval -E '<ping>'
/bin/ping
```

> Great.
> At first attempt nix obviously said could not be found anywhere in the search path.
> Note that the -I option accepts a single directory.
> Paths added with -I take precedence over `NIX_PATH`.

Прекрасно!
При первой попытке Nix однозначно заявил, что не смог найти программу ни в одном из поисковых путей.
Обратите внимание, что опция `-I` принимает в качестве параметра один каталог.
Пути, добавленные посредством `-I` имеют приоритет на путями из `NIX_PATH`.

> The `NIX_PATH` also accepts a different yet very handy syntax: \"`somename=somepath`\".
> That is, instead of searching inside a directory for a name, we specify exactly the value of that name.

Помимо путей, `NIX_PATH` может содержать отличающийся очень удобный синтаксис: "`somename=somepath`".
То есть, вместо поиска программы внутри каталога, мы указываем точное место нахождения этой программы.

```bash
$ NIX_PATH="ping=/bin/ping" nix-instantiate --eval -E '<ping>'
/bin/ping
$ NIX_PATH="ping=/bin/foo" nix-instantiate --eval -E '<ping>'
error: file `ping' was not found in the Nix search path (add it using $NIX_PATH or -I)
```

> Note in the second case how Nix checks whether the path exists or not.

Обратите внимание, что во втором случае Nix проверяет, существует путь или нет.

> ## The path to repository

## Путь к репозиторию

> You are out of curiosity, right?

Вам ведь любопытно, да?

```bash
$ nix-instantiate --eval -E '<nixpkgs>'
/home/nix/.nix-defexpr/channels/nixpkgs
$ echo $NIX_PATH
nixpkgs=/home/nix/.nix-defexpr/channels/nixpkgs
```

> You may have a different path, depending on how you added channels etc.
> Anyway that\'s the whole point.
> The `<nixpkgs>` stranger that we used in our nix expressions, is referring to a path in the filesystem specified by `NIX_PATH`.

У вас может быть другой путь, в зависимости от того, как вы добавляли каналы, и т.д.
В этом, в общем-то, и суть.
Переменная `<nixpkgs>`, которую мы использовали, не понимая, как она работает, ссылается на каталог в файловой системе, определённый в переменной `NIX_PATH`.

> You can list that directory and realize it\'s simply a checkout of the nixpkgs repository at a specific commit (hint: `.version-suffix`).

Вы можете просмотреть этот каталог и увдиеть, что он просто содержит снимок репозитория nixpkgs, какую-то определённую фиксацию-коммит (подсказка: `.version-suffix`).

> The `NIX_PATH` variable is exported by `nix.sh`, and that\'s the reason why I always asked you to [source nix.sh](https://nixos.org/manual/nix/stable/installation/env-variables.html) at the beginning of my posts.

Переменная `NIX_PATH` экспортируется скриптом `nix.sh` и это именно та причина, по которой я всегда просил вас выполнить команду [`source nix.sh`]((https://nixos.org/manual/nix/stable/installation/env-variables.html)) в начале своих постов.

> You may wonder: then I can also specify a different [nixpkgs](https://github.com/NixOS/nixpkgs) path to, e.g., a `git checkout` of `nixpkgs`?
> Yes, you can and I encourage doing that.
> We\'ll talk about this in the next pill.

Возможно, вы задаётесь вопросом: значит, я могу указать другой путь к [nixpkgs](https://github.com/NixOS/nixpkgs), например, выполнив `git checkout` из репозитория `nixpkgs`?
Да, вы можете, более того, я рекомендую вам это сделать.
Мы подробно обсудим этот вопрос в следующей пилюле.

> Let\'s define a path for our repository, then!
> Let\'s say all the `default.nix`, `graphviz.nix` etc. are under `/home/nix/mypkgs`:

Давайте тогда определим путь к нашему репозиторию!
Пусть все пакеты `default.nix`, `graphviz.nix` и прочие будут в каталоге `/home/nix/mypkgs`:

```bash
$ export NIX_PATH=mypkgs=/home/nix/mypkgs:$NIX_PATH
$ nix-instantiate --eval '<mypkgs>'
{ graphviz = <code>; graphvizCore = <code>; hello = <code>; mkDerivation = <code>; }
```

> Yes, `nix-build` also accepts paths with angular brackets.
> We first evaluate the whole repository (`default.nix`) and then pick the graphviz attribute.

Да, `nix-build` тоже принимает пути в угловых скобках.
Сначала мы выполням выражение для всего репозитория (из файла `default.nix`), а затем получаем возможность выбрать атрибут, например, `graphviz`.

> ## A big word about nix-env

## Несколько слов о `nix-env`

> The [nix-env](https://nixos.org/manual/nix/stable/command-ref/nix-env.html) command is a little different than `nix-instantiate` and `nix-build`.
> Whereas `nix-instantiate` and `nix-build` require a starting nix expression, `nix-env` does not.

Команда [nix-env](https://nixos.org/manual/nix/stable/command-ref/nix-env.html) несколько отличается от `nix-instantiate` и `nix-build`.
В то время, как `nix-instantiate` и `nix-build` требуют запуска выражения Nix, `nix-env` — нет.

> You may be crippled by this concept at the beginning, you may think `nix-env` uses `NIX_PATH` to find the `nixpkgs` repository. But that\'s not it.

Эта концепция может сбить вас с толку, вы можете подумать, что `nix-env` использует `NIX_PATH` для того, чтобы найти репозиторий `nixpkgs`. Но нет.

> The `nix-env` command uses `~/.nix-defexpr`, which is also part of `NIX_PATH` by default, but that\'s only a coincidence.
> If you empty `NIX_PATH`, `nix-env` will still be able to find derivations because of `~/.nix-defexpr`.

Команда `nix-env` использует `~/.nix-defexpr`, который по умолчанию также является частью `NIX_PATH`, но это всего лишь совпадение.
Если вы очистите `NIX_PATH`, `nix-env` всё равно сможет искать деривации из-за `~/.nix-defexpr`.

> So if you run `nix-env -i graphviz` inside your repository, it will install the nixpkgs one.
> Same if you set `NIX_PATH` to point to your repository.

Так что если вы запустите `nix-env -i graphviz` в каталоге вашего репозитория, утилита установит пакет из `nixpkgs`.
То же самое, если вы установите `NIX_PATH` так, чтобы он указывал на ваш репозиторий.

> In order to specify an alternative to `~/.nix-defexpr` it\'s possible to use the -f option:

Чтобы указать альтернативный путь вместо `~/.nix-defexpr`, можно использовать опцию **-f**:

```bash
$ nix-env -f '<mypkgs>' -i graphviz
warning: there are multiple derivations named `graphviz'; using the first one
replacing old `graphviz'
installing `graphviz'
```

> Oh why did it say there\'s another derivation named graphviz?
> Because both `graphviz` and `graphvizCore` attributes in our repository have the name \"graphviz\" for the derivation:

Но почему утилита сказала, что существует другая деривация с именем `graphviz`?
Потому что оба атрибута `graphviz` и `graphvizCore` в вашем репозитории имеют имя деривации "graphviz":

```bash
$ nix-env -f '<mypkgs>' -qaP
graphviz      graphviz
graphvizCore  graphviz
hello         hello
```

> By default `nix-env` parses all derivations and uses the derivation names to interpret the command line.
> So in this case \"graphviz\" matched two derivations.
> Alternatively, like for `nix-build`, one can use -A to specify an attribute name instead of a derivation name:

По умолчанию `nix-env` разбирает все деривации и использует имена дериваций, чтобы интепретировать коммандную строку.
Так что в данном случае "graphviz" соответствует двум деривациям.
В качестве альтернативы, как и для `nix-build`, вы можете использовать **-A**, чтобы программа использовала имена атрибутов вместо имён дериваций:

```bash
$ nix-env -f '<mypkgs>' -i -A graphviz
replacing old `graphviz'
installing `graphviz'
```

> This form, other than being more precise, it\'s also faster because `nix-env` does not have to parse all the derivations.

Такая форма, помимо того, что она точнее, также и быстрее, поскольку `nix-env` не нужно проверять все деривации.

> For completeness: you must install `graphvizCore` with -A, since without the -A switch it\'s ambiguous.

Для полноты картины: вы *должны* устанавливать `graphvizCore` с ключом **-A**, поскольку без него выбор деривации неоднозначен.

> In summary, it may happen when playing with nix that `nix-env` picks a different derivation than `nix-build`.
> In that case you probably specified `NIX_PATH`, but `nix-env` is instead looking into `~/.nix-defexpr`.

Подводя итог, можно сказать, что вероятна ситуация, когда `nix-env` выберет деривацию, отличную от той, которую выберет `nix-build`.
При этом вы, вероятно, указали правильный путь в `NIX_PATH`, но `nix-env` ищет его в `~/.nix-defexpr`.

> Why is `nix-env` having this different behavior?
> I don\'t know specifically by myself either, but the answers could be:

Почему `nix-evn` ведёт себя иначе?
Я сам точно не знаю, но ответы могут быть такими:

> - `nix-env` tries to be generic, thus it does not look for `nixpkgs` in `NIX_PATH`, rather it looks in `~/.nix-defexpr`.
> - `nix-env` is able to merge multiple trees in `~/.nix-defexpr` by looking at all the possible derivations

- `nix-env` пытается быть универсальной утилитой, поэтому не заглядывает в `NIX_PATH`, чтобы найти `nixpkgs`, а ищет его в `~/.nix-defexpr`.
- `nix-env` позволяет слить в одном файле `~/.nix-defexpr` несколько деревьев, просмотрев все возможные деривации.

> It may also happen to you **that you cannot match a derivation name when installing**, because of the derivation name vs -A switch described above.
> Maybe `nix-env` wanted to be more friendly in this case for default user setups.

Также, возможно, дело в ошибке **нельзя сопоставить имя деривации при установке** (you cannot match a derivation name when installing), возникающей из-за неоднозначного имени деривации, как это описано выше.
Может быть так, что `nix-env` стремится быть более дружественной утилитой при обычных настройках пользователя.

> It may or may not make sense for you, or it\'s like that for historical reasons, but that\'s how it works currently, unless somebody comes up with a better idea.

В этом может быть смысл, а может и не быть, или дело в том, что так сложилось исторически, но сейчас всё работает именно так, по крайней мере, пока кто-нибудь не придумает идею получше.

> ## Conclusion

## Заключение

> The `NIX_PATH` variable is the search path used by nix when using the angular brackets syntax.
> It\'s possible to refer to \"abstract\" paths inside nix expressions and define the \"concrete\" path by means of `NIX_PATH`, or the usual -I flag in nix tools.

Переменная `NIX_PATH` содержит поисковые пути, используемые утилитами Nix, когда вы записываете имя пакета в угловых скобках.
Можно ссылаться на "абстрактные" пути в выражениях Nix и опредлять "конкретный" путь с помощью `NIX_PATH` или обычного флага **-I** в утилитах Nix.

> We\'ve also explained some of the uncommon `nix-env` behaviors for newcomers.
> The `nix-env` tool does not use `NIX_PATH` to search for packages, but rather for `~/.nix-defexpr`.
> Beware of that!

Мы также объяснили некоторые необычные особенности `nix-env` для начинающих.
Утилита `nix-env` не использует `NIX_PATH` для поиска пакетов, а вместо переменной окружения смотрит в файл `~/.nix-defexpr`.
Будьте осторожны!

> In general do not abuse `NIX_PATH`, when possible use relative paths when writing your own nix expressions.
> Of course, in the case of `<nixpkgs>` in our repository, that\'s a perfectly fine usage of `NIX_PATH`.
> Instead, inside our repository itself, refer to expressions with relative paths like `./hello.nix`.

В общем случае, не злоупотребляйте `NIX_PATH` и по возможности используйте относительные пути при написании собственных выражений Nix.
Конечно, если речь идёт о `<nixpkgs>` в нашем репозитории, использовать `NIX_PATH` совершенно нормально.
А вот внутри репозитория ссылайтесь на выражения по относительным путям, например, `./hello.nix`.

> ## Next pill

## В следующей пилюле

> \...we will finally dive into `nixpkgs`.
> Most of the techniques we have developed in this series are already in `nixpkgs`, like `mkDerivation`, `callPackage`, `override`, etc., but of course better.
> With time, those base utilities get enhanced by the community with more features in order to handle more and more use cases and in a more general way.

...мы наконец погрузимся в `nixpkgs`.
Большинство из техник, которые мы разработали в этом цикле, используются в `nixpkgs`. Это и `mkDerivation`, и `callPackage`, и `override`, и др., но, конечно, более проработанные.
Со временем, сообщество расширит эти базовые утилиты, добавляя новые функции, чтобы обрабатывать больше сценариев более общим способом.
