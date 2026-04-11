# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Run

```bash
# Build (produces shaded fat JAR, skip tests)
mvn package -DskipTests

# Build + run full test suite (90 tests)
mvn package

# Output JAR
target/llm-pentest-burp-1.2.0.jar
```

Install into Burp Suite: **Extensions → Installed → Add → Java → select the JAR**.

Requirements: Java 17+, Maven 3.8+, Burp Suite 2023.10+.

## Working Conventions

These are the things the user always expects — do them without being asked.

### After every code change
1. **Build first, fix before proceeding**: run `mvn package -DskipTests`. If it fails, fix compilation errors before making any other changes.
2. **Run the full test suite**: run `mvn package` (includes tests). All 90 tests must pass before pushing.
3. **Commit and push**: stage only relevant files (not unrelated modified files), write a clear commit message covering what changed and why, then `git push origin master`.

### When adding features or fixing bugs
- **Read the file before editing it.** Never propose changes to code you haven't seen.
- **Check all three LLM client touch points** when adding a new provider: `buildRequestBody()`, `buildHttpRequest()`, `extractContent()` in `LLMClient.java`, plus `ExtensionConfig.LLMProvider` enum + `getDefaultModel()`, plus `SettingsPanel.onProviderChange()`.
- **Write tests** for any new feature or bug fix. Tests live in `src/test/java/com/llmpentest/` mirroring the main source tree. Use JUnit 5 + Mockito. Mock Burp API interfaces (they are interfaces, Mockito handles them). Use `MockWebServer` (`okhttp3.mockwebserver`) for `LLMClient` HTTP tests — **not** `mockwebserver3`.
- **Audit for loose ends** before pushing: check all files touched by the change for crash bugs, wrong assumptions, operator precedence issues, and index-out-of-bounds on edge cases.

### Montoya API gotchas
- `ConsolidationAction` is in `burp.api.montoya.scanner`, **not** `burp.api.montoya.scanner.consolidation`.
- MockWebServer JAR ships classes under `okhttp3.mockwebserver`, not `mockwebserver3`.
- `ExtensionConfig.setProvider()` must capture the **old** provider before reassigning `this.provider`, otherwise the auto-fill guards compare new-to-new and always trigger.

### Commit hygiene
- Stage specific files by name — never `git add -A` or `git add .`.
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

### Active scan pipeline

`ActiveScanOrchestrator` drives a 5-step LLM pipeline:
1. Identify injection points → JSON array
2. Generate probes per injection point → JSON array
3. Mutate and send requests via Burp HTTP engine
4. LLM scores baseline vs mutated response
5. Emit `Finding` if confidence ≥ threshold

Encoding types: `raw`, `url`, `b64`.

### LLM client

`LLMClient` is the unified API client. Adding a new provider requires changes in three methods:
- `buildRequestBody()` — format the request JSON
- `buildHttpRequest()` — set auth headers
- `extractContent()` — parse the response

Bundled dependencies (OkHttp, Gson) are **shaded and relocated** to `com.llmpentest.shaded.*` to avoid conflicts with other Burp extensions.

### Configuration persistence

`ExtensionConfig` uses Burp's `PersistedObject` (project-level preference store) — not files or system properties. Call `config.save()` after any setter to persist changes.

### OOB integration

`InteractshClient` registers with `oast.pro` (ProjectDiscovery's public interactsh) on extension load via a virtual thread. It polls every 5 seconds and emits `Finding` objects on callback. The configured server can be overridden in Settings → `interactshServer`.

### Key packages

| Package | Purpose |
|---|---|
| `com.llmpentest` | Extension entry point + HTTP handler |
| `com.llmpentest.api` | LLM provider client + exception types |
| `com.llmpentest.model` | `Finding` POJO + `ExtensionConfig` |
| `com.llmpentest.scanner` | Passive and active scan logic |
| `com.llmpentest.checks` | Deterministic checks (headers, sensitive data, session, interactsh) |
| `com.llmpentest.ui` | Swing panels registered as Burp suite tabs |
| `com.llmpentest.report` | HTML and Markdown report generation |
| `com.llmpentest.util` | CVSS calculator, finding deduplicator |

### UI panels

All panels extend `JPanel` and are registered in `MainPanel.initUI()`. They receive `LLMScannerCheck` as the shared findings store and register `Consumer<Finding>` listeners for live updates.

### Finding lifecycle

`Finding` objects have a `Source` enum (`PASSIVE_SCAN`, `ACTIVE_SCAN`, `WORKBENCH`, `MANUAL`). `FindingDeduplicator` in `util/` prevents duplicate findings from filling the table. `CvssCalculator` in `util/` computes CVSS scores from severity/type metadata.
