#complete
- Right-click "Send to AI Agent" now works in Proxy, Repeater, and Target tabs (messageEditorRequestResponse fallback)
- Pause / Resume button for AI Agent test runs
- Drill-down Details dialog now shows request/response in Burp native editors with follow-up chat
- Export test results to CSV / JSON / HTML after a run completes
- Chat in AI Agent tab retains full context of test results (summary added to conversation history)
- UI responsive layout: removed fixed pixel max-heights across Dashboard, OOB, Settings panels; OOBPanel sidebar is now a resizable JSplitPane
- Provider/model in extension header now updates live when settings are saved (was static, showed stale values)
- AI Agent: plan-first execution — test plan shown in chat before any test runs (⏳ → ✅/🔴 live updates)
- AI Agent: sequential execution — tests run in plan order, results appear in sequence
- AI Agent: Details dialog now shows the actual HTTP request sent (not an LLM description); CRLF normalization fixed in request parser
- AI Agent: result cards no longer capped at 60px; observation snippet visible inline
- AI Agent: LLM-generated narrative summary of all test cases and observed outcomes
- AI Agent: WAF detection + auto-bypass — 403/406/429 with WAF fingerprints automatically queues URL-encoded and comment-obfuscated variants
- AI Agent: interactive plan editing — editable plan card with checkboxes and "▶ Execute Plan" button before any tests run
- AI Agent: re-run individual test — "↻" button on each result card re-executes that single test without restarting the full run
- AI Agent: stop-at-first-finding — "Stop on 1st vuln" checkbox halts execution on the first confirmed vulnerability
- AI Agent: export partial results on stop — "Export partial results?" dialog when user stops a run mid-way
- AI Agent: keyboard shortcuts — Ctrl+Enter to send chat, Esc to stop active run
- AI Agent: multi-request correlation — detects shared auth headers, CSRF tokens, and sequential resource IDs across loaded requests; shown in plan card before execution
- AI Agent: auth refresh mid-run — on 401/403 during a non-auth-bypass test, restores original auth headers and retries once automatically
- AI Agent: separate planner/tester agents — Planner Agent prompt (recon-first, 3-phase fingerprinting) and Tester Agent prompt (precise verification, no speculation)
- AI Agent: token/cost budget — pre-run cost estimate shown in plan card, configurable hard cap per session via spinner; run halts when cap reached
- AI Agent: actionable follow-up tests — when the AI's chat reply describes security tests, a "▶ Run These Tests" button appears below the bubble and feeds the reply as instructions into the planner
- AI Agent: fix test-case generation 400 json_validate_failed (second fix) — removed "followUp" from prompt schema (LLM was emitting dangling followUp keys between array elements); added bracket-matching object extractor as extractJsonArray() strategy 5 so malformed wrappers no longer block test extraction
- AI Agent: copy button added to AI chat bubbles (📋 in bottom-right corner of every AI response)
- AI Agent: fix follow-up test parsing crash "JsonObject" — normalise followUpTests field to array even when LLM returns a single object instead of an array
- AI Agent: result cards layout fixed — switched from BorderLayout (buttons in EAST were squeezed off-screen by CENTER text) to BoxLayout Y_AXIS with buttons on a dedicated third row; 🔍 Inspect and → Repeater are now always visible
- AI Agent: fix applyTransformations() — now handles path, method, query, body keys so LLM-generated path traversal, IDOR, and method-change modifications are applied to the actual request
- AI Agent: fix injectIdorPayload() — numeric path segments (/api/resource/123) now replaced when no query param ID is present
- AI Agent: fix injectPathTraversal() — traversal sequence appended to path when no file=/path=/page= param exists
- Workbench tab restored — was missing from MainPanel; "Send to LLM Workbench" context menu now correctly routes to the Workbench panel
- Copy to clipboard buttons added across UI: Findings detail/request/response panes, OOB interaction log, Traffic Analyzer detail pane, Traffic Analyzer AI chat bubbles
- Traffic Analyzer: path normalization — `/api/users/123` → `/api/users/{id}` for dedup and IDOR candidate surfacing
- Traffic Analyzer: parameter extraction — query params, JSON body keys, form fields extracted per endpoint and included in LLM prompt
- Traffic Analyzer: tech stack fingerprinting — Server, X-Powered-By, cookie patterns, framework hints, auth type detected and shown in summary + report
- Traffic Analyzer: error endpoint priority — 5xx/4xx endpoints surfaced first in analysis prompt with response body snippets
- Traffic Analyzer: chunked analysis — large traffic sets analyzed in batches of 35 endpoints; results merged with vuln/proposal deduplication
- Traffic Analyzer: auth mapping — endpoints without auth headers flagged as [NO-AUTH] in prompt to guide bypass testing
- Traffic Analyzer: post-analysis chat — after analysis completes, a full chat panel appears below results; LLM has complete context of all endpoints, vulnerabilities, tech stack, and business logic; 6 quick-question chips for common questions; multi-turn conversation history
- LLM provider integrations hardened: Anthropic temperature now sent; Groq/OpenAI get response_format:json_object, Custom does not; Custom provider no longer requires API key (supports unauthenticated local proxies); Ollama connection test and model fetch use configured endpoint instead of hardcoded localhost:11434; Settings checkboxes (in-scope only, skip static) now load saved state correctly
- Settings: rate limit raised from 100 to 2000 req/min with step-10 increments; concurrency raised from 20 to 100
- Settings: body size cap — configurable KB limit (4–512 KB) on request/response content sent to LLM; applied in buildHttpContextBlock and buildRequestBody
- Settings: HTTP proxy support — route all LLM API calls through a configurable HTTP proxy; applied immediately on Save via OkHttpClient rebuild
- Settings: per-provider model memory — switching provider now restores the last model you chose for that provider
- Settings: auto-test on save — connection test runs automatically in background after Save; result shown inline in status bar
- Settings: live session usage panel — shows request count, token breakdown (in/out), estimated cost, and error count with Refresh and Reset buttons
- Settings: max tokens raised to 32768 (was 8192); system prompt text area uses monospaced font
- Project renamed to ELLIE (Exploit LLM Intelligence Engine) — named after Ellie from The Last of Us; updated extension name, UI header, context menu, README, CLAUDE.md, and pom.xml display name
- Dead code removed: unused autoIntercept field + KEY_AUTO_INTERCEPT constant + getter/setter pair in ExtensionConfig; unused getComponent() in MainPanel; stale "LLM PenTest Assistant" references in ReportExporter HTML/Markdown footers replaced with ELLIE branding
- Public release risk mitigation: artifactId renamed from llm-pentest-burp to ellie (JAR is now ellie-1.8.0.jar, removing PortSwigger trademark exposure); LICENSE file added (MIT + Apache 2.0 attribution); first-run consent dialog explains LLM data forwarding, OOB disclosure, and authorized-use requirement; OOB/interactsh gated behind explicit opt-in toggle (off by default); Apache 2.0 NOTICE transformer added to shade plugin
- AI Agent: fix follow-up test parse crash "JsonObject" — guard non-object array elements with isJsonObject() check; handle top-level JSON array wrapper; tolerate non-primitive modification values; error log now includes exception class name
- AI Agent: fix Inspect dialog follow-up chat — auto-scroll chatHistory after each append so responses are visible; give followInput keyboard focus on open; improve error message (class name fallback when message is null)
- AI Agent: fix "Run These Tests" disconnect — suppress follow-up button when AI reply is an external-tool guide (ffuf, gobuster, nmap, etc.); rename button to "Generate ELLIE Tests from This" with tooltip clarifying ELLIE interprets suggestions into HTTP-level tests
- AI Agent: Burp-aware system prompt — DEFAULT_SYSTEM_PROMPT now explicitly states ELLIE runs inside Burp Suite and can only execute tests as HTTP requests; includes what good vs bad test suggestions look like; PLANNER AGENT prompt now has hard constraints requiring every test case to be a single HTTP request modification with no external tools or OOB
- AI Agent: horizontal scrollbar added to chat pane (was HORIZONTAL_SCROLLBAR_NEVER)
- AI Agent: fixed horizontal overflow — chat panel now implements Scrollable (getScrollableTracksViewportWidth=true) so content fits the viewport width instead of expanding horizontally; removed 30px left indent on result/export cards
- AI Agent: applyTransformations() audit — fixed modifyQueryParam() (was using full URL instead of path; rewrote to parse request line directly); fixed isHeader() removing "token"/"api-key"/"api-token" false positives; fixed form-encoded body injection branch; fixed SQLi/SSTI no-op fallback via injectIntoBodyOrAppend(); 13 new unit tests covering all transformation types
- AI Agent: conversational test requests now route directly to the test plan flow — when a user types a test request ("test for XSS", "find SQL injection", etc.) with a request loaded, ELLIE skips the LLM chat round-trip and goes straight to the plan-first execution flow

#to do

## Passive Scanner
- False positive memory: when a finding is marked FP by the user, record the pattern (URL + vuln type + parameter) and suppress future identical matches from the same endpoint automatically
- Bulk sitemap scan: trigger LLM analysis over all already-proxied in-scope traffic from the Burp sitemap, not just new traffic arriving after the extension loads

## Findings
- Finding notes: free-text annotation field per finding, visible in the detail pane and included in HTML/Markdown exports
- Nuclei template export: auto-generate a Nuclei YAML template from any confirmed finding that has a PoC request
- Issue tracker export: one-click creation of a Jira or GitHub issue from a finding via a configurable webhook URL in Settings

## OOB
- OOB-to-request correlation: when a callback arrives, match it to the specific AI Agent test case that sent the payload and automatically link the interaction to the correct result card / finding

## Reporting
- Executive summary: one-page LLM-generated non-technical narrative (risk headline, top 3 findings, recommended next steps) generated separately from the full findings report
- PDF export: convert the styled HTML report to PDF via the system print/save-as-PDF API

## Performance / Architecture
- Result caching: skip LLM re-analysis when the same URL + parameter combination with the same transformation has already been analysed in the current session
- Streaming LLM responses: display partial AI output token-by-token as it streams in, improving perceived responsiveness for slow providers or large prompts
- Prompt compression: before sending to the LLM, intelligently truncate very large response bodies to avoid hitting token limits silently

## UX / Polish
- Scan completion notification: OS-level or Burp-status-bar notification when a long AI Agent run finishes (so users don't have to watch the tab)

#nice to have
- Dark/light theme that follows Burp's current theme automatically
- API schema auto-extraction: build a rough OpenAPI spec from observed in-scope traffic in the Traffic Analyzer
- Per-target LLM provider override: use a cheaper/faster model for passive scanning and a smarter model for AI Agent runs

#bugs
