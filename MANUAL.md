
# Table of Contents

- How to:
    - [Starting a New Debug Session](#starting-a-new-debug-session)
        - [Launching a New Process](#launching-a-new-process)
            - [Stdio Redirection](#stdio-redirection)
        - [Attaching to an Existing Process](#attaching-to-a-running-process)
        - [Custom Launch](#custom-launch)
    - [Debugging Externally Launched Code](#debugging-externally-launched-code)
        - [RPC Server](#rpc-server)
    - [Remote Debugging](#remote-debugging)
    - [Reverse Debugging](#reverse-debugging) (experimental)
    - [Inspecting a Core Dump](#inspecting-a-core-dump)
    - [Source Path Remapping](#source-path-remapping)
    - [Parameterized Launch Configurations](#parameterized-launch-configurations)
- [Debugger Features](#debugger-features)
    - [VSCode Commands](#vscode-commands)
    - [Debugger Commands](#debugger-commands)
    - [Debug Console](#debug-console)
    - [Regex Breakpoints](#regex-breakpoints)
    - [Conditional Breakpoints](#conditional-breakpoints)
    - [Data Breakpoints](#data-breakpoints)
    - [Disassembly View](#disassembly-view)
    - [Formatting](#formatting)
        - [Pointers](#pointers)
    - [Expressions](#expressions)
- [Python Scripting](#python-scripting)
    - [Debugger API](#debugger-api)
- [Alternate LLDB backends](#alternate-lldb-backends)
- [Rust Language Support](#rust-language-support)
- [Workspace Configuration](#workspace-configuration)

# Starting a New Debug Session

To start a debugging session you will need to create a [launch configuration](https://code.visualstudio.com/Docs/editor/debugging#_launch-configurations) for your program.   Here's a minimal one:

```javascript
{
    "name": "Launch",
    "type": "lldb",
    "request": "launch",
    "program": "${workspaceFolder}/<executable file>",
    "args": ["-arg1", "-arg2"],
}
```

 These attributes are common to all CodeLLDB launch configurations:

|attribute          |type  |         |
|-------------------|------|---------|
|**name**           |string| *Required.* Launch configuration name, as you want it to appear in the Run and Debug panel.
|**type**           |string| *Required.* Set to `lldb`.
|**request**        |string| *Required.* Session initiation method:<br><li>`launch` to [create a new process](#launching-a-new-process),<br><li>`attach` to [attach to an already running process](#attaching-to-a-running-process),<br><li>`custom` to [configure session "manually" using LLDB commands](#custom-launch).
|**initCommands**   |[string]| LLDB commands executed upon debugger startup.
|**exitCommands**   |[string]| LLDB commands executed at the end of the debugging session.
|**expressions**    |string| The default expression evaluator type: `simple`, `python` or `native`.  See [Expressions](#expressions).
|**sourceMap**      |dictionary| See [Source Path Remapping](#source-path-remapping).
|**relativePathBase**|string | Base directory used for resolution of relative source paths.  Defaults to "${workspaceFolder}".
|**sourceLanguages**|[string]| A list of source languages used in the program.  This is used to enable language-specific debugger features.
|**reverseDebugging**|bool   | Enable [reverse debugging](#reverse-debugging).



## Launching a New Process

These attributes are applicable when the "launch" initiation method is selected:

|attribute          |type|         |
|-------------------|----|---------|
|**program**        |string| Path of the executable file.  *Required*, unless using `cargo` attribute.
|**cargo**          |string| See [Cargo support](#cargo-support).
|**args**           |string &#10072; [string]| Command line parameters.  If this is a string, it will be split using shell-like syntax.
|**cwd**            |string| Working directory.
|**env**            |dictionary| Additional environment variables.  You may refer to existing environment variables using `${env:NAME}` syntax, for example `"PATH" : "${env:HOME}/bin:${env:PATH}"`.
|**stdio**          |string &#10072; [string] &#10072; dictionary| See [Stdio Redirection](#stdio-redirection).
|**terminal**       |string| Destination for debuggee stdio streams: <ul><li>`console` for Debug Console</li><li>`integrated` (default) for VSCode integrated terminal</li><li>`external` for a new terminal window</li></ul>
|**stopOnEntry**    |boolean| Whether to stop debuggee immediately after launching.
|**preRunCommands** |[string]| LLDB commands executed just before launching the debuggee.
|**postRunCommands**|[string]| LLDB commands executed just after launching the debuggee.

Flow during a launch sequence:
1. The `initCommands` sequence is executed.
2. The debugging target object is created using launch configuration attributes (`program`, `args`, `env`, `cwd`, `stdio`).
3. Breakpoints are set.
4. The `preRunCommands` sequence is executed.  These commands may alter debug target configuration (e.g. alter args or env).
5. The debuggee is launched.
6. The `postRunCommands` sequence is executed.

At the end of the debugging session `exitCommands` sequence is executed.

### Stdio Redirection
The **stdio** property is a list of redirection targets for each of the debuggee's stdio streams:
- `null` value will cause redirect to the default debug session terminal (as specified by the **terminal** launch property),
- `"/some/path"` will cause the stream to be redirected to the specified file, pipe or a TTY device<sup>*</sup>,
- if you provide less than 3 values, the list will be padded to 3 entries using the last provided value,
- you may specify more than three values, in which case additional file descriptors will be created (4, 5, etc.)

Examples:
- `"stdio": [null, "log.txt", null]` - connect stdin and stderr to the default terminal, while sending
stdout to "log.txt",
- `"stdio": ["input.txt", "log.txt"]` - connect stdin to "input.txt", while sending both stdout and stderr to "log.txt",
- `"stdio": null` - connect all three streams to the default terminal.

<sup>*</sup> Run `tty` command in a terminal window to find out the TTY device name.

## Attaching to a Running Process

These attributes are applicable when the "attach" initiation method is selected:

|attribute          |type    |         |
|-------------------|--------|---------|
|**program**        |string  |Path of the executable file.
|**pid**            |number  |Process id to attach to.  **pid** may be omitted, in which case debugger will attempt to locate an already running instance of the program. You may also put `${command:pickProcess}` or `${command:pickMyProcess}` here to choose a process interactively.
|**stopOnEntry**    |boolean |Whether to stop the debuggee immediately after attaching.
|**waitFor**        |boolean |Wait for the process to launch.
|**preRunCommands** |[string]|LLDB commands executed just before attaching to the debuggee.
|**postRunCommands**|[string]|LLDB commands executed just after attaching to the debuggee.

Flow during an attach sequence:
1. The `initCommands` sequence is executed.
2. The debugging target object is created using the `program` attribute.
3. Breakpoints are set.
4. The `preRunCommands` sequence is executed.  These commands may alter debug target configuration.
5. The debugger attaches to the specified process.
6. The `postRunCommands` sequence is executed.

At the end of the debug session `exitCommands` sequence is executed.

Note that attaching to a running process may be [restricted](https://en.wikipedia.org/wiki/Ptrace#Support)
on some systems.  You may need to adjust system configuration to enable it.

## Custom Launch

The custom launch method allows user to fully specify how the debug session is initiated.  The flow of a custom launch is as follows:
1. The `targetCreateCommands` command sequence is executed. This sequence is expected to create the debugging target object (see `target create` command).
2. Debugger uses the target to insert breakpoints.
3. The `processCreateCommands` command sequence is executed.  This sequence is expected to create the debuggee process (see `process launch` command).
4. The debugger reports current state of the debuggee to VSCode and starts accepting user commands.

|attribute          |type    |         |
|-------------------|--------|---------|
|**targetCreateCommands**  |[string]|Commands that create the debug target.
|**processCreateCommands** |[string]|Commands that create the debuggee process.


## Debugging Externally Launched Code

Debugging sessions may also be started from outside of VSCode by invoking a specially formatted URI:

- **`vscode://vadimcn.vscode-lldb/launch?name=<configuration name>,[folder=<path>]`**</br>
  This will start a new debug session using the named launch configuration.  The optional `folder` parameter specifies
  the workspace folder where the launch configuration is defined.  If omitted, all folders in the current workspace will be searched.
  - `code --open-url "vscode://vadimcn.vscode-lldb/launch?name=Debug My Project"`
- **`vscode://vadimcn.vscode-lldb/launch/command?<env1>=<val1>&<env2>=<val2>&<command-line>`**</br>
  The \<command-line\> will be split into the program name and arguments array using the usual shell command-line parsing rules.
  - `code --open-url "vscode://vadimcn.vscode-lldb/launch/command?/path/filename arg1 \"arg 2\" arg3"`
  - `code --open-url "vscode://vadimcn.vscode-lldb/launch/command?RUST_LOG=error&/path/filename arg1 'arg 2' arg3"`
- **`vscode://vadimcn.vscode-lldb/launch/config?<yaml>`**</br>
  This endpoint accepts a [YAML](https://yaml.org/) snippet matching one of the above debug session initiation methods.
  The `type` and the `request` attributes may be omitted, and will default to "lldb" and "launch" respectively.
  - JSON-like YAML (if you are not quoting keys in mappings, remember to insert a space after the colon!):<br>
  `code --open-url "vscode://vadimcn.vscode-lldb/launch/config?{program: '/path/filename', args: ['arg1','arg 2','arg3']}"`<br>
  - Line-oriented YAML (`%0A` encodes the 'newline' character):<br>
   `code --open-url "vscode://vadimcn.vscode-lldb/launch/config?program: /path/filename%0Aargs:%0A- arg1%0A- arg 2%0A- arg3"`<br>


Notes:
- All URIs above are subject to normal [URI encoding rules](https://en.wikipedia.org/wiki/Percent-encoding), therefore all '%' characters must be escaped as '%25'.   A more rigorous launcher script would have done that :)<br>
- VSCode URIs may also be invoked using OS-specific tools:
  - Linux: `xdg-open <uri>`
  - MacOS: `open <uri>`
  - Windows: `start <uri>`

Examples:

### Attaching debugger to the current process (C)
```C
char command[256];
snprintf(command, sizeof(command), "code --open-url \"vscode://vadimcn.vscode-lldb/launch/config?{'request':'attach','pid':%d}\"", getpid());
system(command);
sleep(1); // Wait for debugger to attach
```

### Attaching debugger to the current process (Rust)
Ever wanted to debug a build script?
```Rust
let url = format!("vscode://vadimcn.vscode-lldb/launch/config?{{'request':'attach','pid':{}}}", std::process::id());
std::process::Command::new("code").arg("--open-url").arg(url).output().unwrap();
std::thread::sleep_ms(1000); // Wait for debugger to attach
```

### Debugging Rust unit tests
- Create `.cargo` directory in your project folder containing these two files:
    - `config` [(see also)](https://doc.rust-lang.org/cargo/reference/config.html)
        ```TOML
        [target.<current-target-triple>]
        runner = ".cargo/codelldb.sh"
        ```
    - `codelldb.sh`
        ```sh
        #!/bin/bash
        code --open-url "vscode://vadimcn.vscode-lldb/launch/command?LD_LIBRARY_PATH=$LD_LIBRARY_PATH&$*"
        ```
- `chmod +x .cargo/codelldb.sh`
- Execute tests as normal.

### Bazel
- Create `codelldb.sh`:
    ```sh
    #!/bin/bash
    code --open-url "vscode://vadimcn.vscode-lldb/launch/command?LD_LIBRARY_PATH=$LD_LIBRARY_PATH&$*"
    ```
- `chmod +x codelldb.sh`
- `bazel run --run_under=codelldb.sh //<package>:<target>`

## RPC Server
Unfortunately, starting debug sessons via the "open-url" interface has two problems:
- It launches debug session in the last active VSCode window.
- It [does not work](https://github.com/microsoft/vscode-remote-release/issues/4260) with VSCode remoting.

For these reasons, CodeLLDB offers an alternate method of performing external launches: by adding `lldb.rpcServer` setting to a workspace
of folder configuration you can start an RPC server listening for debug configurations on a Unix or a TCP socket:
- The value is the [options](https://nodejs.org/api/net.html#net_server_listen_options_callback) object of the Node.js network server object.
- As a rudimentary security feature, you may add a "`token`" attribute to the server options above, in which case, the submitted
debug configurations must also contain `token` with a matching value.<br>
- After writing configuration data, the client must half-close its end of the connection.
- Upon completion, CodeLLDB will respond with `{ "success": true/false, "message": <optional error message> }`



### Example:
- Configuration in settings.json: `"lldb.rpcServer": { "host": "127.0.0.1", "port": 12345, "token": "secret" }`
- Launch: `echo "{ program: '/usr/bin/ls', token: 'secret' }" | netcat -N 127.0.0.1 12345`

## Remote debugging

For general information on remote debugging please see [LLDB Remote Debugging Guide](http://lldb.llvm.org/remote.html).

### Connecting to lldb-server agent
- Run `lldb-server platform --server --listen *:<port>` on the remote machine.
- Create launch configuration similar to the one below.
- Start debugging as usual.  The executable identified by the `program` property will
be automatically copied to `lldb-server`'s current directory on the remote machine.
If you require additional configuration of the remote system, you may use `preRunCommands` sequence
to execute commands such as `platform mkdir`, `platform put-file`, `platform shell`, etc.
(See `help platform` for a list of available platform commands).

```javascript
{
    "name": "Remote launch",
    "type": "lldb",
    "request": "launch",
    "program": "${workspaceFolder}/build/debuggee", // Local path.
    "initCommands": [
        "platform select <platform>", // Execute `platform list` for a list of available remote platform plugins.
        "platform connect connect://<remote_host>:<port>",
        "settings set target.inherit-env false", // See note below.
    ],
    "env": {
        "PATH": "...", // See note below.
    }
}
```
Note: By default, debuggee will inherit environment from the debugger.  However, this environment  will be of your
**local** machine.  In most cases these values will not be suitable on the remote system,
so you should consider disabling environment inheritance with `settings set target.inherit-env false` and
initializing them as appropriate, starting with `PATH`.

### Connecting to a gdbserver-style agent
This includes not just gdbserver itself, but also execution environments that implement the gdbserver protocol,
such as [OpenOCD](http://openocd.org/), [QEMU](https://www.qemu.org/), [rr](https://rr-project.org/), and others.

- Start remote agent. For example, run `gdbserver *:<port> <debuggee> <debuggee args>` on the remote machine.
- Create a custom launch configuration.
- Start debugging.
```javascript
{
    "name": "Remote attach",
    "type": "lldb",
    "request": "custom",
    "targetCreateCommands": ["target create ${workspaceFolder}/build/debuggee"],
    "processCreateCommands": ["gdb-remote <remote_host>:<port>"]
}
```


Please note that depending on protocol features implemented by the remote stub, there may be more setup needed.
For example, in the case of "bare-metal" debugging (OpenOCD), the debugger may not be aware of memory locations
of the debuggee modules; you may need to specify this manually:
```
target modules load --file ${workspaceFolder}/build/debuggee -s <base load address>`
```

## Reverse Debugging

Also known as [Time travel debugging](https://en.wikipedia.org/wiki/Time_travel_debugging).  Provided you use a debugging backend that supports
[these commands](https://sourceware.org/gdb/onlinedocs/gdb/Packets.html#bc), CodeLLDB be used to control reverse execution and stepping.

As of this writing, the only backend known to work is [Mozilla's rr](https://rr-project.org/).  The minimum supported version is 5.3.0.

There are others mentioned [here](http://www.sourceware.org/gdb/news/reversible.html) and [here](https://github.com/mozilla/rr/wiki/Related-work).
[QEMU](https://www.qemu.org/) reportedly [supports record/replay](https://github.com/qemu/qemu/blob/master/docs/replay.txt) in full system emulation mode.
If you get any of them to work, please let me know!

### Example: (using rr)
Record execution trace:
```sh
rr record <debuggee> <arg1> ...
```

Replay execution:
```sh
rr replay -s <port>
```

Launch config:
```javascript
{
    "name": "Replay",
    "type": "lldb",
    "request": "custom",
    "targetCreateCommands": ["target create ${workspaceFolder}/build/debuggee"],
    "processCreateCommands": ["gdb-remote 127.0.0.1:<port>"],
    "reverseDebugging": true
}
```

## Inspecting a Core Dump
Use custom launch with `target create -c <core path>` command:
```javascript
{
    "name": "Core dump",
    "type": "lldb",
    "request": "custom",
    "targetCreateCommands": ["target create -c ${workspaceFolder}/core"],
}
```

## Source Path Remapping
Source path remapping is helpful in cases when program's source code is located in a different
directory then it was in at build time (for example, if a build server was used).

A source map consists of pairs of "from" and "to" path prefixes.  When the debugger encounters a source
file path beginning with one of the "from" prefixes, it will substitute the corresponding "to" prefix
instead.  Example:
```javascript
    "sourceMap": { "/build/time/source/path" : "/current/source/path" }
```

## Parameterized Launch Configurations
Sometimes you'll find yourself adding the same parameters (e.g. a path of a dataset directory)
to multiple launch configurations over and over again.  CodeLLDB can help with configuration management
in such cases: you can place common configuration values into `lldb.dbgconfig` section of the workspace configuration,
then reference via `${dbgconfig:variable}` in launch configurations.<br>
Example:

```javascript
// settings.json
    ...
    "lldb.dbgconfig":
    {
        "dataset": "dataset1",
        "datadir": "${env:HOME}/mydata/${dbgconfig:dataset}" // "dbgconfig" properties may reference each other,
                                                             // as long as there is no recursion.
    }

// launch.json
    ...
    {
        "name": "Debug program",
        "type": "lldb",
        "program": "${workspaceFolder}/build/bin/program",
        "cwd": "${dbgconfig:datadir}" // will be expanded to "/home/user/mydata/dataset1"
    }
```

# Debugger Features

## VSCode Commands

|                                 |                                                         |
|---------------------------------|---------------------------------------------------------|
|**Show Disassembly...**          |Choose when disassembly is shown. See [Disassembly View](#disassembly-view).
|**Toggle Disassembly**           |Choose when disassembly is shown. See [Disassembly View](#disassembly-view).
|**Display Format...**            |Choose the default variable display format. See [Formatting](#formatting).
|**Toggle Pointee Summaries**     |Choose whether to display pointee's summaries rather than the numeric value of the pointer itself. See [Pointers](#pointers).
|**Display Options...**           |Interactive configuration of the above display options.
|**Attach to Process...**         |Choose a process to attach to from the list of currently running processes.
|**Use Alternate Backend...**     |Choose alternate LLDB instance to be used instead of the bundled one. See [Alternate LLDB backends](#alternate-lldb-backends)
|**Run Diagnostics**              |Run diagnostic test to make sure that the debugger is functional.
|**Generate launch configurations from Cargo.toml**|Generate all possible launch configurations (binaries, examples, unit tests) for the current Rust project.  The resulting list will be opened in a new text editor, from which you can copy/paste desired sections into `launch.json`.|
|**Command Prompt**               |Open LLDB command prompt in a terminal, for managing installed Python packages and other maintenance tasks.|
|**View Memory...**               |View raw memory starting at the specified address.|
|**Search Symbols...**            |Search for a substring among the debug target's symbols.|


## Debugger Commands

CodeLLDB also adds in-debugger commands that may be executed in the Debug Console during a debug dession:

|                 |                                                         |
|-----------------|---------------------------------------------------------|
|**debug_info**   |Provides tools for investigation of debugging information.  See `debug_info -h` for options.

For more details about each command please use `help <command>`.

## Debug Console

The VSCode [Debug Console](https://code.visualstudio.com/docs/editor/debugging#_debug-console-repl) panel serves a dual
purpose in CodeLLDB:
1. Execution of [LLDB commands](https://lldb.llvm.org/use/tutorial.html).
2. Evaluation of [expressions](#expressions).

By default, console input is interpreted as LLDB commands.  If you would like to evaluate an expression instead, prefix it with
'`?`', e.g. '`?a+2`' (Expression type preffixes are added on top of that, i.e. '`?/nat a.size()`').
Console input mode may altered via **"lldb.consoleMode": "evaluate"** setting: in this case expression evaluation will be the default,
while commands will need to be prefixed with either '`/cmd `' or '`' (backtick).

## Regex Breakpoints
Function breakpoints prefixed with '`/re `', are interpreted as regular expressions.
This causes a breakpoint to be set in every function matching the expression.
The list of created breakpoint locations may be examined using the `break list` command.

## Conditional Breakpoints
You may use any of the supported expression [syntaxes](#expressions) to create breakpoint conditions.
When a breakpoint condition evaluates to False, the breakpoint will not be stopped at.
Any other value (or expression evaluation error) will cause the debugger to stop.

## Data Breakpoints
Data breakpoints (or "watchpoints" in LLDB terms) allow monitoring memory location for changes.  You can create data
breakpoints by choosing "Break When Value Changes" from context menu in the Variables panel. (To access advanced features,
such as breaking on memory reads, use LLDB `watch` command).

Note that data breakpoints require hardware support, and, as such, may come with restrictions, depending on CPU platform and OS support.
For example, on x86_64 the restrictions are as follows:
- The monitored memory region must be 1, 2, 4 or 8 bytes in size.
- There may be at most 4 data watchpoints.


## Hit conditions
Syntax:
```
    operator :: = '<' | '<=' | '=' | '>=' | '>' | '%'
    hit_condition ::= operator number
```

The `'%'` operator causes a stop after every `number` of breakpoint hits.

## Logpoints
Expressions embedded in log messages via curly brackets may use any of the supported expression [syntaxes](#expressions).

## Disassembly View
When execution steps into code for which debug info is not available, CodeLLDB will automatically
switch to disassembly view.  This behavior may be controlled using **Show Disassembly**
and **Toggle Disassembly** commands.  The former allows to choose between `never`,
`auto` (the default) and `always`, the latter toggles between `auto` and `always`.

While is disassembly view, 'step over' and 'step into' debug actions will perform instruction-level
stepping rather than source-level stepping.

![disassembly view](images/disasm.png)

## Formatting
You may change the default display format of evaluation results using the `Display Format` command.

When evaluating expressions in Debug Console or in Watch panel, you may control formatting of
individual expressions by adding one of the suffixes listed below.  For example evaluation of `var,x`
will display the value of `var` formatted as hex.

|suffix |format |
|:-----:|-------|
|**c**  | Character
|**x**  | Hex
|**o**  | Octal
|**d**  | Decimal
|**u**  | Unsigned decimal
|**b**  | Binary
|**f**  | Float (reinterprets bits, no casting is done)
|**p**  | Pointer
|**s**  | C string
|**y**  | Bytes
|**Y**  | Bytes with ASCII
|**[\<num\>]**| Reinterpret as an array of \<num\> elements

### Pointers

When displaying pointer and reference variables, CodeLLDB will prefer to display the
value of the object pointed to.  If you would like to see the raw address value,
you may toggle this behavior using **Toggle Pointee Summaries** command.
Another way to display raw pointer address is to add the pointer variable to Watch panel and specify
an explicit format, as described in the previous section.

## Expressions

CodeLLDB implements three expression evaluator types: "simple", "python" and "native".  These are used
wherever user-entered expression needs to be evaluated: in the Watch panel, in the Debug Console (for input
prefixed with `?`) and in breakpoint conditions.<br>
By default, "simple" is assumed, however you may change this using the [expressions](#launching-a-new-process) launch
configuration property.  The default type may also be overridden on a per-expression basis using a prefix.

### Simple expressions
Prefix: `/se `<br>
Simple expressions are designed to enable performing basic arithmetic and logical operations on [formatted
views](https://lldb.llvm.org/varformats.html) of the debuggee variables.  For example, things like indexing an
`std::vector` or comparing `std::string` to a string literal should "just work".

The followng features are supported:
- References to variables: all identifiers are assumed to refer to variables in the debuggee's current stack frame.
  The identifiers may be qualified with namespaces and template parameters (e.g. `std::numeric_limits<float>::digits`).
- Embedded [native expressions](#native-expressions): these must be delimited with `${` and `}`.
- Literals: integers, floats and strings, `True`, `False`.
- Operators: `()`, `**`, `*`, `/`, `//`, `%`, `<<`, `>>`, `~`, `&`, `^`, `|`, `==`, `!=`, `>`, `>=`, `<`, `<=`,
             `not`, `and`, `or` with the same precedence as in Python.
- Attribute access: `<expr>.<attr>`.
- Indexing: `<expr>[<expr>]`.

### Python expressions
Prefix: `/py `<br>
Python expressions support full Python syntax.  In addition to that, any identifier prefixed by `$`, will be replaced
with the value of the corresponding debuggee variable.  Such values may be mixed with regular Python variables.
For example, `/py [math.sqrt(x) for x in $arr]` will evaluate to a list of square roots of the values contained in
the array variable `arr`.

The environment in which Python expressions are executed is shared with the internal Python interpreter of the debugger
and is affected by the `script ...` command.   This may be used to import Python modules you are going to use later.
For example, in order to evaluate `math.sqrt(x)` above, you'll need to have imported the `math` package via
`script import math`.  To import Python modules on debug session startup, use `"initCommands": ["script import ..."]`.

**Technical note**<br>
Evaluation of Python expressions is performed as follows:
- First, the expression is preprocessed and all tokens starting with '$' are replaced with calls to the `__expr()` function,
  For example, the expression `[math.sqrt(x) for x in $arr]` will be re-written as `[math.sqrt(x) for x in __eval('arr')]`
- The resulting string is evaluated by the Python interpreter, with the `__eval()` function performing variable
  lookups and evaluation of native expressions, returning instances of [`Value`](#value).

### Native expressions
Prefix: `/nat `<br>
Native expressions use LLDB's built-in expression evaluators.  The specifics depend on source language of the
current debug target (e.g. C, C++ or Swift).<br>
For example, the C++ expression evaluator offers many powerful features including interactive definition
of new data types, instantiation of C++ classes, invocation of functions and class methods, and more.

Note, however, that native evaluators ignore data formatters and operate on "raw" data structures,
thus they are often not as convenient as "simple" or "python" expressions.

# Python Scripting

## Debugger API

CodeLLDB provides extended Python API via `codelldb` module (which is auto-imported into debugger's main script context).

- **evaluate(expression: `str`, unwrap=False) -> `Value` | `lldb.SBValue`** : Performs dynamic evaluation of [native expressions](#native-expressions) returning instances of [`Value`](#value).
    - **expression**: The expression to evaluate.
    - **unwrap**: Whether to unwrap the result and return it as `lldb.SBValue`.
- **unwrap(obj: `Value`) -> `lldb.SBValue`** : Extracts an [`lldb.SBValue`](https://lldb.llvm.org/python_api/lldb.SBValue.html) from [`Value`](#value).
- **wrap(obj: `lldb.SBValue`) -> `Value`** : Wraps [`lldb.SBValue`](https://lldb.llvm.org/python_api/lldb.SBValue.html) in a [`Value`](#value) object.
- **display_html(html: `str`, title: `str` = None, position: `int` = None, reveal: `bool` = False)** : Displays content in a VSCode WebView panel:
    - **html**: HTML markup to display.
    - **title**: Title of the panel.  Defaults to the name of the current launch configuration.
    - **position**: Position (column) of the panel.  The allowed range is 1 through 3.
    - **reveal**: Whether to reveal a panel if one already exists.

## Value
`Value` objects ([source](adapter/value.py)) are proxy wrappers around [`lldb.SBValue`](https://lldb.llvm.org/python_api/lldb.SBValue.html),
which add implementations of standard Python operators.

## Installing Packages

CodeLLDB bundles its own copy of Python, which may be different from the version of your default Python.
As such, it likely won't be able to use third-party packages you've installed through `pip`.  In order to install packages
for use in CodeLLDB, you will need to use the **LLDB: Command Prompt** command in VSCode, followed by `pip install --user <package>`.

# Alternate LLDB Backends

CodeLLDB can use external LLDB backends instead of the bundled one.  For example, when debugging Swift programs,
one might want to use a custom LLDB instance that has Swift extensions built in.   In order to use an alternate backend,
you will need to provide location of the corresponding LLDB dynamic library (which must be v10.0 or later) via
**lldb.library** configuration setting.

Where to find the LLDB dynamic library:
- Linux: `<lldb root>/lib/liblldb.so.<verson>`,<br>
    `<lldb root>` is wherever you've installed LLDB, or `/usr`, if it's a standard distro package.
- MacOS: `<lldb framework>/LLDB` if built as Apple framework, `<lldb root>/lib/liblldb.<version>.dylib` otherwise.<br>
    `<lldb framework>` is typically located under `/Library/Developer/<toolchain>/.../PrivateFrameworks`.
- Windows: `<lldb root>/bin/liblldb.dll`.

Since locating liblldb is not always trivial, CodeLLDB provides the **Use Alternate Backend...** command to assist with this task.
You will be prompted to enter the file name of the main LLDB executable, which CodeLLDB will then use to find the dynamic library.

Note: Debian builds of LLDB have a bug whereby they search for `lldb-server` helper binary relative to the current
executable module (which in this case is CodeLLDB), rather than relative to liblldb (as they should).  As a result,
you may see the following error after switching to an alternate backend: "Unable to locate lldb-server-\<version\>".
To fix this, determine where `lldb-server` is installed (via `which lldb-server-<version>`), then add
this configuration entry: `"lldb.adapterEnv": {"LLDB_DEBUGSERVER_PATH": "<lldb-server path>"}`.


# Rust Language Support

CodeLLDB natively supports visualization of most common Rust data types:
- Built-in types: tuples, enums, arrays, array and string slices.
- Standard library types: `Vec`, `String`, `CString`, `OSString`, `Path`, `Cell`, `Rc`, `Arc` and more.

To enable this feature, add `"sourceLanguages": ["rust"]` into your launch configuration.

![source](images/source.png)

## Cargo support

Several Rust users had pointed out that debugging tests and benchmarks in Cargo-based projects is somewhat
difficult since names of the output test/bench binary generated by Cargo is not deterministic.
To cope with this problem, CodeLLDB can query Cargo for a list of its compilation outputs.  In order
to use this feature, replace `program` property in your launch configuration with `cargo`:
```javascript
{
    "type": "lldb",
    "request": "launch",
    "cargo": {
        "args": ["test", "--no-run", "--lib"],      // Cargo command line to build the debug target
                                                    // "args": ["build", "--bin=foo"] is another possibility
        // The rest are optional
        "env": { "RUSTFLAGS": "-Clinker=ld.mold" }, // Extra environment variables.
        "problemMatcher": "$rustc",                 // Problem matcher(s) to apply to cargo output.
        "filter": {                                 // Filter applied to compilation artifacts.
            "name": "mylib",
            "kind": "lib"
        }
    }
}
```
Try to be as specific as possible when specifying the build target, because if there is more than one
binary output, CodeLLDB won't know which one you want it to debug!

Normally, Cargo output will be used to set the `program` property (but only if it isn't defined).
However, in order to support custom launch and other oddball scenarios, there is also
a substitution variable, which expands to the same thing: `${cargo:program}`.

CodeLLDB will also use `Cargo.toml` in the workspace root to generate initial debug
configurations when there is no existing `launch.json`.

# Workspace Configuration

## Default Launch Configuration Settings
|                                |                                                         |
|--------------------------------|---------------------------------------------------------|
|**lldb.launch.initCommands**    |Commands executed *before* initCommands of individual launch configurations.
|**lldb.launch.preRunCommands**  |Commands executed *before* preRunCommands of individual launch configurations.
|**lldb.launch.postRunCommands** |Commands executed *before* postRunCommands of individual launch configurations.
|**lldb.launch.exitCommands**    |Commands executed *after* exitCommands of individual launch configurations.
|**lldb.launch.env**             |Additional environment variables that will be merged with 'env' of individual launch configurations.
|**lldb.launch.cwd**             |Default program working directory.
|**lldb.launch.stdio**           |Default stdio destination.
|**lldb.launch.expressions**     |Default expression evaluator.
|**lldb.launch.terminal**        |Default terminal type.
|**lldb.launch.sourceMap**       |Additional entries that will be merged with 'sourceMap's of individual launch configurations.
|**lldb.launch.relativePathBase**|Default base directory used for resolution of relative source paths.  Defaults to "${workspaceFolder}".
|**lldb.launch.sourceLanguages** |A list of source languages used in the program.  This is used to enable language-specific debugger features.

## General
|                                   |                                                         |
|-----------------------------------|---------------------------------------------------------|
|**lldb.dbgconfig**                 |See [Parameterized Launch Configurations](#parameterized-launch-configurations).
|**lldb.evaluationTimeout**         |Timeout for expression evaluation, in seconds (default=5s).
|**lldb.displayFormat**             |The default format for variable and expression values.
|**lldb.showDisassembly**           |When to show disassembly:<li>`auto` - only when source is not available.,<li>`never` - never show.,<li>`always` - always show, even if source is available.
|**lldb.dereferencePointers**       |Whether to show the numeric value of pointers, or a summary of the pointee.
|**lldb.suppressMissingSourceFiles**|Suppress VSCode's messages about missing source files (when debug info refers to files not present on the local machine).
|**lldb.consoleMode**               |Controls whether the debug console input is by default treated as debugger commands or as expressions to evaluate:<li>`commands` - treat debug console input as debugger commands.  In order to evaluate an expression, prefix it with '?' (question mark).",<li>`evaluate` - treat debug console input as expressions.  In order to execute a debugger command, prefix it with '/cmd ' or with '\`' (backtick), <li>`split` - (experimental) use the debug console for evaluation of expressions, open a separate terminal for LLDB console.

## Advanced
|                       |                                                         |
|-----------------------|---------------------------------------------------------|
|**lldb.library**       |The [alternate](#alternate-lldb-backends) LLDB library to use. This can be either a file path (recommended) or a directory, in which case platform-specific heuristics will be used to locate the actual library file.
|**lldb.cargo**         |Name of the command to invoke as Cargo.
|**lldb.adapterEnv**    |Extra environment variables passed to the debug adapter.
|**lldb.verboseLogging**|Enables verbose logging.  The log can be viewed in Output/LLDB panel.
|**lldb.reproducer**    |Enable capture of a [reproducer](https://lldb.llvm.org/design/reproducers.html).  May also contain a path of the directory to save the reproducer in.
|**lldb.terminalPromptClear**|A sequence of strings sent to the terminal in order to clear its command prompt.  Defaults to `["\n"]`.  To disable prompt clearing, set to `null`.
|**lldb.evaluateForHovers**  |Enable value preview when cursor is hovering over a variable.
|**lldb.commandCompletions** |Enable command completions in debug console.
|**lldb.rpcServer**          |See [RPC server](#rpc-server).
