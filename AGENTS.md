# AGENTS.md

This repo builds a **Burp Suite extension** (`llm-pentest-burp`) that uses LLMs to find and report web vulnerabilities.

## Quick reference

```bash
# Build (fat JAR, skip tests)
mvn package -DskipTests

# Build + all tests (112 tests)
mvn package

# Output JAR (version changes — check pom.xml)
target/llm-pentest-burp-*.jar
```

Install: **Burp Suite → Extensions → Installed → Add → Java → select the JAR**.

Requirements: Java 17+, Maven 3.8+, Burp Suite 2023.10+.

## Must-do after every change

1. `mvn package -DskipTests` — fix compile errors first
2. `mvn package` — all 112 tests must pass
3. Commit with `Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>` and `git push origin master`

## Repo quirks

- **MontoyaApi is an interface** — Mockito mocks it directly, no need for adapter patterns
- **Shaded deps** — OkHttp and Gson are relocated to `com.llmpentest.shaded.*` in the JAR
- **MockWebServer** — use `okhttp3.mockwebserver`, not `mockwebserver3`
- **Config persistence** — `ExtensionConfig` uses Burp's `PersistedObject`. Call `config.save()` after setters.
- **Findings flow** — `LLMScannerCheck.addFinding()` broadcasts to all registered `Consumer<Finding>` listeners

## Adding a new LLM provider

Three methods in `LLMClient.java` need changes:
1. `buildRequestBody()` — format request JSON
2. `buildHttpRequest()` — set auth headers
3. `extractContent()` — parse response

Also update: `ExtensionConfig.LLMProvider` enum + `getDefaultModel()` + `SettingsPanel.onProviderChange()`.

## Montoya API gotchas

- `ConsolidationAction` is in `burp.api.montoya.scanner`, **not** `burp.api.montoya.scanner.consolidation`
- `HttpRequestResponse.service()` does **not exist** — use `api.http().sendRequest(HttpRequest)` directly
- Proxy history: `api.proxy().history()` returns items; use `.toString()` for raw request data

## Version management

Update in 3 places after bumping: `pom.xml`, `LLMPenTestExtension.VERSION`, `MainPanel` header label.

See [CLAUDE.md](./CLAUDE.md) for full architecture documentation.
