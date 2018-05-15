# sbt v1.1.4 xMainの読み込み


---

## Background

launcherからの読み出し先がxMainであり、
その後どのように実行されているか気になるため

---

## TL; DR

* `State`, `Command`が重要

--

### `sbt.State`

```scala
/**
 * Data structure representing all command execution information.
 *
 * @param configuration provides access to the launcher environment, including the application configuration, Scala versions, jvm/filesystem wide locking, and the launcher itself
 * @param definedCommands the list of command definitions that evaluate command strings.  These may be modified to change the available commands.
 * @param onFailure the command to execute when another command fails.  `onFailure` is cleared before the failure handling command is executed.
 * @param remainingCommands the sequence of commands to execute.  This sequence may be modified to change the commands to be executed.  Typically, the `::` and `:::` methods are used to prepend new commands to run.
 * @param exitHooks code to run before sbt exits, usually to ensure resources are cleaned up.
 * @param history tracks the recently executed commands
 * @param attributes custom command state.  It is important to clean up attributes when no longer needed to avoid memory leaks and class loader leaks.
 * @param next the next action for the command processor to take.  This may be to continue with the next command, adjust global logging, or exit.
 */
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

### `sbt.Command`

```scala
/**
 * An operation that can be executed from the sbt console.
 *
 * <p>The operation takes a [[sbt.State State]] as a parameter and returns a [[sbt.State State]].
 * This means that a command can look at or modify other sbt settings, for example.
 * Typically you would resort to a command when you need to do something that's impossible in a regular task.
 */
sealed trait Command {
  def help: State => Help
  def parser: State => Parser[() => State]

  def tags: AttributeMap
  def tag[T](key: AttributeKey[T], value: T): Command

  def nameOption: Option[String] = this match {
    case sc: SimpleCommand => Some(sc.name)
    case _                 => None
  }
}
```

--

### `import sbt.internal.util.complete.Parser`

```scala
/**
 * A String parser that provides semi-automatic tab completion.
 * A successful parse results in a value of type `T`.
 * The methods in this trait are what must be implemented to define a new Parser implementation, but are not typically useful for common usage.
 * Instead, most useful methods for combining smaller parsers into larger parsers are implicitly added by the [[RichParser]] type.
 */
sealed trait Parser[+T] {
  def derive(i: Char): Parser[T]
  def resultEmpty: Result[T]
  def result: Option[T]
  def completions(level: Int): Completions
  def failure: Option[Failure]
  def isTokenStart = false
  def ifValid[S](p: => Parser[S]): Parser[S]
  def valid: Boolean
}
```

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

* 初期のCommandを設定している
```scala
def early: Command = Command.arb(earlyParser, earlyHelp)((s, other) => other :: s)
```

* `BasicCommands.earlyParser` と `BasicCommands.help`
```scala
private[this] def earlyParser: State => Parser[String] = (s: State) => {
  val p1 = token(EarlyCommand + "(") flatMap (_ => otherCommandParser(s) <~ token(")"))
  val p2 = token("-") flatMap (_ => levelParser)
  p1 | p2
}

private[this] def earlyHelp = Help(EarlyCommand, EarlyCommandBrief, EarlyCommandDetailed)
```

--

### `sbt.Command.arb`

* arb（任意の; arbitrary）は `parser[T]` に対して `effect`（ `T => State` ）適応し、Stateに変更できるようにする。そして、helpを結合させる
```scala
def arb[T](
  parser: State => Parser[T],
  help: Help = Help.empty
)(
  effect: (State, T) => State
): Command = custom(applyEffect(parser)(effect), help)
```

--

### `sbt.Command.custom`

* parserとhelpを結合してCommandとして返す
```scala
def custom(
  parser: State => Parser[() => State],
  help: Help = Help.empty
): Command =
  customHelp(parser, const(help))
```

--

### `sbt.Command.customHelp`

```scala
def customHelp(
  parser: State => Parser[() => State],
  help: State => Help
): Command =
  new ArbitraryCommand(parser, help, AttributeMap.empty)
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

## `sbt.Builtincommands.defaults`

* `name="add-default-commands"` とした `DefaultCommands` を `Command` として設定
* Command詳細は `DefaultCommands` を参照下さい
```scala
def defaults = Command.command(DefaultsCommand) { s =>
  s.copy(definedCommands = DefaultCommands)
}
```

--

### `sbt.Command.command`

```scala
/** Construct a no-argument command with the given name and effect. */
def command(
  name: String,
  help: Help = Help.empty
)(
  f: State => State
): Command =
  make(name, help)(state => success(() => f(state)))
```

--

### `sbt.Command.make`

```scala
def make(
  name: String,
  help: Help = Help.empty
)(
  parser: State => Parser[() => State]
): Command =
  new SimpleCommand(name, help, parser, AttributeMap.empty)
```

```scala
private[sbt] final class SimpleCommand(
    val name: String,
    private[sbt] val help0: Help,
    val parser: State => Parser[() => State],
    val tags: AttributeMap
) extends Command {
  assert(Command validID name, s"'$name' is not a valid command name.")
  def help = const(help0)
  def tag[T](key: AttributeKey[T], value: T): SimpleCommand =
    new SimpleCommand(name, help0, parser, tags.put(key, value))
  override def toString = s"SimpleCommand($name)"
}
```

---

## `sbt.BasicCommandStrings.runEarly`

* early(初期の)設定として実行させるコマンド
* `sbt early(initialize)` とterminalで打っても確認できる
```scala
def runEarly(command: String) = s"$EarlyCommand($command)"
val EarlyCommand = "early"
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
