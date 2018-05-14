# (執筆中）sbtは何故起動するのか

---

## 背景

* sbtのlibraryを読んでいる中で、何が読まれてどんな動作でsbtが立ち上がるのかわからなくなったため

---

## TL; DR

* [sbt-launcher-package](https://github.com/sbt/sbt-launcher-package)が基本
    * `src/universal` をベースとして読み進めるのが良さそう

---

## `which sbt`

* コマンドの所在
    ```bash
    $ which sbt
    /usr/local/bin/sbt
    ```

---

## `cat /usr/local/bin/sbt` 

* コマンド内部
    ```bash
    $ cat /usr/local/bin/sbt
    #!/bin/sh
    if [ -f "$HOME/.sbtconfig" ]; then
        echo "Use of ~/.sbtconfig is deprecated, please migrate global settings to /usr/local/etc/sbtopts" >&2
        . "$HOME/.sbtconfig"
    fi
    exec "/usr/local/Cellar/sbt/1.1.4/libexec/bin/sbt" "$@"
    ```
    * `$HOME/.sbtconfig` が存在する場合は、コマンド呼び出し時にconfigを実行する
    * `exec` コマンドを実行して終了する
    * `$@` 引数全てを渡す（今回の場合はsbtコマンドに対しての引数がそのまま渡される）

---

## `cat sbt/1.1.4/libexec/bin/sbt` 

* dir
    * `/usr/local/Cellar/sbt/1.1.4/libexec/bin/`

--

## function

* `realpath()`
    * 対象とするファイルを変数 `TARGET_FILE` と個数 `COUNT` に入れる
    * `echo "$(pwd -P)/$TARGET_FILE"` 
* `is_cygwin()`
    * Cygwinであれば0、それ以外1を返す
* `cygwinpath()`
    * Cygwin特有のファイルpathをechoする

--
## function

* `usage()`
    * 使用方法等
    * どこか他でまとめたい
* `process_my_args()`
    * parameter指定によるjavaに対する変数宣言等を実施
        * どこか他でまとめたい
    * `[[ "${sbt_version}XXX" != "XXX" ]] && addJava "-Dsbt.version=$sbt_version"`
* `loadConfigFile()`
    * evalしている

--

## libraryの読みこみ


```
. "$(dirname "$(realpath "$0")")/sbt-launch-lib.bash"
```


--

## declare

```
declare -r noshare_opts="-Dsbt.global.base=project/.sbtboot -Dsbt.boot.directory=project/.boot -Dsbt.ivy.home=project/.ivy"
declare -r sbt_opts_file=".sbtopts"
declare -r etc_sbt_opts_file="/usr/local/etc/sbtopts"
declare -r dist_sbt_opts_file="${sbt_home}/conf/sbtopts"
declare -r win_sbt_opts_file="${sbt_home}/conf/sbtconfig.txt"
```

--

## bash下部

```bash
# Here we pull in the default settings configuration.
[[ -f "$dist_sbt_opts_file" ]] && set -- $(loadConfigFile "$dist_sbt_opts_file") "$@"

# Here we pull in the global settings configuration.
[[ -f "$etc_sbt_opts_file" ]] && set -- $(loadConfigFile "$etc_sbt_opts_file") "$@"

#  Pull in the project-level config file, if it exists.
[[ -f "$sbt_opts_file" ]] && set -- $(loadConfigFile "$sbt_opts_file") "$@"

#  Pull in the project-level java config, if it exists.
[[ -f ".jvmopts" ]] && export JAVA_OPTS="$JAVA_OPTS $(loadConfigFile .jvmopts)"

run "$@"
```
    * what's a `run` command
        * `sbt-launch-lib.bash` にて定義されている

---

## cat `sbt-launch-lib.bash`

諸々が動いている定義

* dir
    * `/usr/local/Cellar/sbt/1.1.4/libexec/bin/`
* [sbt-launch.jar](https://github.com/sbt/launcher)でのLibrary読み込み後、sbtのコマンドを実行

--

## jarの展開

`jar -xvf sbt-launch.jar`

```
$ cat module.properties
version=2.3.0-sbt-b18f59ea3bc914a297bb6f1a4f7fb0ace399e310
```
    * [ivy](https://github.com/sbt/ivy)のversion

--

## [ivy](http://ant.apache.org/ivy/)

> The agile dependency manager

* Mavenの機能の中からビルド機能やプロジェクト管理機能を無くして、ライブラリーの依存関係の管理に特化したツール


---

## [launcher](https://github.com/sbt/launcher)

* xsbtは複数versionのScalaコンパイラに依存
* xsbtiはxsbtとsbtの中間役であり、Javaで書かれている

--

## Package `xsbt.boot.Boot`

* launcherのエントリーポイント

> The entry point to the launcher

--

## Package `xsbt.boot.Boot`の内部を見る - 全体像

```scala
def main(args: Array[String]) {
  standBy()
  val config = parseArgs(args)
  // If we havne't exited, we set up some hooks and launch
  System.clearProperty("scala.home") // avoid errors from mixing Scala versions in the same JVM
  System.setProperty("jline.shutdownhook", "false") // shutdown hooks cause class loader leaks
  System.setProperty("jline.esc.timeout", "0") // starts up a thread otherwise
  CheckProxy()
  run(config)
}
```

--

## Package `xsbt.boot.Boot`の内部を見る - `parseArgs`

* parseArgsは引数をLauncherArgumentsにparse
```scala
def parseArgs(args: Array[String]): LauncherArguments = ...
```

```scala
class LauncherArguments(val args: List[String], val isLocate: Boolean)
```
    * `isLocate` は `--locate` parameterが存在しているかどうかを指している

--

## Package `xsbt.boot.Boot` の内部を見る - `run`

* runImplにLauncherArgumentを引き渡し、再帰的にrunさせる
```scala
// this arrangement is because Scala does not always properly optimize away
// the tail recursion in a catch statement
final def run(args: LauncherArguments): Unit = runImpl(args) match {
  case Some(newArgs) => run(newArgs)
  case None          => ()
}
```

--

## Package `xsbt.boot.Boot` の内部を見る - `runImpl`

* `Launch` に投げてエラーハンドリングをしている
```scala
private def runImpl(args: LauncherArguments): Option[LauncherArguments] =
  try
    Launch(args) map exit
  catch {
    case b: BootException           => errorAndExit(b.toString)
    case r: xsbti.RetrieveException => errorAndExit("Error: " + r.getMessage)
    case r: xsbti.FullReload        => Some(new LauncherArguments(r.arguments.toList, false))
    case e: Throwable =>
    e.printStackTrace
    errorAndExit(Pre.prefixError(e.toString))
  }
```

--

## Package `xsbt.boot.Boot` の内部を見る - `Launch`

* ファイルや引数を利用してLaunchを生成可能
```scala
def apply(arguments: LauncherArguments): Option[Int] = apply((new File("")).getAbsoluteFile, arguments)
def apply(currentDirectory: File, arguments: LauncherArguments): Option[Int] = ...
```
    * ちなみに、後者のapplyは他からの呼び出しはされていない

--

## Package `xsbt.boot.Boot` の内部を見る - `Launch.apply`

```scala
def apply(currentDirectory: File, arguments: LauncherArguments): Option[Int] = {
  val (configLocation, newArgs2, state) = Configuration.find(arguments.args, currentDirectory)
  val config = state match {
    case SerializedFile => LaunchConfiguration.restore(configLocation)
    case PropertiesFile => parseAndInitializeConfig(configLocation, currentDirectory)
  }
  if (arguments.isLocate) {
    if (!newArgs2.isEmpty) {
      // TODO - Print the arguments without exploding proguard size.
      System.err.println("Warning: --locate option ignores arguments.")
    }
    locate(currentDirectory, config)
  } else {
    // First check to see if there are java system properties we need to set. Then launch the application.
    updateProperties(config)
    launch(run(Launcher(config)))(makeRunConfig(currentDirectory, config, newArgs2))
  }
}
```

--

## Package `xsbt.boot.Boot` の内部を見る - `Configuration.find`

`-D` option や `sbt.boot.properties` を呼び出すために使用している

```scala
@tailrec def find(args: List[String], baseDirectory: File): (URL, List[String], ConfigurationStorageState.Value) =
  args match {
    case head :: tail if head.startsWith("@load:") => (directConfiguration(head.substring(6), baseDirectory), tail, SerializedFile)
    case head :: tail if head.startsWith("@")      => (directConfiguration(head.substring(1), baseDirectory), tail, PropertiesFile)
    case head :: tail if head.startsWith(SysPropPrefix) =>
      setProperty(head stripPrefix SysPropPrefix)
      find(tail, baseDirectory)
    case _ =>
      val propertyConfigured = System.getProperty("sbt.boot.properties")
      val url = if (propertyConfigured == null) configurationOnClasspath else configurationFromFile(propertyConfigured, baseDirectory)
      (url, args, PropertiesFile)
  }
```

* `case head :: tail if head.startsWith("@load:")     ` => `@load:XXXXX`によりserialize fileを設定
* `case head :: tail if head.startsWith("@")          ` => `@:XXXXX`によりproperties fileを設定
* `case head :: tail if head.startsWith(SysPropPrefix)` => `-DXXXXX`によりsys.propsを設定でき、`@load` や `@` を後述可能
* `case _                                             ` => `sbt.boot.properties`をファイル名とするproperies fileを設定

--

## Package `xsbt.boot.Boot` の内部を見る - `Launch.apply -> config`

* SerializedされたファイルもしくはPropertiesFileからpropertiesを引っ張り出す
```scala
val config = state match {
  case SerializedFile => LaunchConfiguration.restore(configLocation)
  case PropertiesFile => parseAndInitializeConfig(configLocation, currentDirectory)
}
```

* ProperiesFileについては[sbtのsbt.boot.properties](https://github.com/sbt/sbt/blob/1.x/launch/src/main/input_resources/sbt/sbt.boot.properties)を参照

--

## Package `xsbt.boot.Boot` の内部を見る - `LaunchConfiguration`

* Launchで設定できるConfig一覧
```scala
final case class LaunchConfiguration(
  scalaVersion: Value[String],
  ivyConfiguration: IvyOptions,
  app: Application,
  boot: BootSetup,
  logging: Logging,
  appProperties: List[AppProperty],
  serverConfig: Option[ServerConfiguration]
)
```


--

## Package `xsbt.boot.Boot` の内部を見る - `Launch.apply -> launch`
```scala
launch(
  run(Launcher(config))
)(
  makeRunConfig(currentDirectory, config, newArgs2)
)
```

--

## Package `xsbt.boot.Boot` の内部を見る - `Launch -> launch`

* `run(config)` を実行して、その結果により諸々処理する
```scala
final def launch(run: RunConfiguration => xsbti.MainResult)(config: RunConfiguration): Option[Int] =
  {
    run(config) match {
      case e: xsbti.Exit     => Some(e.code)
      case c: xsbti.Continue => None
      case r: xsbti.Reboot   => launch(run)(new RunConfiguration(Option(r.scalaVersion), r.app, r.baseDirectory, r.arguments.toList))
      case x                 => throw new BootException("Invalid main result: " + x + (if (x eq null) "" else " (class: " + x.getClass + ")"))
    }
  }
```

--

## Package `xsbt.boot.Boot` の内部を見る - `Launch -> run`

* sbtの諸々を実行するもの（後述）
```scala
/** The actual mechanism used to run a launched application. */
def run(launcher: xsbti.Launcher)(config: RunConfiguration): xsbti.MainResult =
  {
    import config._
    val appProvider: xsbti.AppProvider = launcher.app(app, orNull(scalaVersion)) // takes ~40 ms when no update is required
    val appConfig: xsbti.AppConfiguration = new AppConfiguration(toArray(arguments), workingDirectory, appProvider)

    // TODO - Jansi probably should be configurable via some other mechanism...
    JAnsi.install(launcher.topLoader)
    try {
      val main = appProvider.newMain()
      try { withContextLoader(appProvider.loader)(main.run(appConfig)) }
      catch { case e: xsbti.FullReload => if (e.clean) delete(launcher.bootDirectory); throw e }
    } finally {
      JAnsi.uninstall(launcher.topLoader)
    }

final class RunConfiguration(
  val scalaVersion: Option[String],
  val app: xsbti.ApplicationID,
  val workingDirectory: File,
  val arguments: List[String]
)
```



--

## 実行方法（documentより）

```
java -jar <raw launch jar> @<my launcher properties file>
```



---

## xsbti, xsbt ([sbt/zinc](https://github.com/sbt/zinc))

ZincはScalaの定期的なコンパイラ

> Zinc is the incremental compiler for Scala


## 元のcompilerはscalacを使用

* [bridge213.sh](https://github.com/sbt/zinc/blob/1.x/bin/bridge213.sh)

```bash
...
"$SCALA_X_HOME/bin/scalac" \
...
```

---

## [`scalac`](https://github.com/scala/scala)

[command](https://github.com/scala/scala/blob/2.13.x/build.sbt#L1068)

```
val commands =
List(("scalac",   "compiler", "scala.tools.nsc.Main"),
    ("scala",    "repl-frontend", "scala.tools.nsc.MainGenericRunner"),
    ("scaladoc", "scaladoc", "scala.tools.nsc.ScalaDoc"))
```
