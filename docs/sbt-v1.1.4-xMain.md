# sbt v1.1.4 xMainの読み込み


---

## Background

launcherからの読み出し先がxMainであり、
その後どのように実行されているか気になるため

---

## `sbt.xMain`

```scala
/** This class is the entry point for sbt. */
final class xMain extends xsbti.AppMain {
  def run(configuration: xsbti.AppConfiguration): xsbti.MainResult = {
    import BasicCommands.early
    import BasicCommandStrings.runEarly
    import BuiltinCommands.defaults
    import sbt.internal.CommandStrings.{ BootCommand, DefaultsCommand, InitCommand }
    val state = initialState(
      configuration,
      Seq(defaults, early),
      runEarly(DefaultsCommand) :: runEarly(InitCommand) :: BootCommand :: Nil)
    runManaged(state)
  }
}
```

--

### `xsbti.AppConfiguration`

```java
public interface AppConfiguration
{
    public String[] arguments();
    public File baseDirectory();
    public AppProvider provider();
}
```

--

### `xsbti.AppProvider`

```java
public interface AppProvider
{
    public ScalaProvider scalaProvider();
    public ApplicationID id();
    public ClassLoader loader();
    @Deprecated
    public Class<? extends AppMain> mainClass();
    public Class<?> entryPoint();
    public AppMain newMain();
    
    public File[] mainClasspath();

    public ComponentProvider components();
}
```

---

## `sbt.BasicCommands.early`

```scala
def early: Command = Command.arb(earlyParser, earlyHelp)((s, other) => other :: s)
```

--

### `sbt.Command.arb`

```scala
def arb[T](
  parser: State => Parser[T],
  help: Help = Help.empty
)(
  effect: (State, T) => State
): Command = custom(applyEffect(parser)(effect), help)
```

--

### `sbt.State`

```scala
final case class State(
    configuration: xsbti.AppConfiguration,
    definedCommands: Seq[Command],
    exitHooks: Set[ExitHook],
    onFailure: Option[Exec],
    remainingCommands: List[Exec],
    history: State.History,
    attributes: AttributeMap,
    globalLogging: GlobalLogging,
    currentCommand: Option[Exec],
    next: State.Next
) extends Identity {
  lazy val combinedParser = Command.combine(definedCommands)(this)

  def source: Option[CommandSource] =
    currentCommand match {
      case Some(x) => x.source
      case _       => None
    }
}
```
--

### `sbt.Command.applyEffect`

```scala
def applyEffect[T](
  p: Parser[T]
)(
  f: T => State
): Parser[() => State] = p map (t => () => f(t))

def applyEffect[T](
  parser: State => Parser[T]
)(
  effect: (State, T) => State
): State => Parser[() => State] =
  s => applyEffect(parser(s))(t => effect(s, t))
```



---

## `sbt.StandardMain.initialState`

```scala
def initialState(configuration: xsbti.AppConfiguration,
                 initialDefinitions: Seq[Command],
                 preCommands: Seq[String]): State = {
  // This is to workaround https://github.com/sbt/io/issues/110
  sys.props.put("jna.nosys", "true")

  import BasicCommandStrings.isEarlyCommand
  val userCommands = configuration.arguments.map(_.trim)
  val (earlyCommands, normalCommands) = (preCommands ++ userCommands).partition(isEarlyCommand)
  val commands = (earlyCommands ++ normalCommands).toList map { x =>
    Exec(x, None)
  }
  val initAttrs = BuiltinCommands.initialAttributes
  val s = State(
    configuration,
    initialDefinitions,
    Set.empty,
    None,
    commands,
    State.newHistory,
    initAttrs,
    initialGlobalLogging,
    None,
    State.Continue
  )
  s.initializeClassLoaderCache
}
```
