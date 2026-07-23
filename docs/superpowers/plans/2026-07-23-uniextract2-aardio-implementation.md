# UniExtract2 aardio Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the AutoIt runtime with a pure aardio x86 application that provides a reliable single-task extraction queue, strict INI rules, shared CLI/GUI adapters, and the approved first-release format set.

**Architecture:** Build one UI-independent engine around task tables, a strict rule loader, deterministic detection/dispatch, and a `process.popen` runner bound to a per-task `process.job.limitKill` object. GUI, CLI, and silent execution are thin interaction adapters over the same queue and executor.

**Tech Stack:** aardio `win.ui`, `thread`, `thread.var`, `process.popen`, `process.job.limitKill`, `string.ini`, `fsys`, standard INI files, and the existing UniExtract2 helper binaries.

## Global Constraints

- Target Windows 10 and Windows 11 only.
- Publish the aardio main executable as x86.
- Do not call, package, or fall back to the legacy AutoIt `UniExtract.exe`.
- Keep third-party extractors in local published folders; do not embed them into the EXE.
- Use a redesigned native `win.form` UI.
- Use one FIFO extraction worker; no concurrent extraction in the first release.
- Use portable storage only under the application root; fail startup when required directories are not writable.
- Store INI and language files as UTF-8 without BOM.
- Support `zh-CN`, `zh-TW`, and `en-US`; missing keys fall back to English.
- Keep UI work on the UI thread. Worker imports and inputs are explicit.
- Pass arguments as arrays to `process.popen`; normal extraction must not use `cmd.exe`.
- Test from a clean published directory before removing legacy entry files.
- Source baseline: `docs:guide/language/thread.md`, `docs:library-guide/std/process/process.md`, `lib:process/popen.aardio`, `lib:process/job/limitKill.aardio`, `lib:string/ini.aardio`, `docs:library-guide/builtin/io/path.md`, `docs:guide/ide/file.md`.

## Delivery Gates

1. **Gate A — Foundation:** paths, strict INI, rules, task models, and process-tree cancellation pass tests.
2. **Gate B — Core CLI:** ZIP, 7Z, RAR, TAR, GZIP, BZIP2, XZ, CAB, ISO, and WIM work through the shared CLI queue.
3. **Gate C — First-release behavior:** native GUI, languages, passwords, Inno, NSIS, MSI, MSU, DMG, and PE scan work.
4. **Gate D — Release:** a clean published directory passes the acceptance matrix and contains no AutoIt runtime entry point.

---

### Task 1: Project skeleton and test harness

**Files:**
- Create: `main.aardio`
- Create: `default.aproj`
- Create: `app/constants.aardio`
- Create: `tests/run.aardio`
- Create: `tests/testlib.aardio`

**Interfaces:**
- Produces: `tests.testlib.run(name,fn)`, `tests.testlib.expect(actual,expected,message)`, `app.constants.EXIT_*`.

- [ ] Write a failing smoke test importing `app.constants` and asserting `APP_NAME == "UniExtract2"` and `EXIT_NOT_WRITABLE == 8`.
- [ ] Run `tests/run.aardio` with **F5**; expect an import failure.
- [ ] Add `tests.testlib` with a narrow `try/catch` around each test and `assert`-based comparisons.
- [ ] Add `app.constants` with exit codes 0–9 exactly as approved.
- [ ] Add a minimal `win.form` startup file that contains no domain logic.
- [ ] Add `default.aproj` with `ui="win"`, `output="UniExtract.exe"`, `publishDir="/dist/"`, `dstrip="false"`; mark `rules`, `config`, `lang`, `passwords`, `bin`, `def`, and `support` as `local="true"`; mark `tests` ignored.
- [ ] Run the smoke test; expect `ALL TESTS PASSED`.
- [ ] Commit:

```bash
git add main.aardio default.aproj app/constants.aardio tests
git commit -m "build: add aardio project skeleton and test harness"
```

---

### Task 2: Application-root paths and writable startup checks

**Files:**
- Create: `app/paths.aardio`
- Create: `app/environment.aardio`
- Create: `tests/paths.aardio`
- Modify: `tests/run.aardio`

**Interfaces:**
- Produces: `app.paths.root()`, `app.paths.full(relativePath)`, `app.paths.isInsideRoot(path)`, `app.environment.ensurePortableDirectories() -> true | null,error`.

- [ ] Add tests proving `/config/settings.ini` resolves inside the application root and `C:\Windows\system32\cmd.exe` is outside.
- [ ] Add a test that creates and removes a real probe file in `config`, `logs`, `temp`, and `passwords`.
- [ ] Implement path conversion with `io.fullpath`; normalize separators only after conversion.
- [ ] Implement real write probes with `io.open(...,"wb")`; do not infer writability from ACL metadata alone.
- [ ] Return `null,error` naming the exact directory on failure.
- [ ] Run tests from both the IDE project and a published copy.
- [ ] Commit:

```bash
git add app/paths.aardio app/environment.aardio tests/paths.aardio tests/run.aardio
git commit -m "feat: add portable path and writable-directory checks"
```

---

### Task 3: Strict UTF-8 INI loading and atomic configuration saving

**Files:**
- Create: `core/strictIni.aardio`
- Create: `services/config.aardio`
- Create: `config/settings.ini`
- Create: `tests/strictIni.aardio`

**Interfaces:**
- Produces: `core.strictIni.load(path) -> table | null,error`, `services.config.defaults()`, `load()`, `save(settings)`.

- [ ] Add tests for duplicate sections, duplicate keys, malformed lines, BOM rejection, save/reload, and old-file preservation after a forced save failure.
- [ ] Pre-scan raw text before calling `string.ini.parse`; include file, section, key, and line in errors.
- [ ] Reject UTF-8 BOM bytes `EF BB BF`.
- [ ] Save to `settings.ini.tmp`, close the file, move the old file to `.bak`, move the temporary file into place, then remove `.bak`.
- [ ] On replacement failure, restore `.bak` and return `null,error`.
- [ ] Keep the approved sections and defaults: `general`, `output`, `scan`, `password`, `queue`, `window`.
- [ ] Run tests; inspect saved bytes to confirm no BOM.
- [ ] Commit:

```bash
git add core/strictIni.aardio services/config.aardio config/settings.ini tests
git commit -m "feat: add strict ini loading and atomic config saves"
```

---

### Task 4: Deterministic rule loading and validation

**Files:**
- Create: `core/ruleLoader.aardio`
- Create: `rules/extensions.ini`
- Create: `rules/signatures.ini`
- Create: `rules/extractors.ini`
- Create: `rules/priorities.ini`
- Create: `rules/aliases.ini`
- Create: `tests/ruleLoader.aardio`

**Interfaces:**
- Produces: `core.ruleLoader.loadAll() -> {extensions,signatures,extractors,priorities,aliases} | null,error`.

- [ ] Add tests for valid ZIP/7Z mappings, illegal hex, negative offset, missing handler, unknown template variable, and an extractor path outside the application root.
- [ ] Add compound-extension mappings before simple mappings, including `tar.gz` and `part1.rar`.
- [ ] Add signatures for 7Z, ZIP, RAR4, RAR5, CAB, and MSI.
- [ ] Add `extractor.7zip` with numbered argument keys and explicit success/warning exit codes.
- [ ] Validate even-length hexadecimal signatures, integer offsets, positive timeouts, allowed placeholders, referenced handler names, and in-root executable paths.
- [ ] Do not require optional helper binaries at startup; do require all helpers needed for Gate B.
- [ ] Run tests; expect deterministic in-memory tables.
- [ ] Commit:

```bash
git add core/ruleLoader.aardio rules tests
git commit -m "feat: add strict extraction rule loader"
```

---

### Task 5: Task/result models and output conflict policies

**Files:**
- Create: `core/task.aardio`
- Create: `core/result.aardio`
- Create: `core/outputPolicy.aardio`
- Create: `tests/taskOutput.aardio`

**Interfaces:**
- Produces: `core.task.create(inputPath,options)`, `core.result.success/failure/canceled`, `core.outputPolicy.resolve(path,policy)`.

- [ ] Add tests for missing input, invalid mode, initial state `queued`, stable fields, and `rename` producing `_2`, `_3` without deleting existing directories.
- [ ] Define the approved task fields: `id`, `inputPath`, `outputPath`, `tempPath`, `fileName`, `fileBase`, `fileExt`, `mode`, `state`, `candidates`, `selectedFormat`, `selectedExtractor`, `passwordIndex`, `processId`, `progress`, `startedAt`, `finishedAt`, `result`, `warnings`.
- [ ] Implement policies `ask`, `merge`, `replace`, `rename`; `ask` returns an interaction-required error to non-GUI adapters.
- [ ] Never delete an existing directory during path resolution.
- [ ] Run tests.
- [ ] Commit:

```bash
git add core/task.aardio core/result.aardio core/outputPolicy.aardio tests
git commit -m "feat: add task models and output policies"
```

---

### Task 6: Process runner with separate streams, timeout, and process-tree cancellation

**Files:**
- Create: `core/processRunner.aardio`
- Create: `tests/helpers/spawn-child.aardio`
- Create: `tests/processRunner.aardio`

**Interfaces:**
- Produces: `core.processRunner.run(spec,cancelVar,onOutput) -> {started,exitCode,stdout,stderr,timedOut,canceled,durationMs,processId,error} | null,error`.
- `spec`: `{executable,arguments,workDir,timeoutMs,codepage}`.

- [ ] Publish a test helper that writes one line to stdout, one to stderr, starts a long-running child, and waits.
- [ ] Add tests for stream capture, launch failure, timeout, explicit cancellation, exit code, and child-process cleanup.
- [ ] Create a dedicated `process.job.limitKill()` object for each run.
- [ ] Start with `process.popen(executable,arguments,{workDir=...})`; never concatenate untrusted paths into one command string.
- [ ] Bind the `process.popen` object to the job before entering the read loop.
- [ ] Poll `peek(0)`, append stdout/stderr separately, and call `onOutput(out,err)`.
- [ ] On cancel or timeout, close the job handle to terminate the entire job tree; then close the pipe object.
- [ ] Verify in Task Manager that no helper child remains.
- [ ] Do not start Task 8 until this test passes repeatedly.
- [ ] Commit:

```bash
git add core/processRunner.aardio tests/helpers tests/processRunner.aardio tests/run.aardio
git commit -m "feat: add cancellable process-tree runner"
```

---

### Task 7: Extension/signature detection and deterministic dispatch

**Files:**
- Create: `core/detector.aardio`
- Create: `core/dispatcher.aardio`
- Create: `tests/detector.aardio`

**Interfaces:**
- Produces candidates `{format,detector,rulePriority,score,handler,extractor}`; `dispatcher.select(candidates,rules)`.

- [ ] Add tests for wrong extensions, longest compound extension, signature beating extension, and equal-score ambiguity.
- [ ] Read only the bytes required by configured signatures; never read an entire large file for fixed signatures.
- [ ] Compute score as detector weight + format weight + rule priority.
- [ ] Tie order: dedicated handler, signature over extension, more specific format, otherwise interaction required.
- [ ] Keep candidate order independent of hash-table iteration by sorting dense arrays.
- [ ] Run tests.
- [ ] Commit:

```bash
git add core/detector.aardio core/dispatcher.aardio tests/detector.aardio tests/run.aardio
git commit -m "feat: add deterministic file detection and dispatch"
```

---

### Task 8: Safe argument templates, output verification, and generic 7-Zip extraction

**Files:**
- Create: `core/argumentTemplate.aardio`
- Create: `core/outputVerifier.aardio`
- Create: `handlers/generic/archive.aardio`
- Create: `tests/genericArchive.aardio`

**Interfaces:**
- Produces: `argumentTemplate.expand(rule,context) -> string[]`, `outputVerifier.snapshot/hasChanges`, `handlers.generic.archive.extract(task,extractor,context)`.

- [ ] Add tests proving paths with spaces, `&`, parentheses, and Chinese characters remain one process argument.
- [ ] Allow only `{input}`, `{output}`, `{temp}`, `{password}`, `{programDir}`, `{fileName}`, `{fileBase}`, `{fileExt}`.
- [ ] Reject any unknown placeholder.
- [ ] Snapshot output file count and total bytes before extraction; success requires a new/changed output unless the handler explicitly allows an empty archive.
- [ ] Call only `core.processRunner`; do not implement a second process loop.
- [ ] Treat 7-Zip exit code 0 as success and 1 as warning requiring output validation; other codes fail.
- [ ] Add a tiny licensed ZIP fixture containing `hello.txt`.
- [ ] Run extraction under Unicode and spaced paths.
- [ ] Commit:

```bash
git add core/argumentTemplate.aardio core/outputVerifier.aardio handlers/generic/archive.aardio tests
git commit -m "feat: add safe argument expansion and generic archive handler"
```

---

### Task 9: Core archive mappings, passwords, and multipart normalization

**Files:**
- Create: `core/passwordProvider.aardio`
- Create: `handlers/archive/encrypted.aardio`
- Create: `handlers/archive/multipart.aardio`
- Create: `passwords/passwords.txt`
- Modify: `rules/extensions.ini`
- Modify: `rules/extractors.ini`
- Create: `tests/passwordMultipart.aardio`

**Interfaces:**
- Produces: ordered deduplicated passwords, `multipart.firstVolume(path)`, encrypted retry returning `succeeded`, `failed`, `canceled`, or `password_required`.

- [ ] Add tests for password order/deduplication, maximum attempts, `.part03.rar -> .part01.rar`, `.003 -> .001`, and missing first volume.
- [ ] Retry only after a handler classifies the failure as a password error; never convert every failure into a password prompt.
- [ ] Mask passwords in all logs and command previews.
- [ ] Clear only the current attempt's temporary output between retries.
- [ ] Map ZIP, 7Z, RAR, TAR, GZIP, BZIP2, XZ, CAB, ISO, and WIM to validated helpers while preserving distinct format IDs.
- [ ] Add fixtures for normal, corrupt, encrypted, empty, multipart, wrong extension, and existing output cases.
- [ ] Pass Gate B CLI extraction before continuing.
- [ ] Commit:

```bash
git add core/passwordProvider.aardio handlers/archive passwords rules tests
git commit -m "feat: add passwords multipart archives and core format mappings"
```

---

### Task 10: Executor and one-worker FIFO queue

**Files:**
- Create: `core/executor.aardio`
- Create: `core/taskQueue.aardio`
- Create: `tests/taskQueue.aardio`

**Interfaces:**
- Produces: `executor.execute(task,runtime)`, queue methods `add`, `start`, `pauseAfterCurrent`, `cancelCurrent`, `clearWaiting`, `move`.

- [ ] Add fake-executor tests for FIFO order, failure continuation, cancellation continuation, duplicate policy, pause-after-current, move-up/down, and inability to clear a running task.
- [ ] Make the executor the only owner of task state transitions.
- [ ] Emit plain table events: `task_state`, `task_progress`, `task_log`, `task_result`, `queue_state`, `password_request`, `method_request`.
- [ ] Use one `thread.invoke` worker and a `thread.var` cancellation flag; import worker dependencies inside the worker.
- [ ] Define pause as “finish the current task, then start no next task.” Do not suspend external process threads.
- [ ] Release `thread.var` when the queue is disposed.
- [ ] Run tests and a two-file real extraction.
- [ ] Commit:

```bash
git add core/executor.aardio core/taskQueue.aardio tests/taskQueue.aardio tests/run.aardio
git commit -m "feat: add executor and single-worker fifo queue"
```

---

### Task 11: Shared CLI adapter and exit-code aggregation

**Files:**
- Create: `app/arguments.aardio`
- Create: `app/cli.aardio`
- Create: `tests/arguments.aardio`
- Modify: `main.aardio`

**Interfaces:**
- Produces: `arguments.parse(argv) -> options | null,error`, `cli.run(options) -> exitCode`.

- [ ] Add exact parsing tests for `/extract`, `/scan`, `/output`, `/sub`, `/here`, `/password-file`, `/conflict`, `/silent`, `/log`, `/format`, `/method`, `/no-open`, and `/?`.
- [ ] Reject unknown switches, missing switch values, and invalid conflict policies.
- [ ] Use `rename` when silent mode has no explicit conflict policy.
- [ ] Route GUI and CLI through the same task factory, queue, detector, executor, and handlers.
- [ ] Aggregate exit codes exactly: 0 all success; 1 any task failure; 2 argument error; 3 missing input; 4 config/rules; 5 helper missing; 6 canceled; 7 password required; 8 not writable; 9 internal.
- [ ] Run published CLI tests; verify a mixed batch returns 1.
- [ ] Commit:

```bash
git add app/arguments.aardio app/cli.aardio main.aardio tests
git commit -m "feat: add shared command-line adapter"
```

---

### Task 12: Localization, logging, and native main-form shell

**Files:**
- Create: `services/language.aardio`
- Create: `services/logger.aardio`
- Create: `lang/zh-CN.ini`
- Create: `lang/zh-TW.ini`
- Create: `lang/en-US.ini`
- Create: `ui/mainForm.aardio`
- Modify: `main.aardio`
- Create: `tests/language.aardio`

**Interfaces:**
- Produces: `language.load(locale)`, `language.t(key)`, `logger.write(level,module,taskId,message)`, `ui.mainForm.create(appContext)`.

- [ ] Add tests for all English keys, Chinese overrides, and English fallback.
- [ ] Write logs as `[time] [taskId] [module] LEVEL message`; never include actual passwords.
- [ ] Create one designer-owned `DSG` block with toolbar, queue list, details panel, total-progress row, and rich-edit log.
- [ ] Keep handlers and queue/process logic outside the form module.
- [ ] Bind toolbar actions only to queue/application commands.
- [ ] Verify resize behavior and that adding many files does not block the UI thread.
- [ ] Commit:

```bash
git add services/language.aardio services/logger.aardio lang ui/mainForm.aardio main.aardio tests
git commit -m "feat: add localization logging and native main form"
```

---

### Task 13: Progress parsing and task-detail events

**Files:**
- Create: `core/progressParser.aardio`
- Create: `tests/progressParser.aardio`
- Modify: `core/processRunner.aardio`
- Modify: `ui/mainForm.aardio`

**Interfaces:**
- Produces: `progressParser.parse(text) -> {percent,current,total} | null`.

- [ ] Add tests for `35%`, `[3 on 10]`, `3 of 10`, `3/10`, invalid values, and unrelated output.
- [ ] Clamp percentages to 0–100.
- [ ] Emit progress only when a stable pattern is found; otherwise keep an indeterminate progress bar.
- [ ] Update only the matching task row and currently selected detail panel through the UI proxy.
- [ ] Run a real 7-Zip extraction; verify no false 100% on failure.
- [ ] Commit:

```bash
git add core/progressParser.aardio core/processRunner.aardio ui/mainForm.aardio tests
git commit -m "feat: add extraction progress parsing"
```

---

### Task 14: Dedicated Inno, NSIS, MSI/MSU, DMG, and PE handlers

**Files:**
- Create: `handlers/installer/inno.aardio`
- Create: `handlers/installer/nsis.aardio`
- Create: `handlers/installer/msi.aardio`
- Create: `handlers/image/dmg.aardio`
- Create: `handlers/executable/peScan.aardio`
- Create: `docs/migration/helper-command-map.md`
- Create: `tests/specialHandlers.aardio`
- Modify: `rules/extractors.ini`
- Modify: `rules/signatures.ini`

**Interfaces:**
- Each handler exports `detect(task,context)` where deeper detection is required and `extract(task,extractor,context)`.

- [ ] Inventory exact helper executable, working directory, arguments, stable error markers, and output validation from `UniExtract.au3`; cite the legacy constant/function for every row in `helper-command-map.md`.
- [ ] Do not add a rule until the helper exists under `bin/` and its command has been manually run on a licensed fixture.
- [ ] Add normal and corrupt fixtures for each handler.
- [ ] Keep handlers as thin orchestration over `core.processRunner`; do not duplicate timeout, cancellation, stream, or logging loops.
- [ ] Ensure PE scan never executes the analyzed input.
- [ ] Test filenames containing spaces, Chinese characters, parentheses, and `&` to prove argument isolation.
- [ ] Require expected output files, not exit code alone.
- [ ] Pass Gate C format tests.
- [ ] Commit:

```bash
git add handlers rules docs/migration/helper-command-map.md tests
git commit -m "feat: add installer image and pe handlers"
```

---

### Task 15: Settings dialogs and shell integration

**Files:**
- Create: `ui/settingsForm.aardio`
- Create: `ui/passwordForm.aardio`
- Create: `ui/scanResultForm.aardio`
- Create: `services/shellIntegration.aardio`
- Modify: `ui/mainForm.aardio`
- Modify: `services/config.aardio`

**Interfaces:**
- Modal forms return plain values/tables; shell service exports install/remove for current user and explicit all-users installation.

- [ ] Add settings round-trip tests for language, conflict policy, password file, scan limits, and queue continuation.
- [ ] Password dialog returns accepted/password or canceled; it contains no extraction logic.
- [ ] Method dialog receives candidates and returns one candidate ID.
- [ ] Install current-user verbs for extract dialog, extract here, extract to subdirectory, and scan.
- [ ] Quote the published EXE and use documented CLI switches.
- [ ] All-users installation may elevate only after the user explicitly requests it; startup never auto-elevates.
- [ ] Verify install/remove idempotence and ownership boundaries.
- [ ] Commit:

```bash
git add ui services/shellIntegration.aardio tests
git commit -m "feat: add settings dialogs and shell integration"
```

---

### Task 16: Full fixture matrix and clean-directory release verification

**Files:**
- Create: `tests/manifests/fixtures.ini`
- Create: `tests/integration/allFormats.aardio`
- Create: `tests/integration/cancellation.aardio`
- Create: `tests/integration/cli.aardio`
- Create: `docs/release-checklist.md`
- Modify: `default.aproj`

**Interfaces:**
- Produces a fixture manifest with path, SHA-256, expected format, expected status, and expected output.

- [ ] Add every approved first-release format and cases for corrupt data, wrong extension, Unicode path, spaces, long names, read-only input, passwords, multipart, empty archive, and existing output.
- [ ] Run all unit/integration tests from the IDE.
- [ ] Publish with `dstrip="false"` to `dist/`.
- [ ] Copy `dist/` outside the repository and verify `UniExtract.exe`, `bin`, `rules`, `lang`, `config`, and `passwords` exist.
- [ ] Run scan, silent extraction, encrypted extraction, cancellation, and mixed-batch CLI tests from the clean directory.
- [ ] Verify no IDE path appears in logs and no child process remains after cancellation.
- [ ] Test a non-writable copy; expect a clear error and exit code 8, with no AppData fallback or elevation.
- [ ] Commit:

```bash
git add tests/manifests tests/integration docs/release-checklist.md default.aproj
git commit -m "test: add full format and clean-release acceptance matrix"
```

---

### Task 17: Remove legacy runtime entry points and finalize documentation

**Files:**
- Delete after Gate D: `UniExtract.au3`
- Delete after Gate D: legacy AutoIt updater entry files that invoke or replace `UniExtract.exe`
- Modify: `README.md`
- Modify: `docs/COMMAND-LINE.md`
- Modify: `docs/FORMATS.md`
- Create: `docs/ARCHITECTURE.md`
- Create: `docs/MIGRATION.md`

**Interfaces:**
- Produces a branch whose only runnable main application is the aardio project.

- [ ] Confirm the release contains no `.au3`, AutoIt compiler/runtime file, legacy `UniExtract.exe`, legacy-spawn code, tests, or fixtures.
- [ ] Use `git rm` only after all Gate D tests pass; keep history in Git rather than creating a `legacy-autoit/` copy.
- [ ] Document Windows 10/11 x86, portable-only storage, write checks, languages, FIFO and pause semantics, CLI exit codes, format coverage, and third-party licenses.
- [ ] Re-run the published acceptance matrix after deletion.
- [ ] Commit:

```bash
git add README.md docs
git rm UniExtract.au3
git commit -m "refactor: complete pure aardio runtime migration"
```

---

## Self-Review

### Coverage

- Pure aardio/no fallback: Tasks 1, 16, 17.
- Windows 10/11 x86/local resources: Tasks 1, 16.
- Portable write checks: Tasks 2, 16.
- Strict INI/rules: Tasks 3, 4.
- Process safety and child cleanup: Task 6.
- Detection/dispatch: Task 7.
- Core archives/passwords/multipart: Tasks 8, 9.
- FIFO/pause/cancel: Task 10.
- CLI/exit codes: Task 11.
- UI/languages/logs/progress: Tasks 12, 13.
- Installers/images/PE: Task 14.
- Settings/context menus: Task 15.
- Release and legacy removal: Tasks 16, 17.

### Type consistency

- Task fields are defined once in Task 5.
- Process-result fields are defined once in Task 6.
- Candidate fields are defined once in Task 7.
- Handler status values are `succeeded`, `failed`, `canceled`, or `password_required`.
- Thread/UI messages are plain tables and never contain native controls, COM objects, process objects, or closure-dependent instances.

### Stop conditions

- Do not start format integration until Task 6 proves child cleanup.
- Do not start special handlers until Gate B passes.
- Do not delete AutoIt entry points until the clean published acceptance matrix passes.