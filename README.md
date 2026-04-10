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

## Build Requirements

- **Java 17+** (`java -version`)
- **Maven 3.8+** (`mvn -version`)
- **Burp Suite Professional or Community Edition** (2023.10+)
- Internet access to download Maven dependencies

---

## Building

```bash
git clone <this-repo>
cd llm-pentest-burp
mvn package -DskipTests
```

The shaded (fat) JAR is output to:
```
target/llm-pentest-burp-1.1.0.jar
```

The Maven shade plugin bundles OkHttp, Gson, and FlatLaf with relocated packages
so they don't conflict with other Burp extensions.

---

## Installation

1. Open Burp Suite
2. Go to **Extensions** → **Installed** → **Add**
3. Extension type: **Java**
4. Select `target/llm-pentest-burp-1.1.0.jar`
5. Click **Next** — you should see "✓ LLM PenTest Assistant v1.1.0 loaded" in the output

A new **"LLM PenTest Assistant"** tab appears in Burp's main tab bar.

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
