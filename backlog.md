#complete
- Right-click "Send to AI Agent" now works in Proxy, Repeater, and Target tabs (messageEditorRequestResponse fallback)
- Pause / Resume button for AI Agent test runs
- Drill-down Details dialog now shows request/response in Burp native editors with follow-up chat
- Export test results to CSV / JSON / HTML after a run completes
- Chat in AI Agent tab retains full context of test results (summary added to conversation history)
- UI responsive layout: removed fixed pixel max-heights across Dashboard, OOB, Settings panels; OOBPanel sidebar is now a resizable JSplitPane
- Provider/model in extension header now updates live when settings are saved (was static, showed stale values)

#to do
- Enable the user to drill down into the request and response for each request being sent by the AI Agent
- Enable the AI agent to summarize all the test cases run and the obeserved outcome
- Make the agent smart to identify and diagnose issues during execution to pivot such as WAF bypasses, etc.
- once execution is complete, enable transferring the content to a report export such as csv, json, html for AI Agent
- at user terminiation provide an option to still export the results for AI Agent
- does it make sense to divide this into seperate agents that plan and test seperately

#nice to have

#bugs
