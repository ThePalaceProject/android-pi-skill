---
name: palace-android
description: Conventions and build guardrails for PalaceProject (org.thepalaceproject) Android/Kotlin projects. Use when working on, editing, or building any project whose gradle.properties declares a GROUP containing org.thepalaceproject, or whose pom.xml has a groupId containing org.thepalaceproject. Covers the only permitted build and format commands, where the enforced ktlint rules live (and the .editorconfig settings you must not introduce), the house Kotlin code style, the style conventions no linter checks, and the API/dependency preference (JVM over Android).
---

# PalaceProject android conventions

This skill is a **guardrail** for working on *existing* PalaceProject Android
projects. It is not for scaffolding new projects.

PalaceProject Android projects are **Kotlin + Gradle** (with the Android
Gradle Plugin). A few related PalaceProject projects are plain Kotlin +
Maven. The build/format commands below cover both; the Kotlin style rules
apply to both.

## Detect that you are in a PalaceProject project

You are in a PalaceProject project if **any** of these hold:

- the root `gradle.properties` has a `GROUP` property whose value contains
  `org.thepalaceproject` (e.g. `GROUP=org.thepalaceproject`,
  `GROUP=org.thepalaceproject.audiobook`)
- the root `pom.xml` has a `groupId` containing `org.thepalaceproject`

## Build commands — hard rules

The **only** build commands you are permitted to run are:

- **Gradle project:**
  ```
  ./gradlew clean assemble test
  ```
- **Maven project:**
  ```
  mvn clean verify
  ```

That is the complete list. Do **not** invent or "helpfully add" other tasks
or goals, no `-x`/`--exclude-task`, no single-module selection, no
`--tests` filters, no `assembleDebug` variants, on your own initiative.
Running a made-up task that fails, then debugging the failure, wastes time
a permitted command would not have.

The permitted **formatting** command is:

- **Gradle project:**
  ```
  ./gradlew ktlintFormat
  ```

`ktlintFormat` rewrites source files in place, so after running it check
`git status` and the diff — it can modify files you did not edit by hand.
Do not hand-format to match the enforced style; run `ktlintFormat` and let
ktlint do it.

The ktlint *check* task is `./gradlew ktlintCheck`; it is wired into the
build (when `org.thepalaceproject.build.enableKtLint=true` in
`gradle.properties`, the `clean` task depends on `ktlintCheck`), so a normal
build already gates on style. You do not need to run it separately unless
asked.

### Always log build output to a file

Every build you run must be captured in a log file, and you inspect the
file. You **never** pipe the build into `grep` or any other filter:

```
mktemp /tmp/palace-build-XXXXXX.log
# => /tmp/palace-build-a1b2c3.log   (your path will differ)

./gradlew clean assemble test > /tmp/palace-build-a1b2c3.log 2>&1
```

Use that same literal path in every subsequent command: a shell variable
does not survive across separate command invocations, but the file does. Do
not use `$(mktemp ...)` inside the redirection — that hides the path from
you entirely.

Then inspect the log as many times as needed:

```
tail -n 40 <the log file>
grep -n "error:" <the log file>
```

This is a hard rule because a piped invocation
(`./gradlew clean assemble test | grep error`) destroys the evidence: the
output is consumed by the pipe, and the exit status you see is `grep`'s, not
the build's. If the pattern is wrong and matches nothing, you cannot tell
what the build did — and the only way to find out is to run the build
again. With a log file, the build runs exactly **once**; every subsequent
inspection is a cheap `grep` / `read` / `tail` of the file.

The log path must be **unique per build** (`mktemp` guarantees this
atomically) — other agents or sessions build other projects concurrently —
and it must not be written into the project's working tree.

## Where the enforced style lives (do NOT duplicate it)

The enforced Kotlin style is a single canonical file per project: the
repository root **`.editorconfig`**. ktlint reads it. The ktlint version is
pinned by the build convention (currently 1.8.0, in the
`palace-build-conventions` submodule via the `org.thepalaceproject.ktlint`
plugin) — do not run a downloaded or system ktlint binary of a different
version against the tree; the results will not match what the
build enforces.

**Do not restate the rule list from memory.** When in doubt about what is
enforced, read the repository's `.editorconfig`. When in doubt about what a
violation means, run `./gradlew ktlintFormat` and read the diff.

### .editorconfig levers you must NOT introduce

These settings look harmless and destroy the house style. Never add them,
even if a rule you are enabling "needs" them:

- **`ktlint_code_style = intellij_idea`** — do not set it. Leave the code
  style unset (the default `ktlint_official`). Under `intellij_idea`,
  ktlint's `function-signature` autocorrect collapses multi-line function
  signatures onto a single line whenever they fit the line limit — the
  opposite of how this codebase writes signatures (see the house style
  section below) — and it rewrites every affected signature in the tree at
  once.
- **`ktlint_function_signature_rule_force_multiline_when_parameter_count_greater_or_equal_than`** —
  do not set it. It makes the signature rules rewrite function signatures
  across the tree.
- **`ktlint_standard_chain-wrapping`** — do not enable it. It forbids
  method chains whose continuation lines begin with the operator (for
  example, a line starting with `.`); that is how this codebase writes
  chains.

### Rules intentionally disabled (and why — do not "helpfully" enable them)

The `.editorconfig` disables a small set of ktlint rules. Each one is
disabled for a reason that is visible in the code:

- **`function-expression-body`** — would force functions with a
  single-expression body to be written as expression bodies
  (`fun f() = ...`) instead of block bodies (`fun f() { return ... }`).
  The house style is explicit: block bodies with a `return`.
- **`class-naming`** / **`function-naming`** — protocol-version types and
  functions are named after the protocol version they implement (e.g.
  `SerializedBookmark20210828`, `OPDS1.2...`). These are deliberate.
- **`property-naming`** — error-code objects use camelCase constants by
  design (e.g. `BorrowErrorCodes`), and some JSON/XML field names are kept
  as snake_case to mirror the wire format.
- **`package-name`** — legacy packages predate the current naming scheme.

If a disabled rule is genuinely in the way for a new file, stop and ask.

## House code style: verbose and explicit

The house style is **verbose and explicit**, never terse. Where a choice is
between a compact form and an explicit form, the explicit form wins. ktlint
enforces the rules it can check automatically; the following are conventions
that no linter rule checks — they are enforced by review:

1. **No elvis operator (`?:`) in new code.** The idiomatic Kotlin
   `val x = f() ?: default` / `f() ?: return` is banned. Write the null
   check as a block:
   ```kotlin
   val x = f()
   if (x == null) {
     return
   }
   ```
   Rationale: `?:` fuses a value computation and a control-flow decision
   into one expression, which is exactly the compactness the house style
   rejects.

2. **No `?: return` / `?: return null` early-outs.** Use the block form
   from rule 1 instead.

3. **No one-line if-else expressions** (the "ternary" form
   `val x = if (c) a else b`). Use a block:
   ```kotlin
   val x
   if (c) {
     x = a
   } else {
     x = b
   }
   ```
   or `return` from each branch where the structure allows it.

4. **No braceless single-statement `if`s.** `if (x) return` is banned;
   `if (x) { return }` is required. ktlint's `if-else-bracing` and
   `when-entry-bracing` rules check part of this automatically; in addition,
   keep `when` branches braced even where ktlint would not require it.

5. **Function signatures.** 3 or more parameters: one parameter per line.
   1 or 2 parameters: a single line. Do not hand-wrap a 1- or 2-parameter
   signature across multiple lines; `ktlintFormat` will collapse it back
   onto one line.

6. **No star imports** (the `.editorconfig` sets the star-import threshold
   to a value that disables them).

7. **`max_line_length = 120`.** If a line cannot be wrapped by the
   formatter (a long import, a long string literal, a long call argument),
   extract
   a local `val`, qualify a shorter import, or split the string — do not
   disable the limit locally and do not leave the line long.

8. **Trailing commas** at call sites and declaration sites are part of the
   enforced style (including in `when` condition lists).

9. **Indentation:** 2 spaces in `.kt` files, 4 spaces in `.kts` files.

10. **No extension functions.** `fun String.trimTrailing()` and
    `fun Collection<T>.firstNonNull()` are banned, whether top-level or
    declared inside a class. Write an ordinary function that takes the
    would-be receiver as a named first parameter instead:
    ```kotlin
    fun trimTrailing(receiver: String): String
    ```
    (Extension properties — `val String.lastChar` — are banned by the same
    rule; use a named function or an ordinary property.)
    Rationale: an extension function is resolved by import rather than by
    the receiver's type, so the receiver argument is invisible in the call
    site's parameter list and the resolution is determined by which file
    imports it — the hidden coupling the house style rejects. ktlint has
    no standard rule that detects this; it is enforced by review.

11. **No top-level functions.** A `fun` declared at file scope is banned.
    Put the function in a class as a member function.

12. **One class per file.** At most one class at file scope (the same
    for `object`, `interface`, and `enum class` declarations). Nested
    classes are fine — they are part of the enclosing declaration. ktlint
    has no rule that detects a second top-level class in a file (its
    `filename` rule only checks the file's name), so this is enforced by
    review.

## API and dependency preference: JVM over Android

Prefer standard JVM libraries and well-tested, mature, well-maintained
third-party Java libraries over their Android-specific equivalents:

- **Avoid Android platform APIs.** Unless Google absolutely mandates the
  use of a particular API, avoid it. Android APIs cannot be trusted to
  work as documented, to stay working over time, or not to be
  unceremoniously deprecated, forcing rewrites of the parts of the
  application that use them. Prefer standard JVM libraries.
- **Avoid Android-specific third-party libraries.** Use well-tested,
  mature, well-maintained third-party Java libraries instead; the
  Android-specific ones get abandoned after a year.

## Project layout facts

- The repository is a composite build: the root project plus the
  `palace-*` modules, plus the `palace-build-conventions` **git submodule**
  (the build-logic: `PalaceKtLint.kt`, `PalaceProjectProperties.kt`, and
  the `org.thepalaceproject.ktlint` plugin). A `git checkout` in the
  parent repository does **not** touch the submodule's working tree;
  changes inside it must be committed inside the submodule.
- Build parameters live in the root `gradle.properties` under
  `org.thepalaceproject.build.*` (SDK levels, JDK build/bytecode targets,
  `enableKtLint`, …) and in the `GROUP` / `POM_*` properties.
- The root `build.gradle.kts` hard-fails if the Gradle version does not
  exactly match the pinned version; use the repository's `./gradlew`, not a
  system Gradle.
- `ktlint.jar` at the repository root is a build artifact (downloaded and
  hash-verified by the `ktlintDownload` task). It is not a tool for
  hand-running; use `./gradlew ktlintFormat` / `ktlintCheck`.

## Ground rules

- **The repository's own files are the source of truth, not this skill.**
  When in doubt, read the actual `.editorconfig`, `gradle.properties`, and
  `build.gradle.kts` in the current project. If this skill and the repo
  disagree, the repo wins.
- Make the smallest correct change. Do not reformat unrelated code —
  `ktlintFormat` will apply to the whole tree and a mass reformat bloats
  the diff and hides your change.

## Typical workflow

1. Read the module's `build.gradle.kts` and one existing source file in
   the target package to learn the local shape.
2. Edit, matching the style above.
3. Run `./gradlew clean assemble test > <the log file> 2>&1` (Maven
   projects: `mvn clean verify`), then inspect the log file.
4. Fix the errors the log names; re-run until green. If style violations
   are reported, run `./gradlew ktlintFormat` first — it fixes the
   violations it can fix automatically — then re-run the build. A wrong
   `grep` pattern is fixed by re-grepping the log, never by re-running the
   build.
