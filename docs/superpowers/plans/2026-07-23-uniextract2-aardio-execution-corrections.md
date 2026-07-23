# UniExtract2 aardio Execution Corrections

> This document is an authoritative correction to the approved design and implementation plan. It records constraints verified against the bundled aardio documentation and the official aardio 43.x package during Task 1.

## 1. Physical library layout

All importable user modules must be stored under the application root `lib\` directory. Namespace names map to paths below `lib\`.

| Namespace | Physical path |
|---|---|
| `app.*` | `lib/app/*.aardio` |
| `core.*` | `lib/core/*.aardio` |
| `handlers.*` | `lib/handlers/*.aardio` |
| `services.*` | `lib/services/*.aardio` |
| `ui.*` | `lib/ui/*.aardio` |
| `tests.*` helpers | `lib/tests/*.aardio` |

Therefore, every implementation-plan path beginning with `app/`, `core/`, `handlers/`, `services/`, or `ui/` must be interpreted with a leading `lib/`.

Test entry scripts remain under `tests/`, but they must be executed with the repository root as the aardio application root so imports resolve from `lib\`.

## 2. Namespace global access

Inside a custom namespace, aardio global objects and built-ins must be accessed through the parent/global prefix `..` unless the module explicitly imports them into the namespace.

Examples:

```aardio
namespace tests.testlib;

..assert(condition,"message");
..io.stdout.write("text");
var text = ..tostring(value);
```

Implementation must not copy earlier examples that use unqualified `io`, `assert`, `tostring`, `string`, `table`, or other globals inside a custom namespace.

## 3. Project file

`default.aproj` embeds the single `lib` folder for application modules. Runtime data directories remain local/non-embedded:

- `rules`
- `config`
- `lang`
- `passwords`
- `bin`
- `def`
- `support`

`tests` remains ignored for publishing.

## 4. Automated verification

The official `aardio.exe` does not expose a stable headless `/run`, `/compile`, or `/publish` command-line interface. CI uses an ephemeral modification of the official IDE startup trigger and calls:

```aardio
ide.createProcessEx(appRoot,,testScript);
```

This starts the test source with the repository root as its application root, without UI automation. The trigger modification occurs only in the downloaded CI copy of aardio and is never committed or distributed.

## 5. Precedence

These corrections override conflicting physical paths and namespace examples in:

- `docs/superpowers/specs/2026-07-23-uniextract2-aardio-design.md`
- `docs/superpowers/plans/2026-07-23-uniextract2-aardio-implementation.md`

The approved functional scope and architectural boundaries are unchanged.
