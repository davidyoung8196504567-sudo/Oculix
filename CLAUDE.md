# CLAUDE.md

Guidance for Claude Code (and other AI assistants) working in this repository.

## What this project is

**OculiX** — a cross-platform visual GUI automation platform for the JVM. It drives
any graphical interface by *what it looks like* (OpenCV template matching + OCR),
not by DOM selectors or accessibility APIs.

It is the active continuation of **Sikuli** (Tom Yeh / Tsung-Hsiang Chang, MIT UIST
2009) → **SikuliX1** (Raimund Hocke, 2010–2026, archived March 2026). The
`org.sikuli.*` package namespace and much of the historical prose are deliberately
preserved out of respect for the lineage — **do not "modernize" package names**.

- Group ID: `io.github.oculix-org`
- Current reactor version: **4.0.0** (all modules share one version)
- License: MIT
- Upstream: `oculix-org/Oculix` (this clone's `origin` may be a fork)

## Repository layout

```
pom.xml                  Aggregator POM (packaging=pom). NOT a parent — see "No parent inheritance".
.mvn/maven.config        Global mvn flags (parallel build, log styling, extension classpath)
.mvn/jvm.config          -Xmx3g, UTF-8 encodings
.mvn/extensions/         Pre-built oculix-build-extensions.jar (checked in, loaded via maven.config)
.githooks/               prepare-commit-msg — auto Co-authored-by trailers
build-extensions/        Maven core extension: build banner + reactor-wide dependencyManagement
API/                     Core engine (384 java files) — artifact `oculixapi`
IDE/                     Swing IDE + scripting runtime (148 files) — artifact `oculixide`
MCP/                     Model Context Protocol server (45 files) — artifact `oculixmcp`
Reporter/                Self-contained HTML test reporter (23 files) — artifact `oculixreporter`
Support/                 Parked/dead code kept for reference — NOT a Maven module
Additional-Wrappers/     Design docs (CDCs) for Python / Node / .NET wrappers
docs/                    Product-level design docs
scripts/                 Python tooling (i18n staging → live bundle merge)
translation/             i18n *staging* bundles (auto-translated, pre-native-review)
tmp_workflows/           Scratch copies of CI YAML — not active
test-cli.sikuli/         Manual CLI smoke script
```

`Support/` and `tmp_workflows/` are **not** part of the build. `Support/ParkedForFinalDeletionOrReused/`
holds pre-4.0 copies of classes that moved to IDE — never edit or reference them.

## Modules

### API — `io.github.oculix-org:oculixapi`

The visual automation engine. Since 4.0.0 it is **a library only**: the scripting
runtime (Jython/JRuby/Runner/launcher) was moved out into IDE, and the ~2200-line
`RunTime` god-object was dismantled. Nothing on `oculixapi`'s classpath can boot an IDE.

Key packages:

| Package | Contents |
|---|---|
| `org.sikuli.script` | Public API surface: `Screen`, `Region` (5k lines), `Pattern`, `Match`, `Image`, `Finder`, `App`, `Location`, `Key`, `OCR`, `TextRecognizer`, `FindFailed` |
| `org.sikuli.support` | `Commons` (2.9k lines — platform/native/paths funnel), `FileManager`, `Observer`/`Observing`, `AppLauncher`, `RemoteMode`, `TesseractLastSeen`, `ActionLogRenderer` |
| `org.sikuli.support.devices` | `IScreen`, `IRobot`, `ScreenDevice`, `MouseDevice`, `RobotDesktop`, `KeyboardLayout` |
| `org.sikuli.support.runner` | Only the SPI now: `IRunner`, `AbstractRunner`, `ProcessRunner` |
| `org.sikuli.support.recorder` | Recording engine + code generators |
| `org.sikuli.basics` | `Settings`, `Debug`, `PreferencesUser`, `OS`, hotkey managers |
| `org.sikuli.vnc` | `VNCScreen`, `VNCRobot`, `XKeySym` (full X keysym map) |
| `org.sikuli.android` | `ADBClient`, `ADBScreen`, `ADBRobot`, `ADBDevice` |
| `com.sikulix.ocr` | `OCREngine` + `TesseractEngine` / `PaddleOCREngine` / `PaddleOCRClient` |
| `com.sikulix.util` | `SSHTunnel`, `SikuliLogger`, `TextNormalizer` |

**Vendored third-party sources** live under API and must be treated as read-only
unless you are deliberately patching them: `com.jcraft.jsch` / `com.jcraft.jzlib`
(JSch, for `SSHTunnel`), `se.vidstige.jadb` (ADB client), `com.tulskiy.keymaster`
and `jxgrabkey` (global hotkeys). Never reformat them.

Native OpenCV comes from **Apertix** (`apertix.opencv.version`, dual Linux builds:
modern glibc ≥ 2.38 and `manylinux_2_28` legacy) and Tesseract from **Legerix**
(`legerix.version`) — both Maven artifacts, no manual native install.

The RFB/VNC protocol implementation is **not** in this repo: it comes from the
`io.github.oculix-org:tigervnc-java-oculix` artifact, which is **GPLv2** and is kept
as a separate artifact precisely so the MIT codebase stays MIT. It supplies
`com.tigervnc.*` and `com.sikulix.vnc.*`, which `org.sikuli.vnc.VNCScreen` wraps.
Do not vendor GPL code into this tree. (`tigervnc-java-oculix.bundle` at the repo root
is a git bundle snapshot of that project, not a build input.)

### IDE — `io.github.oculix-org:oculixide`

Swing IDE (FlatLaf-based) *plus* the whole scripting runtime. Main class:
`org.sikuli.ide.Sikulix`.

- `org.sikuli.ide` — `SikulixIDE` (4.4k lines, the app shell), `EditorPane`,
  `EditorConsolePane`, `PatternWindow`, `ButtonCapture`, `CloseableTabbedPane`
- `org.sikuli.ide.ui` — `OculixSidebar` (incl. `LanguagePicker`), `ScriptExplorer`,
  `WelcomeTab`, `WorkspaceDialog`, `SidebarSubmenu`, `recorder/*` (Modern Recorder)
- `org.sikuli.ide.theme` — `OculixColors`, `OculixFonts`, `OculixDarkLaf`, `OculixLightLaf`
- `org.sikuli.support.runner` — the actual runners (`JythonRunner`, `JRubyRunner`,
  `PythonRunner`, `PowershellRunner`, `RobotRunner`, `JarRunner`, `ZipRunner`, …)
- `org.sikuli.support.runnerSupport` — `JythonSupport`, `JRubySupport`
- `org.sikuli.support.ide` — IDE support SPI, autocomplete, syntax highlighting (Jygments port), `SikuliIDEI18N`
- `org.sikuli.idesupport` — CLI arg handling (`CommandArgs`, `CommandArgsEnum`), desktop/taskbar integration

### MCP — `io.github.oculix-org:oculixmcp`

MCP server exposing OculiX as tools for LLM agents. Main class `org.sikuli.mcp.cli.Main`.
Subcommands: `run` (stdio), `serve` (Streamable HTTP), `rotate-key`,
`rotate-session-key`, `recover`, `verify`.

Its differentiator is the **auditable journal**: every tool call is written to an
append-only JSONL entry that is SHA-256 chained (`prev_hash`/`entry_hash`) and
Ed25519 signed. Packages: `server/` (dispatcher, sessions), `tools/` (16 `*Tool`
classes behind `ToolRegistry`), `audit/`, `crypto/` (`CanonicalJson`, `KeyManager`, `Hashing`),
`transport/` (`HttpTransport`, `BearerAuth`, `TokenIssuer`, `TlsPolicy`, `KeyRing`),
`gate/` (`ActionGate`).

Invariants to respect when touching MCP:
- The server **never falls back to unsigned mode**. A missing/unreadable private key
  is a fail-fast refusal, not a degraded start.
- `CanonicalJson` output is part of the hash contract — changing serialization
  breaks verification of existing journals.
- `open` vs `confidential` mode gates which tools are registered
  (`ReadTextInRegionTool` is open-only; `*ToDiskTool` are confidential-only).

Read `MCP/README.md` before changing anything here — it is current and accurate.

### Reporter — `io.github.oculix-org:oculixreporter`

Opt-in single-file HTML test report (base64 screenshots, history, flaky detection).
Entry point `ReportedScreen` wraps `Screen`; listeners for JUnit 5, TestNG and
Selenium. `oculixapi` is `provided`; Selenium/TestNG are `optional`. Targets Java 11
(everything else targets 17).

### build-extensions — `oculix-build-extensions`

A Maven **core extension** (not a plugin), loaded via
`-Dmaven.ext.class.path=.mvn/extensions/oculix-build-extensions.jar` in `.mvn/maven.config`
(there is deliberately no `.mvn/extensions.xml`). Two participants:

- `OculixBuildBanner` — cosmetic gecko banner on every `mvn` run.
- `DependencyManagementInjector` — injects shared `<dependencyManagement>` pins into
  every reactor project at model-load time. **This is where transitive CVE pins live.**
  Add a pin there, not in four child POMs.

## Build and test

Requires **JDK 17+** (compiler `<release>17</release>` for API/IDE/MCP; Reporter is 11)
and Maven 3.8+. `.java-version` says 11 and README/CONTRIBUTING say "Java 11+" — that
is stale for 4.0.0; the build itself will not run on 11.

```bash
# Full reactor, no tests (first build pulls Apertix/Legerix, 2-5 min)
mvn clean install -DskipTests

# Compile a single module (with its deps)
mvn -pl API compile
mvn -pl IDE -am compile

# Tests for one module
mvn -pl API test
mvn -pl MCP test
mvn -pl API test -Dtest=MatchUtilsTest

# Platform fat JARs (assembly descriptors makeapi-*.xml / makeide-*.xml)
mvn -pl API -P complete-win-jar package -DskipTests    # or complete-mac-jar / complete-lux-jar
mvn -pl IDE -P complete-lux-jar package -DskipTests
mvn -pl MCP -am -P mcp-fatjar package -DskipTests

# Run the IDE from a built fat jar
java -jar IDE/target/oculixide-4.0.0-<os>.jar          # <os> = windows | macos | linux
```

`.mvn/maven.config` already applies `-T 2C`, `--color=always`, `--no-transfer-progress`
and the extension classpath, so you do not need to pass them.

### Testing reality

Test coverage is thin and deliberately unit-scoped — 25 test classes total
(12 MCP, 6 IDE, 6 API, 1 Reporter) plus one bench. JUnit 5 (`junit-jupiter`), surefire 3.5.5.

- Most API/IDE behaviour is **screen-dependent and cannot be unit tested**. Headless
  CI only compiles API and IDE; it does not run their tests.
- `API/src/test/.../MousePreflightSmokeTest` needs a display — CI starts Xvfb
  (`DISPLAY=:99`) on Linux before running it.
- `API/src/test/java/bench/OcrPerfBench.java` is a benchmark, not a test; it runs in a
  dedicated Windows workflow.
- **If you touch the IDE, launch it and exercise the change.** This is an explicit
  project rule (CONTRIBUTING + PR checklist), not a nicety — compile-green IDE PRs
  that crash on open have shipped before.

## Conventions

### Code

- **Package namespace is `org.sikuli.*`** (Reporter uses `org.oculix.report.*`,
  build-extensions `io.github.oculix.build`). Keep it. Historical prose and comments
  referring to SikuliX are intentional lineage, not stale text to clean up.
- File header on every Java file:
  ```java
  /*
   * Copyright (c) 2010-2026, sikuli.org, sikulix.com, oculix-org - MIT license
   */
  ```
  Older files carry an earlier year range and the pre-OculiX form — leave them alone
  unless you are substantially rewriting the file.
- Javadoc on new/rewritten classes carries author tags, and AI pair-programming is
  credited explicitly (~115 files do this):
  ```java
  * @author Julien Mer (julienmerconsulting)
  * @author Claude (Anthropic)
  * @since 3.0.3
  ```
- Logging goes through `org.sikuli.basics.Debug` (`Debug.log(level, fmt, args)`,
  `Debug.info`, `Debug.error`, `Debug.action`) — printf-style format strings, not
  concatenation. Early-startup logging uses `Commons.startLog(...)`. Do not add
  `System.out.println`.
- **Native loading has a single funnel: `Commons.loadOpenCV()`.** Never add a bare
  `System.loadLibrary` call. The Linux path is tiered (modern vs `manylinux_2_28`
  glibc variants) and deliberately bypasses Apertix's `loadLocally()`.
- Script runners register through the `IRunner` / `AbstractRunner` pattern plus
  `META-INF/services` SPI files. Follow `JythonRunner` / `PowershellRunner`.
- IDE theming: implement `org.sikuli.ide.ThemeAware` for anything holding
  LaF-sensitive state. `SikulixIDE` dispatches `beforeThemeChange()` → `FlatLaf.updateUI()`
  → `afterThemeChange()`. Do not rely on `FlatLaf.updateUI()` alone. Brand colors come
  from `OculixColors` (`OX_<HUE>_<TONE>`), fonts from `OculixFonts` (Inter / JetBrains
  Mono / Fraunces, fail-soft to system fonts).
- **UI strings must be ASCII / SVG icons — no emoji, no box-drawing, no arrow glyphs.**
  Issue #432 was a systemic sweep removing every surrogate-pair emoji (`🦎`, `▶`),
  U+2500 box-drawing and U+2502 pipes from menus, console, sidebar, status bar and
  dialogs because they render as tofu on some platforms. Use the SVG assets under
  `IDE/src/main/resources/icons/menu/` (rendered via `flatlaf-extras`) or plain ASCII.
  This rule applies to IDE UI text only — Markdown docs still use emoji freely.
- Public API stability in `API/` is taken seriously: no breaking change without a
  deprecation path, even across major versions.

### i18n

22 locale bundles live in `IDE/src/main/resources/i18n/IDE_<locale>.properties`
(`IDE_en_US` is the fallback). Lookup is `SikuliIDEI18N._I(key, args...)`, which runs
values through `MessageFormat` when args are present.

The workflow is two-stage:

1. `translation/IDE_<locale>.properties` — auto-translated **staging**, awaiting native review.
2. `scripts/merge-staging-to-live.py` promotes staging keys into the live bundle.
   It only *adds* keys and never overwrites existing (hand-translated) values, restores
   placeholder sentinels to the exact `IDE_en_US` `MessageFormat` form, doubles bare
   apostrophes on placeholder-carrying values, and re-encodes to `\uXXXX` ASCII.

If you edit a live bundle by hand: escape non-ASCII as `\uXXXX`, and remember that a
lone `'` in a value with placeholders swallows the `{0}`. `IDE/src/main/resources/i18n/IDE/*.po`
are historical gettext sources, no longer the source of truth.

### Git and PRs

- Work branches use prefixes: `fix/`, `feat/`, `docs/`, `refactor/`, `ci/`. Branch from
  `master`. Release branches are `release/oculix` (stable) and `release/oculix-rc` (RC).
- **Commit subject style — the docs and the history disagree.** `CONTRIBUTING.md` and
  the PR checklist say "no `feat:`/`fix:`/`chore:` prefixes". The actual recent history
  is ~80% scoped conventional commits (`fix(ide/menu): …`, `feat(support): …`,
  `i18n(de): …`, `chore(deps): …`). Match the surrounding history when committing here;
  don't "fix" either convention as a drive-by.
- Reference the issue number in the subject scope or body (`(#432)`, `Closes #NNN`).
- Commit hook: run once per clone —
  ```bash
  git config core.hooksPath .githooks
  git config oculix.coauthor "Your Name <you@example.com>"
  ```
  `.githooks/prepare-commit-msg` appends `Co-authored-by:` in both directions — the
  human when Claude is the author, Claude when the human is. Nearly every recent
  commit carries such a trailer; it is the norm here, not an exception.
- PR bodies must fill `.github/PULL_REQUEST_TEMPLATE.md`: linked issue, root cause,
  what changed, how tested (OS + Java + steps), regressions considered, screenshots
  for visual changes.
- One PR, one concern. No drive-by renames, reformatting or unrelated cleanups — this
  is enforced in review.
- `CODEOWNERS`: `@julienmerconsulting` + `@adriancostin6` repo-wide; `/MCP/` is
  `@julienmerconsulting` only.

## No parent inheritance (important)

The root `pom.xml` aggregates modules but **only `build-extensions` declares it as a
`<parent>`**. API, IDE, MCP and Reporter are standalone POMs with their own
`groupId`/`version`/`developers`/`scm` blocks so each can be published to Maven Central
independently. Consequences:

- Version bumps must be applied to **five** POMs (root + four modules).
- Shared dependency pins do **not** flow through inheritance — they are injected by
  `DependencyManagementInjector` in `build-extensions`. Put reactor-wide CVE pins
  there. (The root `pom.xml` also has a `<dependencyManagement>` block, which only
  affects `build-extensions` itself and the aggregator.)
- `build-extensions` is excluded from Central publishing via both `maven.deploy.skip`
  and the central plugin's `skipPublishing` in its `release` profile.

## CI / release

`.github/workflows/`:

| Workflow | Trigger | What it does |
|---|---|---|
| `api-compile.yml` | PR touching `API/**` or root pom | `mvn -pl API compile` on JDK 17 |
| `ide-compile.yml` | PR touching `API/**`/`IDE/**` | `mvn -pl IDE -am compile` |
| `codeql.yml` | push/PR to master, weekly Mon 04:17 UTC | CodeQL Java, builds with `-Pcomplete-win-jar` |
| `test-mouse-preflight.yml` | branch/PR + dispatch | `MousePreflightSmokeTest` on ubuntu/windows/macos, Xvfb on Linux |
| `ocr-perf-bench.yml` | push/PR touching the OCR pipeline | `OcrPerfBench` on windows-latest, uploads output |
| `release.yml` | push to `release/oculix` or dispatch | Builds API+IDE+MCP fat jars for win/mac/linux, publishes GitHub Release (marks it `latest`) |
| `release-rc.yml` | push to `release/oculix-rc` or dispatch | Same, but pre-release; guards that the version has an `-rc`/`-beta`/`-alpha` suffix |
| `plublish-maven.yml` | dispatch only | `mvn deploy -P release` to Maven Central (GPG signed) — filename typo is intentional/known |
| `project-status-sync.yml`, `strip-status-on-close.yml` | issue events | Sync `status:*` labels ↔ Project v2 board |

**Release notes come from `CHANGELOG.md`**: each `## [vX.Y.Z]` section is consumed
verbatim by the release workflows and published as the GitHub Release body. Write those
sections for end users, not for maintainers.

Dependabot runs monthly per module (`/API`, `/IDE`, `/MCP`), grouped, and pins
`org.jruby:jruby-complete` to the 9.4 line (10.x needs Java 21).

## Traps and gotchas

- **`.gitignore` swallows `*.txt` and `Test*.java` globally.** Whitelists exist only
  for `IDE/src/main/resources/*.txt` and `Reporter/src/main/java/**/Test*.java`. A new
  `.txt` resource anywhere else (e.g. under `IDE/src/main/resources/org/sikuli/...`)
  will be silently untracked — `git add -f` it and check `git status` after adding
  resources. Name new test classes `*Test.java`, never `Test*.java`.
- `.claude/` is gitignored; this `CLAUDE.md` at the repo root is not.
- **Legerix must stay the LAST dependency in `API/pom.xml`.** Assembly unpacks in
  declaration order, and Legerix's `tessdata` (eng/fra/spa/chi_sim/hin) + native libs
  must win over Tess4J's (eng + osd only). There is a comment saying so; it is load-bearing.
- Stale docs — do not treat these as current: `API/README.md` and `IDE/README.md` are
  SikuliX 2.1.0-era text (they claim VNC/Android are suspended; both work),
  `RELEASE_NOTES.md` stops at 3.0.2, the root `README.md` badges and Maven snippet still
  say 3.0.3, and `SECURITY.md`'s support table still lists 3.0.x. `CHANGELOG.md`,
  `CONTRIBUTING.md` and `MCP/README.md` are current.
- OCR: Tesseract natives ship for Windows via Tess4J and via Legerix for the platform
  fat jars; on Linux/macOS the MCP jar expects a system Tesseract. PaddleOCR is an
  opt-in HTTP microservice on `127.0.0.1:5000` with transparent Tesseract fallback.
- `TesseractLastSeen` caches OCR results per text per resolution on disk with atomic
  writes — if you change OCR result shapes, consider cache invalidation.
- `VNCScreen` tracks live sessions in a `static Map<String, VNCScreen>` backed by a
  plain `HashMap` (not concurrent). Read `VNCScreen.start*`/`stopAll` before changing
  anything about parallel sessions.
- Two duplicate `IIDESupport` SPI files exist (`org.sikuli.idesupport.IIDESupport` and
  `org.sikuli.support.ide.IIDESupport`) from the 4.0.0 package move. Keep both in sync
  if you add a runner-backed IDE support class.

## Working style expected here

The project has an explicitly high bar for AI-assisted contributions, spelled out in
`CONTRIBUTING.md`:

- Read the surrounding code first — 15 years of SikuliX conventions, some weird, some
  load-bearing.
- Reproduce a bug before fixing it; a fix you can't reproduce is a guess.
- Never silence a symptom (catching an NPE and returning `null` is not a fix) — find
  and state the root cause.
- No new runtime dependency, and no CI/build/release change, without prior agreement
  in an issue.
- Everything submitted must be understood and explainable line by line.
