# 初心者でも分かるbuild.sbtのsettings/task expression

---

## Goal

* `build.sbt` を理解した上で記述できるようになれたら嬉しい

---

## Agenda

* `SettingKey[T]` 、 `TaskKey[T]` 、 `InputKey[T]` の性質
* Keys.scalaに関して
* 余力があれば、その他Libraryなど紹介

---

## Information

```scala
sbt.version=1.1.4
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

* ranks

|Key/Rank|A|B|C|D|
|---:|:---:|:---:|:---:|:---:|
|Task   |5|30|200|20000|
|Setting|10|40|100|10000|

* default

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

## [xsbti](https://www.scala-sbt.org/1.x/api/xsbti/index.html)

---

## [sbt/Defaults](https://www.scala-sbt.org/1.1.2/api/sbt/Defaults$.html)

```scala
object Defaults extends BuildCommon
```

---

## NOTE

```
# cat /usr/local/bin/sbt

if [ -f "$HOME/.sbtconfig" ]; then
  echo "Use of ~/.sbtconfig is deprecated, please migrate global settings to /usr/local/etc/sbtopts" >&2
  . "$HOME/.sbtconfig"
fi
exec "/usr/local/Cellar/sbt/1.1.4/libexec/bin/sbt" "$@"
```
