> # Nix Store Paths

# Пути хранения Nix

> Welcome to the 18th Nix pill.
> In the previous [17th](#nixpkgs-overriding-packages) pill we have scratched the surface of the `nixpkgs` repository structure.
> It is a set of packages, and it\'s possible to override such packages so that all other packages will use the overrides.

Добро пожаловать на восемнадцатую пилюлю Nix.
В предыдущей [семнадцатой пилюле](17-nixpkgs-overriding-packages.md) поверхностно изучили со структурой репозитория `nixpkgs`.
Это набор покетов, и мы можем переопределять такие пакеты так, что все остальные пакеты смогут использовать переопределения.

> Before reading existing derivations, I\'d like to talk about store paths and how they are computed.
> In particular we are interested in fixed store paths that depend on an integrity hash (e.g. a sha256), which is usually applied to source tarballs.

Прежде, чем познакомиться с существующими деривациями, я бы хотел поговорить о путях хранения и о том, как они вычисляются.
В настности нас интересуют фиксированные пути хранения, которые зависят от хеша целостности (такого, как sha256), которые обычно есть у tar-архивов.

> The way store paths are computed is a little contrived, mostly due to historical reasons.
> Our reference will be the [Nix source code](https://github.com/NixOS/nix/blob/07f992a74b64f4376d5b415d0042babc924772f3/src/libstore/store-api.cc#L197).

Способ, которым вычисляются пути хранения, несколько запутанный, в основном, по историческим причинам.
Он описан в [исходном коде Nix](https://github.com/NixOS/nix/blob/07f992a74b64f4376d5b415d0042babc924772f3/src/libstore/store-api.cc#L197).

> ## Source paths

## Исходные пути

> Let\'s start simple.
> You know nix allows relative paths to be used, such that the file or directory is stored in the nix store, that is `./myfile` gets stored into `/nix/store/.......`
> We want to understand how is the store path generated for such a file:

Давайте начнём с простого.
Вы знаете, что Nix позволяет использовать относительные пути, поскольку файл или каталог находятся в хранилище Nix, то есть, например `./myfile` лежит в `/nix/store/.......`.
Мы хотим разобраться, как именно генерируется путь хранения для такого файла:

```bash
$ echo mycontent > myfile
```

> I remind you, the simplest derivation you can write has a `name`, a `builder` and the `system`:

Напомню, что у простейшей деривации, которую вы можете создать, должны быть имя (`name`), сборщик (`buidler`) и система (`system`):

```bash
$ nix repl
nix-repl> derivation { system = "x86_64-linux"; builder = ./myfile; name = "foo"; }
«derivation /nix/store/y4h73bmrc9ii5bxg6i7ck6hsf5gqv8ck-foo.drv»
```

> Now inspect the .drv to see where is `./myfile` being stored:

Теперь проверьте файл `.drv`, чтобы узнать, где хранится `./myfile`:

```bash
$ nix derivation show /nix/store/y4h73bmrc9ii5bxg6i7ck6hsf5gqv8ck-foo.drv
{
  "/nix/store/y4h73bmrc9ii5bxg6i7ck6hsf5gqv8ck-foo.drv": {
    "outputs": {
      "out": {
        "path": "/nix/store/hs0yi5n5nw6micqhy8l1igkbhqdkzqa1-foo"
      }
    },
    "inputSrcs": [
      "/nix/store/xv2iccirbrvklck36f1g7vldn5v58vck-myfile"
    ],
    "inputDrvs": {},
    "platform": "x86_64-linux",
    "builder": "/nix/store/xv2iccirbrvklck36f1g7vldn5v58vck-myfile",
    "args": [],
    "env": {
      "builder": "/nix/store/xv2iccirbrvklck36f1g7vldn5v58vck-myfile",
      "name": "foo",
      "out": "/nix/store/hs0yi5n5nw6micqhy8l1igkbhqdkzqa1-foo",
      "system": "x86_64-linux"
    }
  }
}
```

> Great, how did nix decide to use `xv2iccirbrvklck36f1g7vldn5v58vck`?
> Keep looking at the nix comments.

Хороший вопрос: почему Nix решил выбрать имя `xv2iccirbrvklck36f1g7vldn5v58vck`?
Взглянем на комметарии в исходном коде Nix.

> **Note:** doing `nix-store --add myfile` will store the file in the same store path.

**Обратите внимание:** запуск `nix-store --add myfile` сохранит файл по тому же пути хранения.

> ### Step 1, compute the hash of the file

### Шаг 1: вычисляем хеш файла

> The comments tell us to first compute the sha256 of the NAR serialization of the file.
> Can be done in two ways:

Комментарии подсказывают нам, что сначала вычисляется хеш sha256, точно таким же образом, как и при сериализации файла в архив NAR.
Это можно сделать двумя способами:

```bash
$ nix-hash --type sha256 myfile
2bfef67de873c54551d884fdab3055d84d573e654efa79db3c0d7b98883f9ee3
```

> Or:

Или:

```bash
$ nix-store --dump myfile|sha256sum
2bfef67de873c54551d884fdab3055d84d573e654efa79db3c0d7b98883f9ee3
```

> In general, Nix understands two contents: flat for regular files, or recursive for NAR serializations which can be anything.

В целом, Nix понимает содержимое двух видов: плоское для обычных файлов и ресурсивной для сериализаций NAR, которые могут быть чем угодно.

> ### Step 2, build the string description

## Шаг 2: строим строковое описание

> Then nix uses a special string which includes the hash, the path type and the file name.
> We store this in another file:

Затем Nix использует строку специального вида, которая включает хеш, тип пути и имя файла.
Сохраним её в отдельном файле:

```bash
$ echo -n "source:sha256:2bfef67de873c54551d884fdab3055d84d573e654efa79db3c0d7b98883f9ee3:/nix/store:myfile" > myfile.str
```

> ### Step 3, compute the final hash

### Шаг 3: вычисляем окончательный хеш

> Finally the comments tell us to compute the base-32 representation of the first 160 bits (truncation) of a sha256 of the above string:

В конце концов, комментарии подсказывают, что надо взять представление base-32 от первых 160 бит (полученных путём усечения) хеша sha256, полученного из строки, описанной выше:

```bash
$ nix-hash --type sha256 --truncate --base32 --flat myfile.str
xv2iccirbrvklck36f1g7vldn5v58vck
```

> ## Output paths

## Выходные пути

> Output paths are usually generated for derivations.
> We use the above example because it\'s simple.
> Even if we didn\'t build the derivation, nix knows the out path `hs0yi5n5nw6micqhy8l1igkbhqdkzqa1`.
> This is because the out path only depends on inputs.

Обычно выходные пути генерируется для дериваций.
МЫ использовали пример выше, потому что он простой.
Даже если мы не собрали деривацию, Nix знает, что выходной путь это `hs0yi5n5nw6micqhy8l1igkbhqdkzqa1`.
Это потому, что выходной путь зависит только от входящих параметров.

> It\'s computed in a similar way to source paths, except that the .drv is hashed and the type of derivation is `output:out`.
> In case of multiple outputs, we may have different `output:<id>`.

Он вычисляется почти также, как и пути хранения, за исключением того, что хешируется файл `.drv` и тип деривации — это `output:out`.
В случае нескольких выходящих путей, у нас может быть несколько `output:<id>`.

> At the time nix computes the out path, the .drv contains an empty string for each out path.
> So what we do is getting our .drv and replacing the out path with an empty string:

В тот момент, когда Nix вычисляет выходной путь, в файле `.div` вместо каждого выходящего пути хранится пустая строка.
Так что мы получаем наш `.drv` и заменяем выходящие пути пустой строкой:

```bash
$ cp -f /nix/store/y4h73bmrc9ii5bxg6i7ck6hsf5gqv8ck-foo.drv myout.drv
$ sed -i 's,/nix/store/hs0yi5n5nw6micqhy8l1igkbhqdkzqa1-foo,,g' myout.drv
```

> The `myout.drv` is the .drv state in which nix is when computing the out path for our derivation:

Файл `myout.drv` — это состояние `.drv` перед тем, как Nix вычислит выхоящий путь дя нашей деривации:

```bash
$ sha256sum myout.drv
1bdc41b9649a0d59f270a92d69ce6b5af0bc82b46cb9d9441ebc6620665f40b5  myout.drv
$ echo -n "output:out:sha256:1bdc41b9649a0d59f270a92d69ce6b5af0bc82b46cb9d9441ebc6620665f40b5:/nix/store:foo" > myout.str
$ nix-hash --type sha256 --truncate --base32 --flat myout.str
hs0yi5n5nw6micqhy8l1igkbhqdkzqa1
```

> Then nix puts that out path in the .drv, and that\'s it.

Теперь Nix заишет выходящий путь в файл `.drv` и на этом всё.

> In case the .drv has input derivations, that is it references other .drv, then such .drv paths are replaced by this same algorithm which returns a hash.

В случае, если в `.drv` есть входящие деривации, которые ссылаются на другие `.drv`, вместо путей `.drv` используется хеш, рассчитаный тем же алгоритмом.

> In other words, you get a final .drv where every other .drv path is replaced by its hash.

Иными словами, вы получается окончательный файл `.drv`, где все другие файлы `.drv` заменены их хешами.

> ## Fixed-output paths

## Фиксированные выходящие пути

> Finally, the other most used kind of path is when we know beforehand an integrity hash of a file.
> This is usual for tarballs.

Наконец, существует ещё один вид пути — это мы заранее знаем хеш целостности файла.
Это обычное дело для tar-архивов.

> A derivation can take three special attributes: `outputHashMode`, `outputHash` and `outputHashAlgo` which are well documented in the [nix manual](https://nixos.org/manual/nix/stable/expressions/advanced-attributes.html).

У деривации может быть три специальных атрибута `outputHashMode`, `outputHash` и `outputHashAlgo`, которые хорошо документированы в [руководстве Nix](https://nixos.org/manual/nix/stable/expressions/advanced-attributes.html).

> The builder must create the out path and make sure its hash is the same as the one declared with `outputHash`.

Скрипт сборки должен создать выходной путь и убедиться, что его хеш совпадает с тем, что указан в атрибуте `outputHash`.

> Let\'s say our builder should create a file whose contents is `mycontent`:

Допустим, наш скрипт должен создать файл, который содержит строку `mycontent`:

```bash
$ echo mycontent > myfile
$ sha256sum myfile
f3f3c4763037e059b4d834eaf68595bbc02ba19f6d2a500dce06d124e2cd99bb  myfile
nix-repl> derivation { name = "bar"; system = "x86_64-linux"; builder = "none"; outputHashMode = "flat"; outputHashAlgo = "sha256"; outputHash = "f3f3c4763037e059b4d834eaf68595bbc02ba19f6d2a500dce06d124e2cd99bb"; }
«derivation /nix/store/ymsf5zcqr9wlkkqdjwhqllgwa97rff5i-bar.drv»
```

> Inspect the .drv and see that it also stored the fact that it\'s a fixed-output derivation with sha256 algorithm, compared to the previous examples:

Проверьте `.drv` и убедитесь, что в деривации действительно есть информация о фиксированном выходящем пути и о хеше, вычисленном по алгоритму sha256, в отличие от предыдущим примеров:

```bash
$ nix derivation show /nix/store/ymsf5zcqr9wlkkqdjwhqllgwa97rff5i-bar.drv
{
  "/nix/store/ymsf5zcqr9wlkkqdjwhqllgwa97rff5i-bar.drv": {
    "outputs": {
      "out": {
        "path": "/nix/store/a00d5f71k0vp5a6klkls0mvr1f7sx6ch-bar",
        "hashAlgo": "sha256",
        "hash": "f3f3c4763037e059b4d834eaf68595bbc02ba19f6d2a500dce06d124e2cd99bb"
      }
    },
[...]
}
```

> It doesn\'t matter which input derivations are being used, the final out path must only depend on the declared hash.

Неважно, какие входные деривации используются, окончательный выходной путь будет зависеть только от объявленного хеша.

> What nix does is to create an intermediate string representation of the fixed-output content:

Nix создаёт промежуточное строковое представление для содержимого с фиксированным выходным путём:

```bash
$ echo -n "fixed:out:sha256:f3f3c4763037e059b4d834eaf68595bbc02ba19f6d2a500dce06d124e2cd99bb:" > mycontent.str
$ sha256sum mycontent.str
423e6fdef56d53251c5939359c375bf21ea07aaa8d89ca5798fb374dbcfd7639  myfile.str
```

> Then proceed as it was a normal derivation output path:

Затем обрабатывает его так же, как и любой другой выходной путь деривации:

```bash
$ echo -n "output:out:sha256:423e6fdef56d53251c5939359c375bf21ea07aaa8d89ca5798fb374dbcfd7639:/nix/store:bar" > myfile.str
$ nix-hash --type sha256 --truncate --base32 --flat myfile.str
a00d5f71k0vp5a6klkls0mvr1f7sx6ch
```

> Hence, the store path only depends on the declared fixed-output hash.

Как видим, путь хранения зависит только от фиксированного выходного хеша.

> ## Conclusion

## Заключение

> There are other types of store paths, but you get the idea.
> Nix first hashes the contents, then creates a string description, and the final store path is the hash of this string.

Существуют и другие типы путей хранения, но главное, вы поняли идею.
Сначала Nix хеширует содержимое, затем создаёт строковое представление, а окончательный путь хранения — это хеш строкового представления.

> Also we\'ve introduced some fundamentals, in particular the fact that Nix knows beforehand the out path of a derivation since it only depends on the inputs.
> We\'ve also introduced fixed-output derivations which are especially used by the nixpkgs repository for downloading and verifying source tarballs.

Кроме того, мы рассказали о нескольких фундаметальных вещах, в частности, что Nix заранее знает выходящий путь деривации, поскольку он зависит только от входных параметров.
Мы также рассказали о деривациях с фиксированным выходным путём, которые используются репозиторием `nixpkgs` в частности для загрузки и проверки tar-архивов с исходниками.

> ## Next pill

## В следующей пилюле

> \...we will introduce `stdenv`.
> In the previous pills we rolled our own `mkDerivation` convenience function for wrapping the builtin derivation, but the `nixpkgs` repository also has its own convenience functions for dealing with autotools projects and other build systems.

...мы расскажем про `stdenv`.
В предыдущих пилюлях мы разработали собственную удобную функцию `mkDerivation` для создания дериваций, но в репозитории `nixpkgs` также есть есть удобвные функции для компиляции проектов `autotools` и других систем сборки.
