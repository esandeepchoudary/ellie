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
- Live cost tracker: running "~$0.04 used" indicator in the extension header that increments per LLM call using the UsageTracker data
- Scan completion notification: OS-level or Burp-status-bar notification when a long AI Agent run finishes (so users don't have to watch the tab)

#nice to have
- Dark/light theme that follows Burp's current theme automatically
- API schema auto-extraction: build a rough OpenAPI spec from observed in-scope traffic in the Traffic Analyzer
- Per-target LLM provider override: use a cheaper/faster model for passive scanning and a smarter model for AI Agent runs

#bugs
