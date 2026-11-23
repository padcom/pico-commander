# pico-commander

[commander.js](https://github.com/tj/commander.js) for [Pico](https://github.com/padcom/pico-lang): options, arguments,
subcommands and generated help for command-line programs. It follows commander 14: the same API wherever Pico allows
it, and the same parsing, error messages, exit codes and help layout (checked against commander.js 14.0.3 on the same
programs). It is plain Pico, with no native code, so it runs on every platform Pico does.

It needs Pico 0.9.0 or later: `argv0` and `execPath()` for the program's name and executable subcommands (0.8.0), and the
standard library's `std/text`, `std/term`, `wrap` and `hasColors()`, which it uses for suggestions, help layout and colours
(0.9.0).

## Use

Add it to `project.json` (the package is `padcom/pico-commander`; before 0.5.0 it was `padcom/commander`):

```json
"dependencies": {
  "padcom/pico-commander": { "git": "https://github.com/padcom/pico-commander", "tag": "v0.5.0" }
}
```

```ts
import { Command } from "padcom/pico-commander"

const program = new Command()

program
  .name("string-util")
  .description("CLI to some JavaScript string utilities")
  .version("0.8.0")

program.command("split")
  .description("Split a string into substrings and display as an array")
  .argument("<string>", "string to split")
  .option("--first", "display just the first substring")
  .option("-s, --separator <char>", "separator character", ",")
  .action((args, opts, _cmd) => {
    const parts = args.string("string")!.split(opts.string("separator")!)
    println(`${if (opts.bool("first")) parts.slice(0, 1) else parts:?}`)
  })

program.parse()
```

```
$ string-util --help
Usage: string-util [options] [command]

CLI to some JavaScript string utilities

Options:
  -V, --version             output the version number
  -h, --help                display help for command

Commands:
  split [options] <string>  Split a string into substrings and display as an
                            array
  help [command]            display help for command
```

`program` is also exported, ready-made, as in commander. `parse()` reads `std/process`'s `args`; `parse(["-d", "x"])`
parses a list of your own (it is always the user's arguments: there is no `from: "node"`).

## Options

```ts
program
  .option("-d, --debug", "output extra debugging")                 // a flag
  .option("-p, --pizza-type <type>", "flavour of pizza")            // needs a value
  .option("-c, --cheese [type]", "add cheese", "mozzarella")        // optional value, with a default
  .option("--no-sauce", "remove sauce")                             // `sauce` is true unless --no-sauce
  .option("-t, --toppings <items...>", "toppings")                  // every value up to the next option
  .option("-n, --count <n>", "how many", parseInteger, 1)           // a parser, with a default
  .option("-v, --verbose", "verbosity", increase, 0)                // -vvv is 3
  .option("-I, --include <dir>", "include directory", collect)      // repeatable
  .requiredOption("-o, --out <file>", "where to write")             // must be given
  .addOption(new Option("-s, --size <size>").choices(["small", "medium", "large"]).default("medium"))
  .addOption(new Option("--port <n>").env("PORT").argParser(parseInteger))
  .addOption(new Option("--level [l]").preset("high"))
  .addOption(new Option("--quiet").implies({ logLevel: "off" }))
  .addOption(new Option("--cash").conflicts("creditCard"))
  .addOption(new Option("--secret").hideHelp())
```

Short flags combine (`-abc`, `-n5`), long options take `--name=value`, `--` ends the options, and a negative number
(`-5`, `-1.5`) is a value, not an option, unless a short flag is a digit.

## Values

Commander keeps whatever JavaScript value a parser returns; Pico is statically typed, so a value is a `Value`:
`Bool`, `Str`, `Int`, `Num` or `List` (of strings). `opts()` and the arguments an action gets are `Values`, which read
them back by name, or by position (an action's arguments: `args.string(0)`):

| | |
|---|---|
| `bool(name)` | given and truthy (a flag, a non-empty string, a non-zero number, a list) |
| `string(name)` | a string (or a number as text); `null` for a flag without a value, a list or nothing |
| `strings(name)` | a list; a single string is a list of one; nothing is `[]` |
| `int(name)`, `number(name)` | an `Int`/`Num`, or a string that is one; else `null` |
| `get(name)`, `has(name)`, `keys()`, `length` | the `Value?` itself |

A `Value` prints as JSON with `v.json()` (`"a"`, `1`, `["x"]`) and as commander shows it in help with `v.text()`.

Defaults, presets and implied values can be given as `bool`, `string`, `int`, `number` or a `Value`
(`Value.List(["a"])`). A parser is `(value: string, previous: Value?) => Value throws Error`; throw
`InvalidArgumentError` to reject the text, and commander's message says what was wrong:

```
error: option '-n, --count <n>' argument 'x' is invalid. Not an integer.
```

The package's parsers: `parseInteger`, `parseNumber`, `collect` (a repeated option into a list) and `increase`
(counting a flag).

## Arguments and subcommands

```ts
program.command("clone <source> [destination]")
  .description("clone a repository")
  .alias("cl")
  .option("--depth <n>", "history depth", parseInteger)
  .action((args, opts, cmd) => {
    println(`${args.string("source")} → ${args.string("destination") ?? "."}`)
    println(`${cmd.optsWithGlobals()}`)                  // this command's options and its parents'
  })

program.command("serve", { isDefault: true })            // runs when no subcommand is named
program.command("debug", { hidden: true })               // not in the help
program.addCommand(new Command("build").argument("<files...>"))
program.addArgument(new Argument("[mode]", "how").choices(["fast", "slow"]).default("fast"))
```

**Executable subcommands**, as commander's: with a description, `command` adds a subcommand that is a program of its
own (and returns the parent, for adding more):

```ts
program
  .command("install [package-names...]", "install one or more packages")   // runs pm-install
  .command("search [query]", "search with a query", { executableFile: "pm-find" })
program.parse()
```

`pm install a b` runs `pm-install a b` from the program's directory (`executableDir(dir)` makes that relative to it), or
else from `PATH`, with this program's stdin, stdout and stderr, and exits with its exit code; `pm help install` runs
`pm-install --help`. With `exitOverride()` the end is a `CommanderError` (`commander.executeSubCommandAsync`, the exit
code), from `parseAsync` too. See examples/pm.pi.

A missing required argument, too many arguments (unless `allowExcessArguments()`), an unknown option (unless
`allowUnknownOption()`) and an unknown command are errors, with "(Did you mean …?)" for close misspellings.
`enablePositionalOptions()` and `passThroughOptions()` work as in commander.

## Actions, hooks and async

```ts
program.hook("preAction", (thisCommand, actionCommand) => { … })   // also postAction, preSubcommand

async function deploy(args: Values, opts: Values, cmd: Command): Promise<void> throws Error { … }
program.command("deploy <env>").action(deploy)
program.command("wait").action(async (args, opts, cmd) => { await sleep(100) })
await program.parseAsync()
```

`parse()` runs actions and hooks as it goes. `action` and `hook` also take an async function or lambda (one that
returns a `Promise`); those need `parseAsync()`, which validates the whole command line and then runs the hooks and
actions in order, each awaited; a subcommand is parsed after its `preSubcommand` hooks have finished, so they can change
it. (`actionAsync`/`hookAsync` are the same, for 0.1 programs.) See examples/deploy.pi, which also shows help groups,
styles and `exitOverride()`.

**What `parse` reads**: with no arguments, the program's (`args` in `std/process`), and the program is named after its
executable (`nameFromFilename(argv0)`; under `pico run --interp`, its entry file) unless it has a name. An array is the
arguments alone, unless `{ from: "node" }` (a runtime and a script first: `["node", "app.js", …]`, the script names the
program) or `{ from: "electron" }` (an app first) says otherwise. `rawArgs` keeps what `parse` got, and a command can be
parsed again: each parse starts from the option values it had before the first (`saveStateBeforeParse`,
`restoreStateBeforeParse`).

## Help, version, errors and output

`version(v, flags?, description?)`, `helpOption(flags, description)` (`helpOption(false)` turns it off),
`helpCommand(nameAndArgs, description)` (`helpCommand(false)`, or `true` to add it without subcommands),
`addHelpText("beforeAll" | "before" | "after" | "afterAll", text)` (or a function of `{ error, command }` giving the
text), `usage(text)`, `summary(text)`,
`showHelpAfterError()` or `showHelpAfterError(message)`, `showSuggestionAfterError(false)`, `configureHelp({ helpWidth,
sortOptions, sortSubcommands, showGlobalOptions, style… })`, `help()`, `outputHelp()`, `helpInformation()` (each also with
`{ error: true }`), and `error(message)` or `error(message, { exitCode, code })` for errors of your own. `name()`,
`description()`, `summary()`, `usage()`, `alias()`, `aliases()` and `version()` without an argument read the value back.

By default errors and help-on-error go to stderr and end the program (exit code 1; 0 for `--help` and `--version`).
`exitOverride()` throws a `CommanderError` (with commander's `code`, such as `commander.unknownOption`, and the
`exitCode`) instead; `exitOverride(fn)` calls `fn` with it first. `configureOutput({ writeOut, writeErr, outputError,
getOutHelpWidth, getErrHelpWidth, getOutHasColors, getErrHasColors, stripColor })` sends the output elsewhere; tests
use both:

```ts
let err = ""
program.exitOverride().configureOutput({ writeErr: (s: string) => { err += s } })
```

The help is as wide as the terminal (80 columns when it isn't one).

**Groups**, as in commander 14: `optionsGroup("Network:")` and `commandsGroup("Main:")` put the options and commands
added after them under that heading; `helpGroup(heading)` on an `Option` or a `Command` sets one item's (and on a
`Command` with no argument, the group it's listed in). `helpOption` and `helpCommand` join the current group.

**Styles**: `configureHelp({ styleTitle, styleUsage, styleOptionTerm, styleSubcommandTerm, styleArgumentTerm,
styleCommandDescription, styleDescriptionText, … })` take a function of the text, for colours:

```ts
program.configureHelp({ styleTitle: (s: string) => `\u{1b}[1m${s}\u{1b}[22m` })
```

Columns line up by the columns the text takes in a terminal (`std/term`'s `displayWidth`: colour codes take none, East
Asian wide characters two), and the codes are removed when the output has no colours: `Writer.hasColors()` decides
(`NO_COLOR`, `FORCE_COLOR` and `CLICOLOR_FORCE`, else whether it's a terminal; `getOutHasColors` overrides).

`Help`'s methods are virtual (except the generic `groupItems`): override `createHelp()` in a `Command` subclass for a
layout of your own. As in commander 14, `prepareContext({ error, helpWidth, outputHasColors })` is called first, then
`formatHelp(cmd, helper)`.

## Differences from commander.js

Most come from Pico being statically typed, with no `any` or `undefined`:

- **Values are typed**: see [Values](#values). Options are read with `opts().string("x")` and friends, not as
  properties (there is no `storeOptionsAsProperties`).
- **The action gets `(args, opts, cmd)`**: the arguments as one `Values`, not spread.
- **`parse(array)` reads the arguments alone** by default (commander's default is `from: "node"`): a Pico program's
  arguments don't start with a runtime.
- **Mistakes in the program panic**: duplicate flags, a variadic argument that isn't last, an unknown hook event.
  Commander throws a plain `Error` for these.
- **Executable subcommands** are programs (not `.js` files run with node), found as `name` or `name.exe`; signals
  aren't forwarded to them.
- **Not ported**: `storeOptionsAsProperties`.

## Develop

```
pico fmt --check . && pico lint . && pico test .
pico run examples/string-util.pi -- --help
pico run examples/pizza.pi -- -p hawaiian -c blue -vv
pico run examples/deploy.pi -- up staging --host example.com -v
pico build examples/pm.pi -o bin/pm && pico build examples/pm-install.pi -o bin/pm-install && bin/pm install a
```

## License

MIT, as commander.js is (see [LICENSE](LICENSE)).
