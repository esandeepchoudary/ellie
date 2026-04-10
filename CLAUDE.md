# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Run

```bash
# Build (produces shaded fat JAR)
mvn package -DskipTests

# Output JAR
target/llm-pentest-burp-1.1.0.jar
```

Install into Burp Suite: **Extensions → Installed → Add → Java → select the JAR**.

Requirements: Java 17+, Maven 3.8+, Burp Suite 2023.10+.

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
