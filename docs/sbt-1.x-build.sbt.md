# 初心者でも分かるbuild.sbtのsettings/task expression

---

## 目的

* `build.sbt` を理解した上で記述できるようになれたら嬉しい

---

## 目次

* `SettingKey[T]` 、 `TaskKey[T]` 、 `InputKey[T]` の性質
* Keys.scalaに関して
* 余力があれば、その他Libraryなど紹介

---

## Information

```scala
sbt.version=1.1.4
```

---

## `build.sbt`はどのタイミング読み込まれる？

`sbt XXX`は[sbt-launch.jar](https://github.com/sbt/launcher)から[sbt](https://github.com/sbt/sbt)の`xMain`関数を実行

`sbt-launch.jar` -> `sbt` としての設定ファイルは
[sbt.boot.properties](https://github.com/sbt/sbt/blob/1.x/launch/src/main/input_resources/sbt/sbt.boot.properties)参照

--

## `xMain`

`BootCommand` によって、sbt起動時にprojectの読み込みを実施

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

### `BootCommand`

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

### `LoadProject`

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

### `loadProjectCommands`

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

### `loadProjectCommand`

- command
    - `LoadFailed`
    - `LoadProjectImpl`
- arg
    - Project.Value
        - `plugins`
        - `returns`

```scala
private[this] def loadProjectCommand(command: String, arg: String): String =
  s"$command $arg".trim
```

--

### `LoadProjectImpl` Command

`settingKey`のdefaultの呼び出しは`Load.defaultLoad`によって成される

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
      case ex: compiler.EvalException =>
        s0.log.debug(ex.getMessage)
        ex.getStackTrace map (ste => s"\tat $ste") foreach (s0.log.debug(_))
        ex.setStackTrace(Array.empty)
        throw ex
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
// note that there is State passed in but not pulled out
def defaultLoad(
    state: State,
    baseDirectory: File,
    log: Logger,
    isPlugin: Boolean = false,
    topLevelExtras: List[URI] = Nil
): (() => Eval, BuildStructure) = {
  val (base, config) = timed("Load.defaultLoad until apply", log) {
    val globalBase = getGlobalBase(state)
    val base = baseDirectory.getCanonicalFile
    val rawConfig = defaultPreGlobal(state, base, globalBase, log)
    val config0 = defaultWithGlobal(state, base, rawConfig, globalBase, log)
    val config =
      if (isPlugin) enableSbtPlugin(config0) else config0.copy(extraBuilds = topLevelExtras)
    (base, config)
  }
  val result = apply(base, state, config)
  result
}
```

--

## `Load.defaultLoad.apply`

![Load.defaultLoad.apply](https://www.scala-sbt.org/1.0/docs/files/settings-initialization-load-ordering.png)

[sbt: Setting Initialization](https://www.scala-sbt.org/1.0/docs/Setting-Initialization.html)より引用

--

## `Load.resolveProject -> expandSettings`

`AddSettings.allDefaults`によりloadするファイル群を設定している
型`Setting`により各種設定を読み込んでいる

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

## Settingとは？

build.sbt --load--> Setting

--

### `sealed class Setting[T]`

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



---

## Build Definition

* `settings` 内に記載

```scala
lazy val root = (project in file("."))
.settings(
  name := "Hello",
  scalaVersion := "2.12.3"
)
```
--

## `settings`

* `sbt/Project.scala`

```scala
def settings: Seq[Setting[_]]
def settings(ss: Def.SettingsDefinition*): Project =
      copy(settings = (settings: Seq[Def.Setting[_]]) ++ Def.settings(ss: _*))
```

--

## `Setting`

* `Import.scala`

```scala
type Setting[T] = Def.Setting[T]
```
--

## `Setting`

* `sbt/internal/util/Settings.scala`

```scala
sealed trait Settings[Scope] {
  def data: Map[Scope, AttributeMap]
  def keys(scope: Scope): Set[AttributeKey[_]]
  def scopes: Set[Scope]
  def definingScope(scope: Scope, key: AttributeKey[_]): Option[Scope]
  def allKeys[T](f: (Scope, AttributeKey[_]) => T): Seq[T]
  def get[T](scope: Scope, key: AttributeKey[T]): Option[T]
  def getDirect[T](scope: Scope, key: AttributeKey[T]): Option[T]
  def set[T](scope: Scope, key: AttributeKey[T], value: T): Settings[Scope]
}
```

--

## `Def`

```scala
object Def extends Init[Scope] with TaskMacroExtra
```

---

## DSL(Domain-Specific Language)

```scala
organization := {"com.example"}
```

* `organization` : key
* `:=` : opretator
* `{"com.example"}` : (settings/task) body

---

## Keys

* [`SettingKey[T]`](https://www.scala-sbt.org/1.x/api/sbt/SettingKey.html)
    * サブプロジェクト読み込み時に値を保持
* [`TaskKey[T]`](https://www.scala-sbt.org/1.x/api/sbt/TaskKey.html)
    * 毎回値を計算（副作用あり）
* [`InputKey[T]`](https://www.scala-sbt.org/1.x/api/sbt/InputKey.html)
    * コマンドライン引数を入力として持つタスクキー

--

## Custom Keys

```scala
import sbt.Keys._
lazy val hello = taskKey[Unit]("An example task")
```

> 一般的に、初期化順問題を避けるために val の代わりに lazy val が用いられることが多い。

--

## [Default Keys](https://www.scala-sbt.org/1.x/api/sbt/Keys$.html)

--

## [KeyRanks](https://www.scala-sbt.org/1.x/api/sbt/KeyRanks$.html)

* 何に使われているか調査

ranks

|Key/Rank|A|B|C|D|
|---:|:---:|:---:|:---:|:---:|
|Task   |5|30|200|20000|
|Setting|10|40|100|10000|

default

```scala
// Rank:17.5 = (5 + 30)/2
final val DefaultTaskRank = (ATask + BTask) / 2

// Rank:5
final val DefaultInputRank = ATask // input tasks are likely a main task

// Rank:25 = (10 + 40)/2
final val DefaultSettingRank = (ASetting + BSetting) / 2
```

---

## Task Graph

* task graphとはなんぞや

---

## Build Syntax

```scala
private[sbt] trait BuildSyntax {
  import language.experimental.macros
  def settingKey[T](description: String): SettingKey[T] = macro std.KeyMacro.settingKeyImpl[T]
  def taskKey[T](description: String): TaskKey[T] = macro std.KeyMacro.taskKeyImpl[T]
  def inputKey[T](description: String): InputKey[T] = macro std.KeyMacro.inputKeyImpl[T]

  def enablePlugins(ps: AutoPlugin*): DslEntry = DslEntry.DslEnablePlugins(ps)
  def disablePlugins(ps: AutoPlugin*): DslEntry = DslEntry.DslDisablePlugins(ps)
  def configs(cs: Configuration*): DslEntry = DslEntry.DslConfigs(cs)
  def dependsOn(deps: ClasspathDep[ProjectReference]*): DslEntry = DslEntry.DslDependsOn(deps)
  // avoid conflict with `sbt.Keys.aggregate`
  def aggregateProjects(refs: ProjectReference*): DslEntry = DslEntry.DslAggregate(refs)
}
private[sbt] object BuildSyntax extends BuildSyntax
```


---

# NOTE

--

```
# cat /usr/local/bin/sbt

if [ -f "$HOME/.sbtconfig" ]; then
  echo "Use of ~/.sbtconfig is deprecated, please migrate global settings to /usr/local/etc/sbtopts" >&2
  . "$HOME/.sbtconfig"
fi
exec "/usr/local/Cellar/sbt/1.1.4/libexec/bin/sbt" "$@"
```

--

### `BootCommand` - 2

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

```scala
object Defaults extends BuildCommon
```
