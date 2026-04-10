# LLM PenTest Assistant — Burp Suite Extension

> AI-powered web penetration testing automation via Large Language Models.  
> Supports Anthropic Claude, OpenAI GPT, and Ollama (local models).

---

## Features

| Feature | Description |
|---|---|
| **Passive Scanner** | Hooks into Burp's scan pipeline; every proxied request analyzed by LLM in a virtual thread |
| **Active Scanner** | LLM identifies injection points, generates probes, sends them, scores responses |
| **Workbench** | Paste any request/response for on-demand analysis, explanation, or custom questions |
| **Payload Generator** | 26 vuln types, WAF bypass variants, OOB payloads → Burp Intruder-ready |
| **Findings Table** | Sortable/filterable by severity with full detail, raw HTTP, and LLM analysis pane |
| **HTML/MD Reports** | Standalone styled HTML report or Markdown export of all findings |
| **Context Menu** | Right-click any request in Proxy/Repeater → "🤖 LLM PenTest" submenu |
| **Multi-Provider** | Anthropic Claude, OpenAI GPT-4o, Ollama, or any LiteLLM-compatible endpoint |
| **Rate Limiting** | Configurable req/min to avoid burning API quota |
| **Persistent Settings** | All config saved in Burp's project-level preference store |

---

## Architecture

```
src/main/java/com/llmpentest/
├── LLMPenTestExtension.java       ← Burp entry point (implements BurpExtension)
├── LLMHttpHandler.java            ← HTTP interception hook (pass-through by default)
│
├── api/
│   └── LLMClient.java             ← Unified LLM client: Anthropic / OpenAI / Ollama
│
├── model/
│   ├── ExtensionConfig.java       ← All settings, persisted via Burp's PersistedObject
│   └── Finding.java               ← Vulnerability finding POJO
│
├── scanner/
│   ├── LLMScannerCheck.java       ← Burp ScanCheck: passive analysis hook
│   └── ActiveScanOrchestrator.java← LLM-driven active scanning pipeline
│
├── report/
│   └── ReportExporter.java        ← HTML and Markdown report generation
│
└── ui/
    ├── MainPanel.java             ← Burp tab host
    ├── DashboardPanel.java        ← Stats cards + recent findings
    ├── FindingsPanel.java         ← Sortable findings table + detail pane
    ├── WorkbenchPanel.java        ← Manual request/response analysis
    ├── ActiveScanPanel.java       ← Active scan UI with live progress log
    ├── PayloadsPanel.java         ← Payload generation UI
    ├── SettingsPanel.java         ← Provider config, API keys, scan options
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
target/llm-pentest-burp-1.2.0.jar
```

---

### Step 2 — Load into Burp Suite

1. Open **Burp Suite** and start or open a project.
2. Click the **Extensions** tab in the top navigation bar.
3. Under the **Installed** sub-tab, click **Add**.
4. In the dialog that appears:
   - **Extension type**: `Java`
   - **Extension file**: click **Select file…** and navigate to `target/llm-pentest-burp-1.2.0.jar`
5. Click **Next**.
6. Watch the **Output** pane — you should see:
   ```
   ✓ LLM PenTest Assistant v1.2.0 loaded
     ✓ Passive scanner registered (Burp Pro)
     ✓ interactsh registered: <your-domain>.oast.pro
   ```
   On Community Edition the passive scanner line is replaced with a warning — all other features still work.

A new **"LLM PenTest Assistant"** tab appears in Burp's main tab bar.

---

### Step 3 — Configure your LLM provider

Open the **⚙️ Settings** tab inside the extension:

1. Select your **Provider** from the dropdown (Anthropic, OpenAI, Ollama, or Custom).
2. Enter your **API Key** (not required for Ollama).
3. Set the **Model** (defaults are auto-filled per provider).
4. Click **🔍 Test Connection** to verify before scanning.
5. Click **💾 Save Settings**.

**Ollama (local models):**
- Start Ollama locally: `ollama serve`
- Pull a model: `ollama pull llama3` (or `mistral`, `qwen2.5-coder`, etc.)
- The endpoint auto-fills to `http://localhost:11434/api/chat` — no API key needed.
- To use a remote Ollama instance, change the endpoint URL accordingly.

---

## Configuration

Open the **⚙️ Settings** tab:

### LLM Provider

| Provider | Model examples | API Key needed |
|---|---|---|
| Anthropic (Claude) | `claude-opus-4-20250514`, `claude-sonnet-4-6` | Yes — [console.anthropic.com](https://console.anthropic.com) |
| OpenAI (GPT) | `gpt-4o`, `gpt-4-turbo` | Yes — [platform.openai.com](https://platform.openai.com) |
| Ollama (Local) | `llama3`, `mistral`, `qwen2.5-coder` | No |
| Custom / LiteLLM | Any OpenAI-compatible endpoint | Depends |

**Ollama** requires a running local instance: `ollama serve`

### Scan Settings

- **In-scope only** — restricts passive scanning to Burp's target scope (recommended)
- **Skip static resources** — skips .png, .css, .woff etc. (saves API tokens)
- **Rate limit (req/min)** — prevents hitting provider rate limits
- **Min confidence (%)** — filters out low-confidence LLM findings before they appear in the table

Click **🔍 Test Connection** to verify your API key before scanning.

---

## Usage

### Passive Scanning

1. Configure your LLM provider in Settings
2. In the **📊 Dashboard** tab, click **▶ Enable Passive Scan**
3. Browse your target application through Burp's proxy
4. Findings appear in the **🎯 Findings** tab as they're discovered
5. Click any finding row to see full detail, PoC, and raw HTTP

### Active Scanning

1. Right-click a request in Proxy/Repeater → **🤖 LLM PenTest** → **Send to Active Scan**  
   *or* paste a request directly in the **🎯 Active Scan** tab
2. Set the target host, port, and HTTPS toggle
3. Click **🚀 Start Active Scan**
4. Watch the live log as the LLM:
   - Identifies injection points
   - Generates targeted probes
   - Sends mutated requests
   - Scores responses for vulnerability indicators

### Manual Workbench

The **🔬 Workbench** tab lets you:
- Analyze a request/response pair for vulnerabilities
- Get a plain-English explanation of what an endpoint does
- Generate payloads for a specific parameter
- Ask any free-form security question about the traffic

### Payload Generator

The **💣 Payloads** tab covers 26 vulnerability types including:
- SQL Injection (regular, blind, time-based)
- XSS (reflected, stored, DOM)
- SSRF, XXE, SSTI, Command Injection
- JWT attacks, GraphQL, Prototype Pollution
- Log4Shell, Java/PHP deserialization, and more

Tick **WAF bypass** and **OOB payloads** for more comprehensive coverage.  
Use **🎯 Copy for Intruder** to strip headers/labels and get a clean payload list.

### Context Menu

Right-click any request in Proxy, Repeater, or Intruder:

```
🤖 LLM PenTest
  ├── Analyze for Vulnerabilities
  ├── Explain Request/Response
  └── Generate Payloads...
        ├── SQL Injection
        ├── XSS (Reflected)
        ├── SSRF
        └── ... (12 types)
```

---

## Active Scan Pipeline (Technical Detail)

```
HttpRequest
    │
    ▼
[1] LLM: identify injection points
    → JSON array: [{name, type, value, context}]
    │
    ▼
[2] For each injection point:
    LLM: generate N probes
    → JSON array: [{payload, vuln_type, encoding, rationale}]
    │
    ▼
[3] For each probe:
    Apply mutation to request (query/body/header/cookie/json)
    Send via Burp HTTP engine
    │
    ▼
[4] LLM: compare baseline vs mutated response
    → {has_vulnerability, title, severity, confidence, poc, ...}
    │
    ▼
[5] confidence ≥ threshold → emit Finding
```

Encoding types: `raw`, `url` (percent-encoded), `b64` (base64) — chosen per payload.

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
  "remediation": "Use parameterized queries or prepared statements..."
}
```

---

## Privacy & Security Notes

- API keys are stored in **Burp's project-level persistence** (not in plaintext files)
- HTTP request/response content is sent to your configured LLM provider for analysis
- Use a self-hosted model (Ollama) if you cannot send traffic to external APIs
- The system prompt explicitly instructs the LLM to behave as a security analyst — it does not instruct it to perform any actions beyond analysis
- **Only use against systems you have explicit written authorization to test**

---

## Extending

### Adding a new LLM provider

1. Add an entry to `ExtensionConfig.LLMProvider` enum
2. Add a case in `LLMClient.buildRequestBody()` to format the request body
3. Add a case in `LLMClient.buildHttpRequest()` for auth headers
4. Add a case in `LLMClient.extractContent()` to parse the response

### Adding a new UI panel

1. Create `YourPanel extends JPanel` in `com.llmpentest.ui`
2. Register it in `MainPanel.initUI()` with `tabs.addTab("Label", new YourPanel(...))`

---

## License

MIT License. For authorized penetration testing use only.  
The authors are not responsible for unauthorized use.
