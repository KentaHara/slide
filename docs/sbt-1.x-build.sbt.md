# 初心者でも分かるbuild.sbtのsetting/task

---

![](img/face.jpg)

## 原賢太 / Hara Kenta
1991.6.14
#### 2014.3 九州工業大学 卒業
#### 2016.3 九州工業大学大学院 修了
#### 現在 株式会社ファンコミュニケーションズ


[<img src="img/twitter.png"  style="width:55px; margin:0; border:none"> chara06ken](https://twitter.com/chara06ken) /
[<img src="img/github.svg"   style="width:55px; margin:0; border:none"> KentaHara](https://github.com/KentaHara) /
[<img src="img/facebook.png" style="width:55px; margin:0; border:none">](https://www.facebook.com/blanc.et.noir.rinc)

---

## 目的

`build.sbt` を理解した上で記述できるようになれたら嬉しい

--

`build.sbt`ってなんとなく書けば動きますよね

--

良さでもあります

--

でも、中身を理解して使えたほうが

かっこいいですよね

---

## 目次

`SettingKey[T]`、`TaskKey[T]`の基本事項

▼

`build.sbt`の呼び出しについて（単純な追っかけ）

▼

`Setting[T]`と`SettingKey[T]`、`TaskKey[T]`の関係

▼

KeyのRankについて

---

## 前提条件

```scala
sbt.version=1.1.4
```

---

# `SettingKey[T]`
# `TaskKey[T]`
# の基本事項

--

## [DSL(Domain-Specific Language)](https://www.scala-sbt.org/1.0/docs/Basic-Def.html)

|例|名称|
|---:|:---|
|`organization`|key|
|`:=`|operator|
|`{"com.example"}`|(setting/task) body|

```scala
organization := {"com.example"}
```

--

## operator

|key| |
|---:|:---|
|`:=`|値の置換|
|`+=`|値の追加|
|`++=`|複数の値の追加|

--

## example

```scala
name := "scala-examples"
scalaVersion := "2.12.3"

scalacOptions ++= Seq(
    // ...
)
```

---

# keyの種類について

--

## Keys

* [`SettingKey[T]`](https://www.scala-sbt.org/1.x/api/sbt/SettingKey.html)
    * サブプロジェクト読み込み時に値を保持
* [`TaskKey[T]`](https://www.scala-sbt.org/1.x/api/sbt/TaskKey.html)
    * compile or package時に値を計算
    * [Task vs Setting keys](https://www.scala-sbt.org/1.0/docs/Basic-Def.html#Task+vs+Setting+keys)
* [`InputKey[T]`](https://www.scala-sbt.org/1.x/api/sbt/InputKey.html)
    * コマンドライン引数を入力として持つタスクキー


--

## [Default Keys](https://www.scala-sbt.org/1.x/api/sbt/Keys$.html)

```scala
name := "scala-examples"
libraryDependencies ++= Seq()

scalacOptions ++= Seq()
```

```scala
// settingKey
val name =
  settingKey[String]("Project name.").withRank(APlusSetting)

val libraryDependencies =
  settingKey[Seq[ModuleID]]("Declares managed dependencies.").withRank(APlusSetting)


// taskKey
val scalacOptions =
  taskKey[Seq[String]]("Options for the Scala compiler.").withRank(BPlusTask)
```

--

## Custom Keys

一般的に、初期化順問題を避けるために
valの代わりにlazy valが用いられることが多い

```scala
import sbt.Keys._
lazy val hello = taskKey[Unit]("An example task")
```

--

## Build Definition

`settings` 内に記載

```scala
lazy val root = (project in file("."))
.settings(
  name := "Hello",
  scalaVersion := "2.12.3"
)
```
--

## `settings`

`Def.Setting[_]`により制御
```scala
// sbt.Project
def settings: Seq[Setting[_]]

def settings(ss: Def.SettingsDefinition*): Project =
      copy(settings = (settings: Seq[Def.Setting[_]]) ++ Def.settings(ss: _*))
```

---

# `sbt`

StateとCommandの詳細については話しません

| |ざっくり説明|
|---:|:---|
|State|sbt内でCommandの逐次処理を制御するもの|
|Command|sbt内での実行単位|

--

## `build.sbt`はどのタイミング読み込まれる？

`sbt XXX`は[sbt-launch.jar](https://github.com/sbt/launcher)から[sbt](https://github.com/sbt/sbt)の`xMain`関数を実行

`sbt-launch.jar` -> `sbt` としての設定ファイルは
[sbt.boot.properties](https://github.com/sbt/sbt/blob/1.x/launch/src/main/input_resources/sbt/sbt.boot.properties)参照

--

## `xMain`

`BootCommand` により、sbt起動時にprojectの読み込み

```scala
/** This class is the entry point for sbt. */
final class xMain extends xsbti.AppMain {
  def run(configuration: xsbti.AppConfiguration): xsbti.MainResult = {
    import BasicCommands.early
    import BasicCommandStrings.runEarly
    import BuiltinCommands.defaults
    import sbt.internal.CommandStrings.{
      BootCommand, DefaultsCommand, InitCommand
    }
    val state = initialState(
      configuration,
      Seq(defaults, early),
      runEarly(DefaultsCommand) :: runEarly(InitCommand) :: BootCommand :: Nil
    )
    runManaged(state)
  }
}
```

--

## `BootCommand`

LoadProjectによりProject関連を読み込み

```scala
// sbt/internal/CommandStrings.scala
val BootCommand = "boot"

// sbt/Main.scala
def boot = Command.make(BootCommand)(bootParser)

def bootParser(s: State) = {
  val orElse = () => DefaultBootCommands.toList ::: s
  delegateToAlias(BootCommand, success(orElse))(s)
}

def DefaultBootCommands: Seq[String] =
  WriteSbtVersion :: LoadProject :: NotifyUsersAboutShell :: s"$IfLast $Shell" :: Nil
```

--

## `LoadProject`

`$ sbt reload` と打てば動作確認可能

```scala
// sbt/internal/CommandStrings.scala
def LoadProject = "reload"

def loadProject: Command =
  Command(LoadProject, LoadProjectBrief, LoadProjectDetailed)(loadProjectParser)(
    (s, arg) => loadProjectCommands(arg) ::: s
  )

private[this] def loadProjectParser: State => Parser[String] =
  _ => matched(Project.loadActionParser)

// sbt/Project.scala
val loadActionParser = token(Space ~> ("plugins" ^^^ Plugins | "return" ^^^ Return)) ?? Current
```

--

## `loadProjectCommands`

`loadProjectCommand`で実際に読み込み

```scala
def loadProjectCommands(arg: String): List[String] =
  StashOnFailure ::
    (OnFailure + " " + loadProjectCommand(LoadFailed, arg)) ::
    loadProjectCommand(LoadProjectImpl, arg) ::
    PopOnFailure ::
    State.FailureWall ::
    Nil
```

--

## `loadProjectCommand`

|param| |
| ---: | :--- |
|command| `LoadFailed` or `LoadProjectImpl`|
| arg   | `plugins` or `returns`|

```scala
private[this] def loadProjectCommand(command: String, arg: String): String =
  s"$command $arg".trim
```

--

## `LoadProjectImpl`

keyの呼び出しは`Load.defaultLoad`

```scala
// sbt/Main.scala
def loadProjectImpl: Command =
  Command(LoadProjectImpl)(_ => Project.loadActionParser)(doLoadProject)

def doLoadProject(s0: State, action: LoadAction.Value): State = {
  checkSBTVersionChanged(s0)
  val (s1, base) = Project.loadAction(SessionVar.clear(s0), action)
  IO.createDirectory(base)
  val s = if (s1 has Keys.stateCompilerCache) s1 else registerCompilerCache(s1)

  val (eval, structure) =
    try Load.defaultLoad(s, base, s.log, Project.inPluginProject(s), Project.extraBuilds(s))
    catch {
      //..
    }
  val session = Load.initialSession(structure, eval, s0)
  SessionSettings.checkSession(session, s)
  Project.setProject(session, structure, s)
}
```

--

## `Load.defaultLoad`

`apply(base, state, config)`

```scala
// sbt.Load
def defaultLoad(
    state: State,
    baseDirectory: File,
    log: Logger,
    isPlugin: Boolean = false,
    topLevelExtras: List[URI] = Nil
): (() => Eval, BuildStructure) = {
  // ...
  val result = apply(base, state, config)
  result
}
```

--

## `Load.defaultLoad.apply`

<img src="https://www.scala-sbt.org/1.0/docs/files/settings-initialization-load-ordering.png" style="width:710px; margin:0; border:none">

[sbt: Setting Initialization](https://www.scala-sbt.org/1.0/docs/Setting-Initialization.html)より引用

--

## `Load.resolveProject -> expandSettings`

型`Setting`により各種設定を読み込み

```scala
// Expand the AddSettings instance into a real Seq[Setting[_]] we'll use on the project
def expandSettings(auto: AddSettings): Seq[Setting[_]] = auto match {
  case BuildScalaFiles     => p.settings
  case User                => globalUserSettings.cachedProjectLoaded(loadedPlugins.loader)
  case sf: SbtFiles        => settings(sf.files.map(f => IO.resolve(p.base, f)))
  case sf: DefaultSbtFiles => settings(defaultSbtFiles.filter(sf.include))
  case p: AutoPlugins      => autoPluginSettings(p)
  case q: Sequence =>
    (Seq.empty[Setting[_]] /: q.sequence) { (b, add) =>
      b ++ expandSettings(add)
    }
}

expandSettings(AddSettings.allDefaults)
```

--

## `AddSettings`

```scala
/** The default inclusion of settings. */
val allDefaults: AddSettings = seq(autoPlugins, buildScalaFiles, userSettings, defaultSbtFiles)


/** Adds all settings from autoplugins. */
val autoPlugins : AddSettings = new AutoPlugins(const(true))

/** Settings specified in Build.scala `Project` constructors. */
val buildScalaFiles: AddSettings = BuildScalaFiles

/** Includes user settings in the project. */
val userSettings: AddSettings = User

/** Includes the settings from all .sbt files in the project's base directory. */
val defaultSbtFiles: AddSettings = new DefaultSbtFiles(const(true))
```

--

## SettingsのLoad時info

```
[info] Loading settings from idea.sbt ...

[info] Loading global plugins from /Users/k_hara/.sbt/1.0/plugins
[info] Loading settings from assembly.sbt,plugins.sbt ...
[info] Loading project definition from /Users/k_hara/example/scala-examples/project
[info] Loading settings from build.sbt ...
```

---

# Settingとは？

```
build.sbt --load--> Setting
```

--

## `Setting`

```scala
sealed trait SettingsDefinition {
  def settings: Seq[Setting[_]]
}

sealed class Setting[T] private[Init] (
    val key: ScopedKey[T],
    val init: Initialize[T],
    val pos: SourcePosition
) extends SettingsDefinition {
  def settings = this :: Nil
  // ...
}
```

| type |  |
|---:|:---|
|ScopedKey| Scope + AttributeKeyのcase class|
|Initialize|[タスクグラフ](https://www.scala-sbt.org/1.x/docs/Task-Graph.html)のノードを定義|
|SourcePosition|File pathやFileのLineを保持|

--

## `SettingKey`

macroの話は少し重たいので

Setting typeとなると覚えていただきたい...

```scala
sealed abstract class SettingKey[T]
    extends ScopedTaskable[T]
    with KeyedInitialize[T]
    with Scoped.ScopingSetting[SettingKey[T]]
    with Scoped.DefinableSetting[T] {
// ...
  final def :=(v: T): Setting[T] = macro std.TaskMacro.settingAssignMacroImpl[T]

  final def +=[U](v: U)(implicit a: Append.Value[T, U]): Setting[T] =
    macro std.TaskMacro.settingAppend1Impl[T, U]

  final def ++=[U](vs: U)(implicit a: Append.Values[T, U]): Setting[T] =
    macro std.TaskMacro.settingAppendNImpl[T, U]
// ...
}
```

--

## `TaskKey`

```scala
sealed abstract class TaskKey[T]
    extends ScopedTaskable[T]
    with KeyedInitialize[Task[T]]
    with Scoped.ScopingSetting[TaskKey[T]] {
// ...
  def +=[U](v: U)(implicit a: Append.Value[T, U]): Setting[Task[T]] =
    macro std.TaskMacro.taskAppend1Impl[T, U]

  def ++=[U](vs: U)(implicit a: Append.Values[T, U]): Setting[Task[T]] =
    macro std.TaskMacro.taskAppendNImpl[T, U]
// ...
```

--

## example

```scala
// setting
name := "scala-examples"
scalaVersion := "2.12.3"

// task
scalacOptions ++= Seq(
    // ...
)
```

---

# [KeyRanks](https://www.scala-sbt.org/1.x/api/sbt/KeyRanks$.html)


```scala
val name =
  settingKey[String]("Project name.").withRank(APlusSetting)
```

> task and setting ranks, used to prioritize displaying information

--

## ranks

|Key/Rank|A|B|C|D|
|---:|:---:|:---:|:---:|:---:|
|Task   |5|30|200|20000|
|Setting|10|40|100|10000|

--

## default

```scala
// Rank:17.5 = (5 + 30)/2
final val DefaultTaskRank = (ATask + BTask) / 2

// Rank:5
final val DefaultInputRank = ATask // input tasks are likely a main task

// Rank:25 = (10 + 40)/2
final val DefaultSettingRank = (ASetting + BSetting) / 2
```

--

## `AttributeKey[T]`


他のキーの中でキーの相対的な重要性を識別

```scala
sealed trait AttributeKey[T] {
// ...
  /** Identifies the relative importance of a key among other keys.*/
  def rank: Int
// ...
}

```

--

## `tasks`/`settings` command

| |表示対象rank|
|---:|:---:|
|tasks|6(AMinusTask)以上|
|settings|11(AMinusSetting)以上|

```bash
$ sbt "help settings"
...
-v
  Displays additional settings.
  More 'v's increase the number of settings displayed.
...
$ sbt "settings -vvv" # v*25個表示
```

---

# まとめ

`build.sbt` を理解した上で記述できるようになれたら嬉しい

--

## `SettingKey[T]`、`TaskKey[T]`の基本事項

```scala
organization := {"com.example"}
```

--

## `build.sbt`の呼び出しについて

<img src="https://www.scala-sbt.org/1.0/docs/files/settings-initialization-load-ordering.png" style="width:710px; margin:0; border:none">


--

## `Setting[T]`と`SettingKey[T]`、`TaskKey[T]`の関係

Keyから`Setting[T]`として呼び出し

```scala
sealed abstract class SettingKey[T]
// ...
  final def :=(v: T): Setting[T] = macro std.TaskMacro.settingAssignMacroImpl[T]

  final def +=[U](v: U)(implicit a: Append.Value[T, U]): Setting[T] =
    macro std.TaskMacro.settingAppend1Impl[T, U]

  final def ++=[U](vs: U)(implicit a: Append.Values[T, U]): Setting[T] =
    macro std.TaskMacro.settingAppendNImpl[T, U]
// ...
}
```

--

## KeyのRankについて

help時の出力数の制御のために使用されている

```scala
val name =
  settingKey[String]("Project name.").withRank(APlusSetting)
```

---

# 初心者でも分かるbuild.sbtのsetting/task

---

# NOTE

--

## [sbt release](https://github.com/sbt/sbt-release)

projectのversion管理をするplugin

```scala
addSbtPlugin("com.github.gseitz" % "sbt-release" % "1.0.8")
```

```bash
$ sbt release
```


--

## sbtの呼び出しbash

```
# cat /usr/local/bin/sbt

if [ -f "$HOME/.sbtconfig" ]; then
  echo "Use of ~/.sbtconfig is deprecated, please migrate global settings to /usr/local/etc/sbtopts" >&2
  . "$HOME/.sbtconfig"
fi
exec "/usr/local/Cellar/sbt/1.1.4/libexec/bin/sbt" "$@"
```

--

## BootCommandの詳細

- `sbt boot`で呼び出し可能
- 4つのCommandを実行
    - `WriteSbtVersion`: `project.properties`に対してsbt versionの記述をするCommand
    - `LoadProject`: projectに関する読み込み ← `build.sbt` の読み込みもこのCommand
    - `NotifyUsersAboutShell`: [Batch mode](https://www.scala-sbt.org/1.x/docs/Running.html)で使用するかどうかを通知するCommand
        - compileを実行する場合
        - build.sbtで`suppressSbtShellNotification := true`を宣言
    - `s"$IfLast $Shell"`
        - `sbt`のみで実行した場合は、sbtのshell modeを実行するCommand

--

## [sbt/Defaults](https://www.scala-sbt.org/1.1.2/api/sbt/Defaults$.html)

sbt起動時にsbtに記述されているsettingsを呼び出すために使用
```scala
object Defaults extends BuildCommon
```

--

## KeyRanks

```scala
// main settings
final val APlusSetting = 9
final val ASetting = 10
final val AMinusSetting = 11

// secondary settings
final val BPlusSetting = 39
final val BSetting = 40
final val BMinusSetting = 41

// advanced settings
final val CSetting = 100

// explicit settings
final val DSetting = 10000
```

--

## `runEarly(DefaultsCommand)`

definedCommandsに設定しているだけ

```scala
def defaults = Command.command(DefaultsCommand) { s =>
  s.copy(definedCommands = DefaultCommands)
}
```

--

## `runEarly(InitCommand)`

file読み込みからinit command

```scala
def initialize: Command = Command.command(InitCommand) { s =>
  /*"load-commands -base ~/.sbt/commands" :: */
  readLines(readable(sbtRCs(s))).toList ::: s
}
```

--
## `tasks`/`settings` command

```scala
// sbt.Main
def sortByRank(keys: Seq[AttributeKey[_]]): Seq[AttributeKey[_]] = keys.sortBy(_.rank)

def topNRanked(n: Int) = (keys: Seq[AttributeKey[_]]) => sortByRank(keys).take(n)
def highPass(rankCutoff: Int) =
  (keys: Seq[AttributeKey[_]]) => sortByRank(keys).takeWhile(_.rank <= rankCutoff)
```
