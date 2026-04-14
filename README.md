# LLM PenTest Assistant — Burp Suite Extension

> AI-powered web penetration testing automation via Large Language Models.  
> Supports Anthropic Claude, OpenAI GPT, Google Gemini, Groq, and Ollama (local models).

---

## Features

| Feature | Description |
|---|---|
| **AI Agent** | Split-pane conversational interface: native HTTP editor on the left, multi-turn AI chat on the right. Quick-action buttons, inline test results, and direct Repeater/Findings integration |
| **Passive Scanner** | Hooks into Burp's scan pipeline; every proxied request analyzed by LLM in a virtual thread |
| **Active Scanner** | LLM identifies injection points, generates probes, sends them, scores responses |
| **Traffic Analyzer** | Import target traffic, LLM analyzes for vulnerabilities and proposes new test cases |
| **False Positive Check** | Right-click any Burp finding → AI analysis to confirm if it's a true or false positive |
| **Dashboard** | Vulnerability type breakdown, finding trend chart, duplicate-merged stats, recent findings |
| **Findings Table** | Sortable/filterable by severity with full-text search, bulk status changes, and full detail pane |
| **Payload Generator** | 26 vuln types, WAF bypass variants, OOB payloads → Burp Intruder-ready |
| **HTML/MD Reports** | Standalone styled HTML report or Markdown export of all findings |
| **Context Menu** | Right-click any request in Proxy/Repeater → "🤖 LLM PenTest" submenu |
| **Multi-Provider** | Anthropic Claude, OpenAI GPT-4o, Google Gemini, Groq, Ollama, or any OpenAI-compatible endpoint |
| **Fetch Models** | Pull available models directly from your provider and populate the model dropdown |
| **Rate Limiting** | Configurable req/min with atomic enforcement to avoid burning API quota |
| **Persistent Settings** | All config saved in Burp's project-level preference store |

---

## Architecture

```
src/main/java/com/llmpentest/
├── LLMPenTestExtension.java       ← Burp entry point (implements BurpExtension)
├── LLMHttpHandler.java            ← HTTP interception hook (pass-through by default)
│
├── api/
│   └── LLMClient.java             ← Unified LLM client: Anthropic / OpenAI / Gemini / Groq / Ollama
│
├── model/
│   ├── ExtensionConfig.java       ← All settings, persisted via Burp's PersistedObject
│   └── Finding.java               ← Vulnerability finding POJO
│
├── scanner/
│   ├── LLMScannerCheck.java       ← Burp ScanCheck: 4-track passive analysis hook
│   ├── AITestOrchestrator.java    ← AI Agent test runner with adaptive follow-up tests
│   └── TargetTrafficAnalyzer.java ← LLM traffic analysis + vuln proposal
│
├── checks/
│   ├── SecurityHeadersChecker.java ← Deterministic header rules (HSTS, CSP, CORS, …)
│   ├── SensitiveDataScanner.java   ← Regex-based secrets / PII detection
│   ├── SessionContextAnalyzer.java ← Sliding-window IDOR / logic flaw detection
│   └── InteractshClient.java       ← OOB interaction tracking (interactsh / oast.pro)
│
├── report/
│   └── ReportExporter.java        ← HTML and Markdown report generation
│
└── ui/
    ├── MainPanel.java             ← Burp tab host
    ├── DashboardPanel.java        ← Stats cards, vuln breakdown, trend chart
    ├── FindingsPanel.java         ← Findings table with search and bulk actions
    ├── AITestingPanel.java        ← AI Agent: split HTTP viewer + conversational chat
    ├── WorkbenchPanel.java        ← Manual analysis workbench (paste & analyze)
    ├── TrafficAnalyzerPanel.java  ← Import and analyze target traffic
    ├── ReportPanel.java           ← Report generation UI
    ├── OOBPanel.java              ← Out-of-band interaction viewer
    ├── SettingsPanel.java         ← Provider config, model picker, scan options
    └── ContextMenuProvider.java   ← Right-click menu
```

---

## Build & Install

### Prerequisites

| Tool | Minimum version | Check |
|---|---|---|
| Java JDK | 17 | `java -version` |
| Apache Maven | 3.8 | `mvn -version` |
| Burp Suite | 2023.10 (Pro or Community) | — |

> **Linux / macOS**: Install via `sdkman`, `brew`, or your package manager.  
> **Windows**: Download from [adoptium.net](https://adoptium.net) (JDK) and [maven.apache.org](https://maven.apache.org/download.cgi) (Maven), then add both to `PATH`.

---

### Step 1 — Clone and build

```bash
git clone https://github.com/esandeepchoudary/bur-pai.git
cd bur-pai
mvn package -DskipTests
```

Maven downloads dependencies on first run (OkHttp, Gson, FlatLaf). The shade plugin bundles them with relocated packages so they don't conflict with other Burp extensions.

Successful output ends with:
```
[INFO] BUILD SUCCESS
```

The output JAR is at:
```
target/llm-pentest-burp-1.7.4.jar
```

---

### Step 2 — Load into Burp Suite

1. Open **Burp Suite** and start or open a project.
2. Click the **Extensions** tab in the top navigation bar.
3. Under the **Installed** sub-tab, click **Add**.
4. In the dialog that appears:
   - **Extension type**: `Java`
   - **Extension file**: click **Select file…** and navigate to `target/llm-pentest-burp-1.7.4.jar`
5. Click **Next**.
6. Watch the **Output** pane — you should see:
   ```
   ✓ LLM PenTest Assistant v1.7.0 loaded
     ✓ Passive scanner registered (Burp Pro)
     ✓ interactsh registered: <your-domain>.oast.pro
   ```
   On Community Edition the passive scanner line is replaced with a warning — all other features still work.

A new **"LLM PenTest Assistant"** tab appears in Burp's main tab bar.

---

### Step 3 — Configure your LLM provider

Open the **⚙️ Settings** tab inside the extension:

1. Select your **Provider** from the dropdown.
2. Enter your **API Key** (not required for Ollama).
3. Set the **Model** — click **Fetch Models** to pull available models from your provider and select from the dropdown.
4. Click **🔍 Test Connection** to verify before scanning.
5. Click **💾 Save Settings**.

**Ollama (local models):**
- Start Ollama locally: `ollama serve`
- Pull a model: `ollama pull llama3` (or `mistral`, `qwen2.5-coder`, etc.)
- The endpoint auto-fills to `http://localhost:11434` — no API key needed.
- Click **Fetch Models** to populate the model dropdown from your Ollama instance.

---

## Configuration

Open the **⚙️ Settings** tab:

### LLM Provider

| Provider | Default model | API Key | Key source |
|---|---|---|---|
| Anthropic (Claude) | `claude-opus-4-20250514` | Yes | [console.anthropic.com](https://console.anthropic.com) |
| OpenAI (GPT) | `gpt-4o` | Yes | [platform.openai.com](https://platform.openai.com) |
| Google (Gemini) | `gemini-2.0-flash` | Yes | [aistudio.google.com](https://aistudio.google.com) |
| Groq | `llama-3.3-70b-versatile` | Yes | [console.groq.com](https://console.groq.com) |
| Ollama (Local) | `llama3` | No | — |
| Custom / LiteLLM | _(any OpenAI-compatible)_ | Depends | — |

### Scan Settings

- **In-scope only** — restricts passive scanning to Burp's target scope (recommended)
- **Skip static resources** — skips .png, .css, .woff etc. (saves API tokens)
- **Rate limit (req/min)** — enforced atomically; prevents hitting provider rate limits
- **Min confidence (%)** — filters out low-confidence LLM findings before they appear in the table
- **Concurrency** — parallel test threads used by the AI Agent (1–20)

---

## Usage

### AI Agent

The **🤖 AI Agent** tab provides a split-pane interface for conversational security testing.

**Layout:**
- **Left pane** — Burp's native HTTP message editor, showing the request and response for the currently selected target. Supports all standard editor features (syntax highlighting, search, hex view).
- **Right pane** — Multi-turn AI conversation with context retained across the session.

**Quick-action buttons** (one click, no typing required):

| Button | What it does |
|---|---|
| Explain | Describes the endpoint's purpose, data flow, and visible security mechanisms |
| Find Bugs | Full vulnerability analysis across all common classes |
| Attack Surface | Maps every injection point and ranks them by exploitability |
| Test XSS | Targeted XSS test plan with payload variants and WAF bypass suggestions |
| Test SQLi | SQL injection test plan covering error-based, blind, time-based, and UNION techniques |
| Test Auth | Authentication and authorisation analysis — IDOR, privilege escalation, session flaws |
| Generate PoC | Self-contained proof-of-concept for the most likely vulnerability |
| Run Tests | Launches the full automated test orchestrator across all loaded requests |

**Workflow:**
1. Right-click any request in Proxy / Repeater → **Send to AI Agent**
2. The request and response load into the left editor pane
3. Ask a question in the chat or click a quick-action button
4. Inline result cards appear for each test: status badge, test name, and action buttons
5. Click **Details** for full traffic + follow-up chat, **→ Repeater** to send the PoC, or **+ Findings** to add the vulnerability to the main Findings table

When multiple requests are loaded, use the **< Prev / Next >** navigation in the top bar to switch between them — the AI always has context for the currently displayed request.

### Passive Scanning

1. Configure your LLM provider in Settings.
2. In the **📊 Dashboard** tab, enable passive scanning.
3. Browse your target through Burp's proxy.
4. Findings appear in the **🎯 Findings** tab as they're discovered.

Each proxied response triggers four parallel analysis tracks:

| Track | Engine | LLM calls |
|---|---|---|
| 1 | LLM vulnerability analysis | 1 per unique URL |
| 2 | Security headers checker | 0 (deterministic) |
| 3 | Sensitive data scanner | 0 (regex) |
| 4 | Session context analyzer | 1 per N requests (IDOR / logic flaws) |

### Traffic Analyzer

The **🎯 Traffic Analyzer** tab lets you:
- Import target traffic from sitemap or HAR files
- Let the LLM analyze the full request/response history
- Get a vulnerability report with CWE IDs, severity, confidence, and PoC
- Automatically propose new test cases based on discovered patterns

### Context Menu

Right-click any request in Proxy, Repeater, or Intruder:

```
🤖 LLM PenTest
  ├── Send to AI Agent
  ├── Send to LLM Workbench
  ├── Analyze for Vulnerabilities
  ├── Explain Request/Response
  ├── AI: Targeted Scan on... (sub-menu with detected parameters)
  └── Generate Payloads...
        ├── SQL Injection
        ├── XSS (Reflected)
        └── ... (12 types)
```

Right-click any **Finding** in Burp's Scanner results:

```
🤖 LLM PenTest
  └── AI: Analyze for False Positive
```

---

## Active Scan Pipeline

```
HttpRequest
    │
    ▼
[1] LLM: identify injection points
    → [{name, type, value, context}]
    │
    ▼
[2] For each injection point:
    LLM: generate probes
    → [{payload, vuln_type, encoding, rationale}]
    │
    ▼
[3] For each probe:
    Apply mutation → send via Burp HTTP engine
    │
    ▼
[4] LLM: compare baseline vs mutated response
    → {has_vulnerability, title, severity, confidence, poc, …}
    │
    ▼
[5] confidence ≥ threshold → emit Finding
```

The AI Agent test runner adds **adaptive follow-up tests**: when a response suggests a vulnerability (e.g., a SQL error), the orchestrator automatically generates and queues confirmation probes (e.g., time-based blind SQLi) before producing the final summary.

---

## Example LLM Finding JSON Schema

```json
{
  "has_vulnerability": true,
  "title": "SQL Injection in 'id' parameter",
  "vuln_type": "SQL Injection",
  "cwe": "CWE-89",
  "severity": "Critical",
  "confidence": 92,
  "parameter": "id",
  "evidence": "You have an error in your SQL syntax...",
  "description": "The 'id' parameter in GET /api/users is interpolated directly into a SQL query...",
  "poc": "GET /api/users?id=1' OR '1'='1",
  "remediation": "Use parameterized queries or prepared statements."
}
```

---

## Privacy & Security Notes

- API keys are stored in **Burp's project-level persistence** (not in plaintext files)
- Gemini API keys are transmitted via request header (`x-goog-api-key`), never embedded in URLs, to prevent exposure in proxy logs or server access logs
- HTTP request/response content is sent to your configured LLM provider for analysis — use Ollama for air-gapped or sensitive environments
- The system prompt instructs the LLM to behave as a security analyst; it does not instruct it to perform any autonomous actions beyond analysis
- **Only use against systems you have explicit written authorization to test**

---

## Extending

### Adding a new LLM provider

Update the following in order:

1. `ExtensionConfig.LLMProvider` enum — add entry with `displayName` and `defaultEndpoint`
2. `ExtensionConfig.getDefaultModel()` — add `case YOURPROVIDER -> "model-name"`
3. `LLMClient.buildRequestBody()` — format the single-prompt request JSON
4. `LLMClient.buildChatRequestBody()` — format multi-turn conversation JSON
5. `LLMClient.buildHttpRequest()` — set auth headers (and override URL if needed)
6. `LLMClient.extractContent()` — parse the response JSON
7. `LLMClient.parseAndRecordUsage()` — record token counts
8. `LLMClient.fetchAvailableModels()` — model list URL and auth
9. `LLMClient.parseModelList()` — parse model list response
10. `LLMClient.testConnection()` — add to the existing provider case group
11. `SettingsPanel.onProviderChange()` — set default model in the UI

### Adding a new UI panel

1. Create `YourPanel extends JPanel` in `com.llmpentest.ui`
2. Register it in `MainPanel.initUI()` with `tabs.addTab("Label", new YourPanel(...))`

---

## License

MIT License. For authorized penetration testing use only.  
The authors are not responsible for unauthorized use.
