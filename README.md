# LLM PenTest Assistant — Burp Suite Extension

> AI-powered web penetration testing automation via Large Language Models.  
> Supports Anthropic Claude, OpenAI GPT, Groq, and Ollama (local models).

---

## Features

| Feature | Description |
|---|---|
| **AI Agent** | Conversational security testing — chat with the AI about requests, get live results, multi-turn adaptive testing, and deep-dive into findings |
| **Passive Scanner** | Hooks into Burp's scan pipeline; every proxied request analyzed by LLM in a virtual thread |
| **Active Scanner** | LLM identifies injection points, generates probes, sends them, scores responses |
| **Traffic Analyzer** | Import target traffic, LLM analyzes for vulnerabilities and proposes new test cases |
| **False Positive Check** | Right-click any Burp finding → AI analysis to confirm if it's a true or false positive |
| **Dashboard** | Vulnerability type breakdown, finding trend chart, duplicate-merged stats, recent findings |
| **Findings Table** | Sortable/filterable by severity with full-text search, bulk status changes, and full detail pane |
| **Payload Generator** | 26 vuln types, WAF bypass variants, OOB payloads → Burp Intruder-ready |
| **HTML/MD Reports** | Standalone styled HTML report or Markdown export of all findings |
| **Context Menu** | Right-click any request in Proxy/Repeater → "🤖 LLM PenTest" submenu |
| **Multi-Provider** | Anthropic Claude, OpenAI GPT-4o, Groq, Ollama, or any OpenAI-compatible endpoint |
| **Fetch Models** | Pull available models directly from your provider and populate the model dropdown |
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
│   └── LLMClient.java             ← Unified LLM client: Anthropic / OpenAI / Ollama / Groq
│
├── model/
│   ├── ExtensionConfig.java       ← All settings, persisted via Burp's PersistedObject
│   └── Finding.java              ← Vulnerability finding POJO
│
├── scanner/
│   ├── LLMScannerCheck.java       ← Burp ScanCheck: passive analysis hook
│   ├── AITestOrchestrator.java    ← AI Agent test runner with chain-of-thought
│   └── TargetTrafficAnalyzer.java ← LLM traffic analysis + vuln proposal
│
├── report/
│   └── ReportExporter.java        ← HTML and Markdown report generation
│
└── ui/
    ├── MainPanel.java             ← Burp tab host
    ├── DashboardPanel.java        ← Stats cards, vuln breakdown, trend chart
    ├── FindingsPanel.java         ← Findings table with search and bulk actions
    ├── AITestingPanel.java        ← AI Agent: natural language testing UI
    ├── TrafficAnalyzerPanel.java  ← Import and analyze target traffic
    ├── PayloadsPanel.java         ← Payload generation UI
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
target/llm-pentest-burp-1.4.0.jar
```

---

### Step 2 — Load into Burp Suite

1. Open **Burp Suite** and start or open a project.
2. Click the **Extensions** tab in the top navigation bar.
3. Under the **Installed** sub-tab, click **Add**.
4. In the dialog that appears:
   - **Extension type**: `Java`
   - **Extension file**: click **Select file…** and navigate to `target/llm-pentest-burp-1.4.0.jar`
5. Click **Next**.
6. Watch the **Output** pane — you should see:
   ```
   ✓ LLM PenTest Assistant v1.4.0 loaded
     ✓ Passive scanner registered (Burp Pro)
     ✓ interactsh registered: <your-domain>.oast.pro
   ```
   On Community Edition the passive scanner line is replaced with a warning — all other features still work.

A new **"LLM PenTest Assistant"** tab appears in Burp's main tab bar.

---

### Step 3 — Configure your LLM provider

Open the **⚙️ Settings** tab inside the extension:

1. Select your **Provider** from the dropdown (Anthropic, OpenAI, Groq, Ollama, or Custom).
2. Enter your **API Key** (not required for Ollama).
3. Set the **Model** — click **Fetch Models** to pull available models from your provider and select from the dropdown.
4. Click **🔍 Test Connection** to verify before scanning.
5. Click **💾 Save Settings**.

**Ollama (local models):**
- Start Ollama locally: `ollama serve`
- Pull a model: `ollama pull llama3` (or `mistral`, `qwen2.5-coder`, etc.)
- The endpoint auto-fills to `http://localhost:11434` — no API key needed.
- Click **Fetch Models** to populate the model dropdown from your Ollama instance.
- To use a remote Ollama instance, change the endpoint URL accordingly.

---

## Configuration

Open the **⚙️ Settings** tab:

### LLM Provider

| Provider | Model examples | API Key needed |
|---|---|---|
| Anthropic (Claude) | `claude-opus-4-20250514`, `claude-sonnet-4-6` | Yes — [console.anthropic.com](https://console.anthropic.com) |
| OpenAI (GPT) | `gpt-4o`, `gpt-4-turbo` | Yes — [platform.openai.com](https://platform.openai.com) |
| Groq | `llama-3.3-70b`, `mixtral-8x7b` | Yes — [console.groq.com](https://console.groq.com) |
| Ollama (Local) | `llama3`, `mistral`, `qwen2.5-coder` | No |
| Custom / LiteLLM | Any OpenAI-compatible endpoint | Depends |

### Scan Settings

- **In-scope only** — restricts passive scanning to Burp's target scope (recommended)
- **Skip static resources** — skips .png, .css, .woff etc. (saves API tokens)
- **Rate limit (req/min)** — prevents hitting provider rate limits
- **Min confidence (%)** — filters out low-confidence LLM findings before they appear in the table

---

## Usage

### AI Agent

The **🤖 AI Agent** tab provides a conversational interface for security testing. Ask natural language questions about your target:

- "What vulnerabilities might exist in this request?"
- "Test for SQL injection by injecting payloads into all parameters"
- "Explain this request/response interaction"
- "Check for information disclosure in error responses"

**Key Features:**
- **Conversational UI**: Maintains session history for follow-up questions.
- **Multi-Turn Adaptive Testing**: The AI analyzes responses and dynamically generates follow-up probes to confirm vulnerabilities.
- **Live Streaming Results**: Watch as the AI plans and executes tests in real-time.
- **Finding Deep-Dive**: Click "Details" on any result to view full traffic and start a sub-chat specifically about that finding.
- **Repeater Integration**: Send any AI-generated PoC directly to Burp Repeater for manual verification.

**Context menu**: Right-click any request → "🤖 LLM PenTest" → "Send to AI Agent".

### Passive Scanning

1. Configure your LLM provider in Settings
2. In the **📊 Dashboard** tab, click **▶ Enable Passive Scan**
3. Browse your target application through Burp's proxy
4. Findings appear in the **🎯 Findings** tab as they're discovered
5. Use the search bar to filter, and bulk-change status (Confirmed / False Positive / Remediated)

### Traffic Analyzer

The **📡 Traffic Analyzer** tab lets you:
- Import target traffic from sitemap or HAR files
- Let the LLM analyze the full request/response history
- Get a vulnerability report with CWE IDs, severity, confidence, and PoC
- Automatically propose new test cases based on discovered patterns

### Context Menu

Right-click any request in Proxy, Repeater, or Intruder:

```
🤖 LLM PenTest
  ├── Send to AI Agent
  ├── Analyze for Vulnerabilities
  ├── Explain Request/Response
  ├── AI: Targeted Scan on... (sub-menu with params)
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

Four methods in `LLMClient.java` need changes:

1. `buildRequestBody()` — format request JSON
2. `buildHttpRequest()` — set auth headers
3. `extractContent()` — parse response
4. `getDefaultModel()` — return default model name

Also update: `ExtensionConfig.LLMProvider` enum + `SettingsPanel.onProviderChange()` + `testConnection()` + `fetchAvailableModels()`.

### Adding a new UI panel

1. Create `YourPanel extends JPanel` in `com.llmpentest.ui`
2. Register it in `MainPanel.initUI()` with `tabs.addTab("Label", new YourPanel(...))`

---

## License

MIT License. For authorized penetration testing use only.  
The authors are not responsible for unauthorized use.
