# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Run

```bash
# Build (produces shaded fat JAR, skip tests)
mvn package -DskipTests

# Build + run full test suite (112 tests)
mvn package

# Output JAR  (name reflects current version in pom.xml)
target/llm-pentest-burp-<version>.jar
```

Install into Burp Suite: **Extensions → Installed → Add → Java → select the JAR**.

Requirements: Java 17+, Maven 3.8+, Burp Suite 2023.10+.

## Working Conventions

These are the things the user always expects — do them without being asked.

### After every code change
1. **Build first, fix before proceeding**: run `mvn package -DskipTests`. If it fails, fix compilation errors before making any other changes.
2. **Run the full test suite**: run `mvn package` (includes tests). All tests must pass before pushing.
3. **Update the backlog**: edit `backlog.md` to move completed items into `#complete` and remove them from `#to do` or `#bugs`. Do this before committing.
4. **Commit and push**: stage only relevant files (by name, always include `backlog.md` when it changed), write a clear commit message covering what changed and why, then `git push origin master`.

### Backlog (`backlog.md`)
`backlog.md` is the single source of truth for work in flight. Keep it accurate at all times:
- **When starting work on an item**: leave it in `#to do` / `#bugs` until it is fully done.
- **When an item is complete**: move it verbatim to `#complete` in the same commit that delivers the code.
- **When discovering a new bug or idea mid-task**: add it to `#bugs` or `#to do` before the commit.
- **Never leave stale entries**: if something was fixed or is no longer relevant, remove or update it rather than leaving outdated text.
- Always stage and commit `backlog.md` together with the code change it reflects — never in a separate cleanup commit.

### Versioning
The project uses **semantic versioning** (`MAJOR.MINOR.PATCH`). Update the version in **all four places** together whenever a feature or fix lands:
1. `pom.xml` — `<version>` element (controls the output JAR filename)
2. `LLMPenTestExtension.java` — `VERSION` constant and the Javadoc comment on the class
3. `MainPanel.java` — the header label already reads `LLMPenTestExtension.VERSION` dynamically; no change needed there
4. `README.md` — any literal `x.y.z` version strings (JAR path, sample output)

Bump `PATCH` for bug fixes, `MINOR` for new features, `MAJOR` for breaking changes.

### GitHub releases
After every version bump, create a GitHub release in the **same session** as the version commit:

```bash
# Build the release JAR first (tests must already be passing)
mvn package -DskipTests

# Create the release with the JAR attached
gh release create v<VERSION> \
  target/llm-pentest-burp-<VERSION>.jar \
  --title "v<VERSION>" \
  --notes "<one-paragraph summary of what changed in this version>"
```

- The tag must be `v<VERSION>` (e.g. `v1.5.0`) — always prefix with `v`.
- Attach the shaded JAR (`target/llm-pentest-burp-<VERSION>.jar`) as the release asset so users can download it directly from GitHub.
- The release notes should be a concise human-readable summary of the changes (not a raw commit log). Match the tone of the commit message but written for end users.
- Do **not** create a release for intermediate commits that don't bump the version.

### When adding features or fixing bugs
- **Read the file before editing it.** Never propose changes to code you haven't seen.
- **Check all three LLM client touch points** when adding a new provider: `buildRequestBody()`, `buildHttpRequest()`, `extractContent()` in `LLMClient.java`, plus `ExtensionConfig.LLMProvider` enum + `getDefaultModel()`, plus `SettingsPanel.onProviderChange()`.
- **Write tests** for any new feature or bug fix. Tests live in `src/test/java/com/llmpentest/` mirroring the main source tree. Use JUnit 5 + Mockito. Mock Burp API interfaces (they are interfaces, Mockito handles them). Use `MockWebServer` (`okhttp3.mockwebserver`) for `LLMClient` HTTP tests — **not** `mockwebserver3`.
- **Keep UI tabs unique** — avoid features that duplicate existing tabs.
- **Audit for loose ends** before pushing: check all files touched by the change for crash bugs, wrong assumptions, operator precedence issues, and index-out-of-bounds on edge cases.

### Montoya API gotchas
- `ConsolidationAction` is in `burp.api.montoya.scanner`, **not** `burp.api.montoya.scanner.consolidation`.
- MockWebServer JAR ships classes under `okhttp3.mockwebserver`, not `mockwebserver3`.
- `ExtensionConfig.setProvider()` must capture the **old** provider before reassigning `this.provider`, otherwise the auto-fill guards compare new-to-new and always trigger.

### Commit hygiene
- Stage specific files by name — never `git add -A` or `git add .` unless the change is truly all-inclusive.
- Include `Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>` in every commit.
- Do not amend existing commits; always create a new one.

## Architecture

This is a **Burp Suite extension** built on the Montoya API. The entry point is `LLMPenTestExtension.java` which wires up all components via `MontoyaApi`.

### Core data flow

1. `LLMHttpHandler` intercepts all HTTP traffic via Burp's HTTP handler
2. `LLMScannerCheck` (implements `ScanCheck`) runs **4 analysis tracks** per response:
   - **Track 1**: LLM vulnerability analysis — async virtual thread, rate-limited
   - **Track 2**: `SecurityHeadersChecker` — sync, deterministic, zero LLM calls (HSTS, CSP, CORS, etc.)
   - **Track 3**: `SensitiveDataScanner` — sync regex scan for API keys, PII, stack traces
   - **Track 4**: `SessionContextAnalyzer` — multi-request sliding window; triggers LLM async for IDOR/business logic
3. Findings flow into `LLMScannerCheck`'s `CopyOnWriteArrayList<Finding>` and are broadcast to registered `Consumer<Finding>` listeners (used by UI panels)

### LLM client

`LLMClient` is the unified API client. Adding a new provider requires changes in three methods:
- `buildRequestBody()` — format the request JSON
- `buildHttpRequest()` — set auth headers
- `extractContent()` — parse the response

`UsageTracker` in `util/` tracks API requests, tokens, and estimated costs per provider.

Bundled dependencies (OkHttp, Gson) are **shaded and relocated** to `com.llmpentest.shaded.*` to avoid conflicts with other Burp extensions.

### Configuration persistence

`ExtensionConfig` uses Burp's `PersistedObject` (project-level preference store) — not files or system properties. Call `config.save()` after any setter to persist changes.

### OOB integration

`InteractshClient` registers with `oast.pro` (ProjectDiscovery's public interactsh) on extension load via a virtual thread. It polls every 5 seconds and emits `Finding` objects on callback. The configured server can be overridden in Settings → `interactshServer`.

### Key packages

| Package | Purpose |
|---|---|
| `com.llmpentest` | Extension entry point + HTTP handler |
| `com.llmpentest.api` | LLM provider client + `UsageTracker` |
| `com.llmpentest.model` | `Finding` POJO + `ExtensionConfig` |
| `com.llmpentest.scanner` | Passive scan logic, `AITestOrchestrator`, `TargetTrafficAnalyzer` |
| `com.llmpentest.checks` | Deterministic checks (headers, sensitive data, session, interactsh) |
| `com.llmpentest.ui` | Swing panels registered as Burp suite tabs |
| `com.llmpentest.util` | `FindingDeduplicator`, `UsageTracker`, `CvssCalculator` |

### UI panels

All panels extend `JPanel` and are registered in `MainPanel.initUI()`. They receive `LLMScannerCheck` as the shared findings store and register `Consumer<Finding>` listeners for live updates.

### Finding lifecycle

`Finding` objects have a `Source` enum (`PASSIVE_SCAN`, `ACTIVE_SCAN`, `MANUAL`, `TRAFFIC_ANALYSIS`). `FindingDeduplicator` in `util/` prevents duplicate findings from filling the table.
