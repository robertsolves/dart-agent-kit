# dart-agent-kit

Commands, skills and a reviewer subagent that make AI coding agents write better Dart and Flutter. One install script sets them up for Claude Code, Codex, Gemini CLI, Antigravity, OpenCode, Kilo, Cursor, Copilot, Windsurf and Freebuff.

## Why

I write Flutter for a living and use agents every day. Left alone, they write Dart that compiles and still causes trouble later. They add `late` fields that crash at runtime, use `Map<String, dynamic>` all the way through the app, put string keys where an enum should be, write `_buildHeader()` methods instead of widgets, and add a second state library because they didn't check which one the app already uses.

These are the rules I ended up giving every agent, plus the commands that check code against them. All of them read one rule file, `.agents/rules/dart-discipline.md`. Change it and every agent follows the new rules.

## Install

From the root of your project:

```sh
curl -fsSL https://raw.githubusercontent.com/robertsolves/dart-agent-kit/main/install.sh | bash
```

Or clone it and point it at your project:

```sh
git clone https://github.com/robertsolves/dart-agent-kit
./dart-agent-kit/install.sh --project ~/code/my_app
```

By default it sets up the agents it can find in your project or on your PATH. Some useful flags:

| Flag | What it does |
|---|---|
| `--tools all` | set up every supported agent, not only the ones it finds |
| `--tools claude,codex` | set up only these |
| `--only dart-audit-code,dart-reviewer` | install some of it, not all |
| `--dry-run` | show what would change |
| `--copy` | copy files into tool folders instead of symlinking |
| `--list` | show what's in the kit |

Everything goes into `.agents/` in your project. The tool folders (`.claude/`, `.opencode/`, `.kilo/`, `.gemini/` and so on) only get links or small pointer files. Commit all of it so your team gets the same setup.

The commands use `rg` ([ripgrep](https://github.com/BurntSushi/ripgrep)), so install that first.

## What's in it

| Name | Type | What it does |
|---|---|---|
| `dart-audit-code` | command | Reads Dart code and reports breaches of the rules. Changes nothing. |
| `dart-improve-code` | command | One careful pass that hardens and then simplifies code, without changing behaviour. |
| `dart-clean-packages` | command | Runs `dart fix`, `dart format` and `dart analyze` over every package, then greps for rule breaches. Leaves generated files alone. |
| `dart-discipline` | skill | Loads the project rules and the relevant Effective Dart section whenever the agent touches Dart. |
| `dart-search` | skill | Two small scripts. One finds where a symbol is declared and used. The other is a grep with capped output, so it doesn't fill the context. |
| `dart-reviewer` | subagent | A read-only reviewer that returns a ranked list of problems. Good before a PR. |
| `dart-discipline.md` | rule | The rules themselves. Edit this file to suit your project. |

The skill ships with a copy of the [Effective Dart](https://dart.dev/effective-dart) guides, so agents can quote them rather than go from memory.

## Running a command

| Agent | How |
|---|---|
| Claude Code, OpenCode, Kilo, Gemini CLI, Antigravity, Cursor, Copilot, Windsurf | `/dart-audit-code lib/features/auth` |
| Codex | `$dart-audit-code lib/features/auth` |
| Freebuff | `/skill:dart-audit-code lib/features/auth` |
| Anything else | "Read `.agents/commands/dart-audit-code.md` and follow it for lib/features/auth" |

In Claude Code, OpenCode and Kilo you can also ask for the reviewer by name: "use the dart-reviewer agent on the current diff".

## Project facts

The commands work in any project because they never hard-code a path. When one needs your test command, your generated-file patterns or the packages that are out of scope, it reads `.agents/project-facts.md`. The installer creates that file with placeholders. Fill it in by hand, or run `/setup-project-facts` from [agent-commands](https://github.com/robertsolves/agent-commands). A command that finds a placeholder stops and asks you instead of guessing.

## FVM

If your repo has a `.fvmrc`, `dart-clean-packages` runs `fvm dart` and `fvm flutter` instead of plain `dart` and `flutter`, and the skill checks with `fvm dart analyze`.

## Updating and removing

Run the same install command again to update. It replaces what it installed last time and leaves your own files alone. It won't overwrite a file of the same name it didn't create unless you pass `--force`. Don't edit the installed commands, because an update overwrites them. Put project-specific rules in `dart-discipline.md` or `project-facts.md`.

To remove everything:

```sh
curl -fsSL https://raw.githubusercontent.com/robertsolves/dart-agent-kit/main/install.sh | bash -s -- --uninstall
```

`project-facts.md` stays, since you may have filled it in.

## Windows

The script is bash. Run it from Git Bash or WSL. Where it can't make symlinks it copies files instead.

## Strict analysis options

The kit also ships the analyzer settings I use on every Dart and Flutter project. `dart.yaml` lists every rule the Dart linter has, on or off, and every rule that's off has a comment saying why. It turns on `strict-casts`, `strict-inference` and `strict-raw-types` too. `flutter.yaml` is the same plus Flutter's generated folders.

After installing, point your packages at them:

```sh
bash .agents/skills/dart-discipline/scripts/use_strict_analysis.sh --all
```

That adds one `include:` line to each package's `analysis_options.yaml`, picking `flutter.yaml` or `dart.yaml` from its `pubspec.yaml`. If a package already includes something like `flutter_lints`, it leaves it alone and tells you. Add `--replace` to swap it. Turn single rules off in your own file under `linter: rules:`. Expect a lot of warnings the first time on an existing codebase. `dart fix --apply` clears many of them. You need Dart 3.10 or later.

## Combine it with the other kits

This is one of three kits that install side by side: [agent-project-template](https://github.com/robertsolves/agent-project-template), [agent-commands](https://github.com/robertsolves/agent-commands) and [dart-agent-kit](https://github.com/robertsolves/dart-agent-kit). To add any mix in one line:

```sh
curl -fsSL https://raw.githubusercontent.com/robertsolves/agent-commands/main/kit.sh | bash -s -- add template commands dart
```

Or run `... kit.sh | bash -s -- setup` once, and then just ask your agent to "add the dart kit" in any project.

## Related

- [agent-commands](https://github.com/robertsolves/agent-commands): language-agnostic audit and cleanup commands (band-aids, dead code, exploits, stale references and more).

## Licence

MIT
