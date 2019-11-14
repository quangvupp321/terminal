---
author: Mike Griese @zadjii-msft
created on: 2019-11-08
last updated: 2019-11-08
issue id: #607
---

# Commandline Arguments for the Windows Terminal

## Abstract

This spec outlines the changes necessary to add support to Windows Terminal to
support commandline arguments. These arguments can be used to enable customized
launch scenarios for the Terminal, such as booting directly into a specific
profile or directory.

## Inspiration

Since the addition of the "execution alias" `wt.exe` which enables launching the
Windows Terminal from the commandline, we've always wanted to support arguments
to enable custom launch scenarios. This need was amplified by requests like:
* [#576], which wanted to add jumplist entries for the Windows Terminal, but was
  blocked becaues there was no way of communicating to the Terminal _which_
  profile it wanted to launch
* [#1060] - being able to right-click in explorer to "open a Windows Terminal
  Here" is great, but would be more powerful if it could also provide options to
  open specific profiles in that directory.
* [#2068] - We want the user to be able to (from inside the Terminal) not only
  open a new window with the default profile, but also open the new window with
  a specific profile.

Additionally, the final design for the arguments was heavily inspired by the
arguments available to `tmux`, which also enables robust startup configuration
through commandline arguments.

## User Stories

Lets consider some different ways that a user or developer might want want to
use commandline arguments, to help guide the design.

1. A user wants to open the Windows Terminal with their default profile.
  - This one is easy, it's already provided with simply `wt`.
2. A user wants to open the Windows Terminal with a specific profile from their
   list of profiles.
3. A user wants to open the Windows Terminal with their default profile, but
   running a different commandline than usual.
4. A user wants to know the list of arguments supported by `wt.exe`.
5. A user wants to see their list of profiles, so they can open one in
   particular
6. A user wants to open their settings file, without needing to open the
   Terminal window.
7. A user wants to know what version of the Windows Terminal they are running,
   without needing to open the Terminal window.
8. A user wants to open the Windows Terminal at a specific location on the
   screen
9. A user wants to open the Windows Terminal in a specific directory.
10. A user wants to open the Windows Terminal with a specific size
11. A user wants to open the Windows Terminal with only the default settings,
    ignoring their user settings.
12. A user wants to open the Windows Terminal with multiple tabs open
    simultaneously, each with different profiles, starting directories, even
    commandlines
13. A user wants to open the Windows Terminal with multiple tabs and panes open
    simultaneously, each with different profiles, starting directories, even
    commandlines, and specific split sizes
14. A user wants to use a file to provide a reusable startup configuration with
    many steps, to avoid needing to type the commandline each time.

## Solution Design

### Proposal 1 - Parameters

Initially, I had considered arguments in the following style:

* `--help`: Display the help message
* `--version`: Display version info for the Windows Terminal
* `--list-profiles`: Display a list of the available profiles
  - `--all` to also show "hidden" profiles
  - `--verbose`? To also display GUIDs?
* `--open-settings`: Open the settings file
* `--profile <profile name>`: Start with the given profile, by name
* `--guid <profile guid>`: Start with the given profile, by GUID
* `--startingDirectory <path>`: Start in the given directory
* `--initialRows <rows>`, `--initialCols <rows>`: Start with a specific size
* `--initialPosition <x,y>`: Start at an initial location on the screen
* `-- <commandline>`: Start with this commandline instead

However, this style of arguments makes it very challenging to start multiple
tabs or panes simultaneously. How would a user start multiple panes, each with a
different commandline? As configurations become more complex, these commandlines
would quickly become hard to parse and understand for the user.

### Proposal 2 - Commands and Parameters

Instead, we'll try to seperate these arguments by their responsibilities. Some
of these arguments cause something to happen, like `help`, `version`, or
`open-settings`. Other arguments act more like modifiers, like for example
`--profile` or `--startingDirectory`, which provide additional information to
the action of _opening a new tab_. Lets try and define these concepts more
clearly.

**Commands** are arguments that cause something to happen. They're provided in
`kebab-case`, and can have some number of optional or required "parameters".

**Parameters** are arguments that provide additional information to "commands".
They can be provided in either a long form or a short form. In the long form,
they're provided in `--camelCase`, with two hyphens preceeding the argument
name. In short form, they're provided as just a single character preceeded by a
hyphen, like so: `-c`.

Let's enumerate some possible example commandlines, with explanations, to
demonstrate:

### Sample Commandlines

```sh

# Runs the user's "Windows Powershell" profile in a new tab (user story 2)
wt new-tab --profile "Windows Powershell"
wt --profile "Windows Powershell"
wt -p "Windows Powershell"

# Runs the user's default profile in a new tab, running cmd.exe (user story 3)
wt cmd.exe

# display the help text (user story 4)
wt help
wt --help
wt -h
wt -?

# output the list of profiles (user story 5)
wt list-profiles

# open the settings file, without opening the Terminal window (user story 6)
wt open-settings

# Display version info for the Windows Terminal (user story 7)
wt version
wt --version
wt -v

# Start the default profile in directory "c:/Users/Foo/dev/MyProject" (user story 8)
wt new-tab --startingDirectory "c:/Users/Foo/dev/MyProject"
wt --startingDirectory "c:/Users/Foo/dev/MyProject"
wt -d "c:/Users/Foo/dev/MyProject"

# Runs the user's "Windows Powershell" profile in a new tab in directory
#  "c:/Users/Foo/dev/MyProject" (user story 2, 8)
wt new-tab --profile "Windows Powershell" --startingDirectory "c:/Users/Foo/dev/MyProject"
wt --profile "Windows Powershell" --startingDirectory "c:/Users/Foo/dev/MyProject"
wt -p "Windows Powershell" -d "c:/Users/Foo/dev/MyProject"

# open a new tab with the "Windows Powershell" profile, and another with the
#  "cmd" profile (user story 12)
wt new-tab --profile "Windows Powershell" ; new-tab --profile "cmd"
wt --profile "Windows Powershell" ; new-tab --profile "cmd"
wt --profile "Windows Powershell" ; --profile "cmd"
wt --p "Windows Powershell" ; --p "cmd"

# run "my-commandline.exe with some args" in a new tab
wt new-tab my-commandline.exe with some args
wt my-commandline.exe with some args

# run "my-commandline.exe with some args and a ; literal semicolon" in a new
#  tab, and in another tab, run "another.exe running in a second tab"
wt my-commandline.exe with some args and a \; literal semicolon ; new-tab another.exe running in a second tab

# Start cmd.exe, then split it vertically (with the first taking 70% of it's
#  space, and the new pane taking 30%), and run wsl.exe in that pane (user story 13)
wt cmd.exe ; split-pane -t 0 -v -P 30 wsl.exe
wt cmd.exe ; split-pane -P 30 wsl.exe

# Create a new window with the default profile, create a vertical split with the
#  default profile, then create a horizontal split in the second pane and run
#  "media.exe" (user story 13)
wt new-tab ; split-pane -v ; split-pane -t 1 -h media.exe

```

## `wt` Syntax

The `wt` commandline is divided into two main sections: "Options", and "Commands":

`wt [options] [command ; ]...`

Options are a list of flags and other parameters that can control the behavior
of the `wt` commandline as a whole. Commands are a semicolon-delimited list of
commands and arguments for those commands.

If no command is specified in a `command`, then the command is assumed to be a
`new-tab` command by default. So, for example, `wt cmd.exe` is interpreted the
same as `wt new-tab cmd.exe`.

<!--
### Aside: What should the default command be?

These are notes from my draft intentionally left here to help understand the
conclusion that new-tab should be the default command.

Should the default command be `new-window` or `new-tab`?

`new-window` makes sense to take params like `--initialPosition`,
`--initialRows`/`--initialCols`, and _implies_ `new-tab`. However, chained
commands that want to open in the same window _need_ to specify `new-tab`,
otherwise they'll all appear in new windows.

If it's `new-tab`, then how do `--initialRows` (etc) work? `new-tab` generally
_doesn't_ accept those parameters, because it's going to be inheriting the
parent's window size. Do we just ignore them for subsequent invocations? I
suppose that makes sense, once the first tab has set those, then the other tabs
can't really change them.

When dealing with a file full of startup commands, we'll assume all of them are
intended for the given window. So the first `new-tab` in the file will create
the window, and all subsequent `new-tab` commands will create tabs in that same
window.
 -->

TODO: What should we do with an entirely empty command? My gut says _ignore it_.
For example, `wt ; ; ; ` should not just open 4 tabs. But then what about `wt`
by itself? We want that to open a new tab with the default profile. So maybe we
should allow them? Or allow them on the first command only?


### Options

#### `--help,-h,-?`
Runs the `help` command.

#### `--version,-v`
Runs the `version` command.

#### `--session,-s session-id`
Run these commands in the given Windows Terminal session. Enables opening new
tabs in already running Windows Terminal windows. This feature is dependent upon
other planned work landing, so is only provided as an example, of what it might
look like. See [Future Considerations](#Future-Considerations) for more details.

#### `--file,-f congifuration-file`
Run these commands in the given Windows Terminal session. Enables opening new
tabs in already running Windows Terminal windows. See [Future
Considerations](#Future-Considerations) for more details.

### Commands

#### `help`

`help`

Display the help message

#### `version`

`version`

Display version info for the Windows Terminal

#### `open-settings`

`open-settings [--defaults,-d]`

Open the settings file.

**Parameters**:
* `--defaults,-d`: Open the `defaults.json` file instead of the `profiles.json`
  file.

#### `list-profiles`

`list-profiles [--all,-A] [--showGuids,-g]`

Displays a list of each of the available profiles. Each profile displays it's
name, seperated by newlines.

**Parameters**:
* `--all,-A`: Show all profiles, including profiles marked `"hidden": true`.
* `--showGuids,-g`:  In addition to showing names, also list each profile's
  guid. These GUIDs should probably be listed _first_ on each line, to make
  parsing output easier.

#### `new-tab`

`new-tab [--initialPosition x,y]|[--maximized]|[--fullscreen] [--initialRows rows] [--initialCols cols] [terminal_parameters]`

Opens a new tab with the given customizations. On it's _first_ invocation, also
opens a new window. Subsequent `new-tab` commands will all open new tabs in the
same window.

**Parameters**:
* `--initialPosition x,y`: Create the new Windows Terminal window at the given
  location on the screen in pixels. This parameter is only used when initially
  creating the window, and ignored for subsequent `new-tab` commands. When
  combined with any of `--maximized` or `--fullscreen`, TODO: what do?
* `--initialRows rows`: Create the terminal window with `rows` rows (in
  characters). If omitted, uses the value from the user's settings. This
  parameter is only used when initially creating the window, and ignored for
  subsequent `new-tab` commands. When combined with any of `--maximized` or
  `--fullscreen`, TODO: what do?
* `--initialCols cols`: Create the terminal window with `cols` cols (in
  characters). If omitted, uses the value from the user's settings. This
  parameter is only used when initially creating the window, and ignored for
  subsequent `new-tab` commands. When combined with any of `--maximized` or
  `--fullscreen`, TODO: what do?
* `[terminal_parameters]`: See [[terminal_parameters]](#terminal_parameters).


#### `split-pane`

`split-pane [--target,-t target-pane] [-h]|[-v] [--percent,-P split-percentage] [terminal_parameters]`

<!-- `--percent,-P` is pretty close to `-p` from `--profile`. Is this okay? -->

Creates a new pane by splitting the given pane vertically or horizontally.

**Parameters**:
* `--target,-t target-pane`: Creates a new split in the given `target-pane`.
  Each pane has a unique index (per-tab) which can be used to identify them. If
  omitted, defaults to `0` (the first pane).
* `-h`, `-v`: Used to indicate which direction to split the pane. `-v` is
  "vertically" (think `[|]`), and `-h` is "horizontally" (think `[-]`). If
  omitted, defaults to vertical. If both `-h` and `-v` are provided, defaults to
  vertical.
* `--percent,-P split-percentage`: Designtates the amount of space that the new
  pane should take, as a percentage of the parent's space. If omitted, the pane
  will take 50% by default.
* `[terminal_parameters]`: See [[terminal_parameters]](#terminal_parameters).



#### `[terminal_parameters]`

Some of the preceeding commands are used to create a new terminal instance.
These commands are listed above as accepting `[terminal_parameters]` as a
parameter. For these commands, `[terminal_parameters]` can be any of the
following:

`[--profile,-p profile-name]|[--guid,-g profile-guid] [--startingDirectory,-d starting-directory] [commandline]`

* `--profile,-p profile-name`: Use the given profile to open the new tab/pane,
  where `profile-name` is the `name` of a profile. If `name` does not match
  _any_ profiles, uses the default. If both `--profile` and `--guid` are
  omitted, uses the default profile.
* `--guid,-g profile-guid`: Use the given profile to open the new tab/pane,
  where `profile-guid` is the `guid` of a profile. If `guid` does not match
  _any_ profiles, uses the default. If both `--profile` and `--guid` are
  omitted, uses the default profile. If both `--profile` and `--guid` are
  specified at the same time, `--guid` takes precedence.
* `--startingDirectory,-d starting-directory`: Overrides the value of
  `startingDirectory` of the specified profile, to start in `starting-directory`
  instead.
* `commandline`: A commadline to replace the default commandline of the selected
  profile. If the user wants to use a `;` in this commandline, it should be
  escaped as `\;`.

### Graceful Upgrading

The entire power of these commandline args is not feasible to accomplish within
the scope of v1.0 of the Windows Terminal. Core to this design is the idea of a
_graceful upgrade_ from 1.0 to some future version, where the full power of
these arguments can be expressed. For the sake of brevity, we'll assume that
future version is 2.0 for the remainder of the spec.

For 1.0, we're focused on primarily [user stories](#User-stories) 1-10, with
8-10 being a lower priority. During 1.0, we won't be focused on supporting
opening multiple tabs or panes straight from the commandline. We'll be focused
on a much simpler grammar of arguments, with the intention that commandlines
from 1.0 will work _without modification_ as 2.0.

For 1.0, we'll restrict ourselves in the following ways:
* We'll only support one command per commandline. This will be one of the
  following list:
  - `help`
  - `version`
  - `new-tab`
  - `open-settings`
* We'll need to make sure that we process commandlines with escaped semicolons
  in them the same as we will in 2.0. Users will need to escape `;` as `\;` in
  1.0, even if we don't support multiple commands in 1.0.
* If users don't provide a command, we'll assume the command was `new-tab`. This will be in-line with


#### Sample 1.0 Commandlines

```sh
# display the help message
wt help
wt --help
wt -h
wt -?

# Display version info for the Windows Terminal
wt version
wt --version
wt -v

# Runs the user's default profile in a new tab, running cmd.exe
wt cmd.exe

# Runs the user's "Windows Powershell" profile in a new tab
wt new-tab --profile "Windows Powershell"
wt --profile "Windows Powershell"

# run "my-commandline.exe with some args" in a new tab
wt new-tab my-commandline.exe with some args
wt my-commandline.exe with some args

# run "my-commandline.exe with some args and a ; literal semicolon" in a new
#  tab, and in another tab, run "another.exe running in a second tab"
wt my-commandline.exe with some args and a \; literal semicolon

# Start the default profile in directory "c:/Users/Foo/dev/MyProject" (user story 8)
wt new-tab --startingDirectory "c:/Users/Foo/dev/MyProject"
wt --startingDirectory "c:/Users/Foo/dev/MyProject"
wt -d "c:/Users/Foo/dev/MyProject"

# Runs the user's "Windows Powershell" profile in a new tab in directory
#  "c:/Users/Foo/dev/MyProject" (user story 2, 8)
wt new-tab --profile "Windows Powershell" --startingDirectory "c:/Users/Foo/dev/MyProject"
wt --profile "Windows Powershell" --startingDirectory "c:/Users/Foo/dev/MyProject"
wt -p "Windows Powershell" -d "c:/Users/Foo/dev/MyProject"

```

## Implementation Details

TODO: I'm leaving this empty for the time being. I want to get some feedback on
the proposal in this state, before starting work on implementation details. This
style of arguments might be controversial, so I want to make sure we settle on a
syntax before I get too far into the details of parsing and passing these args
around.

## Capabilities

### Accessibility

As a commandline feature, the accessibility of this feature will largely be tied
to the ability of the commandline environment to expose accessibility
notifications. Both `conhost.exe` and the Windows Terminal already support
basica accessibility patterns, so users using this feature from either of those
terminals will be reliant upon their accessibility implementations.

### Security

As we'll be parsing user input, that's always subject to worries about buffer
length, input values, etc. Fortunately, most of this should be handled for us by
the operating system, and passed to us as a commandline via `winMain` and
`CommandLineToArgvW`. We should still take extra care in parsing these args.

### Reliability

This change should not have any particular reliability concerns.

### Compatibility

This change should not regress any existing behaviors.

### Performance, Power, and Efficiency

This change should not particularily impact startup time or any of these other categories.

## Potential Issues

#### Commandline escaping

Escaping commandlines is notoriously tricky to do correctly. Since we're using
`;` to delimit commands, which might want to also use `;` in the commandline
itself, we'll use `\;` as an escaped `;` within the commandline. This is an area
we've been caught in before, so extensive testing will be necessary to make sure
this works as expected.

## Future considerations

* These are some additional argument ideas which are dependent on other features
  that might not land for a long time. These features were still considered as a
  part of the design of this solution, though their implementation is purely
  hypothetical for the time being.
    * Instead of launching a new Windows Terminal window, attach this new
      terminal to an existing one. This would require the work outlined in
      [#2080], so support a "manager" process that could coordinate sessions
      like this.
        - This would be something like `wt --session [some-session-id]
          [commands]`, where `--session [some-session-id]` would tell us that
          `[more-commands]` are intended for the given other session/window.
          That way, you could open a new tab in another window with `wt --session
          0 cmd.exe` (for example).
    * `list-sessions`: A command to display all the active Windows terminal
      instances and their session ID's, in a way compatible with the above
      command. Again, heavily dependent upon the implementation of [#2080].
    * `--elevated`: Should it be possible for us to request an elevated session
      of ourselves, this argument could be used to indicate the process should
      launch in an _elevated_ context. This is considered in pursuit of [#632].
    * `--file,-f configuration-file`: Used for loading a configuration file to
      give a list of commands. This file can enable a user to have a re-usable
      configuration saved somewhere on their machine. When dealing with a file
      full of startup commands, we'll assume all of them are intended for the
      given window. So the first `new-tab` in the file will create the window,
      and all subsequent `new-tab` commands will create tabs in that same
      window.
* In the past we've had requests for having the terminal start with multiple
  tabs/panes by default. This might be a path to enabling that scenario. One
  could imagine the `profiles.json` file including a `defaultConfiguration`
  property, with a path to a .conf file filled with commands. We'd parse that
  file on window creation just the same as if it was parsed on the commandline.
  If the user provides a file on the commandline, we'll just ignore that value
  from `profiles.json`.
* When working on "New Window", we'll want the user to be able to open a new
  window with not only the default profile, but also a specific profile. This
  will help us enable that scenario.

## Resources

Feature Request: wt.exe supports command line arguments (profile, command, directory, etc.) [#607]

Add "open Windows terminal here" into right-click context menu [#1060]

Feature Request: Task Bar jumplist should show items from profile [#576]
Draft spec for adding profiles to the Windows jumplist [#1357]

Spec for tab tear off and default app [#2080]

[Question] Configuring Windows Terminal profile to always launch elevated [#632]

New window key binding not working [#2068]

<!-- Footnotes -->

[#576]: https://github.com/microsoft/terminal/issues/576
[#607]: https://github.com/microsoft/terminal/issues/607
[#632]: https://github.com/microsoft/terminal/issues/632
[#1060]: https://github.com/microsoft/terminal/issues/1060
[#1357]: https://github.com/microsoft/terminal/pull/1357
[#2068]: https://github.com/microsoft/terminal/issues/2068
[#2080]: https://github.com/microsoft/terminal/pull/2080