---
name: Appscan-fix
description: "Use when validating an AppScan issue fix, generating compact triage summaries, checking replay availability, and driving clue-first remediation from MCP issue metadata. Keywords: AppScan, Appscan-fix, fix verification, replay script, DAST issue, IAST clue-first mapping, issueId, get_issue_details."
---

# AppScan Fix Verification (Token-Lean)

## Purpose
Use this skill to validate and summarize a single AppScan issue with minimal token usage while still preserving reproducible evidence.

## Inputs
- `issueId` (required)
- `locale` (optional, default: `en-us`)
- `includeRawTraffic` (optional, default: `false`)
- `includeFullXml` (optional, default: `false`)

## Tooling
Use these tools in this order:
1. `mcp_appscan_mcp_s_get_issues` with filter: `Id eq <issueId>`
2. `mcp_appscan_mcp_s_get_issue_details` for evidence XML only when needed
3. `AltoroJ-boot` skill when compile/boot/runtime reachability verification is intended (hard gate for verification)

## Token-Lean Workflow
1. Fetch issue metadata first and return a compact summary table.
2. Explicitly announce each dataset as it is gathered and state what it will be used for.
3. Only fetch full XML details if user asks for proof, replay details, reproduction data, or remediation evidence.
4. From XML, extract only:
- HTTP method
- URL/path
- mutated parameters
- payload marker
- response evidence marker
5. If the issue has IAST context (`SourceFile`, `CallingMethod`, sink/source type hints, generated JSP servlet references), use these contextual clues first to map likely source and sink locations.
6. If the issue has a `CorrelationGroupId`, fetch the correlated issue set and summarize the shared sink, path, scanner mix, and dominant tainted elements before proposing a fix.
7. If correlation contains SAST members, fetch each SAST issue by Id and extract `Location`, `Line`, `SourceFile`, `SourceFileUri`, `Context`, `Api`, and `CallingMethod`.
8. For each SAST-correlated issue, perform repository lookup at the reported location and verify with a small line tolerance window of +/-3 lines. Mark the result as exact, near-match, or mismatch.
9. Cross-verify each SAST location match against IAST clues (`Path`, `Element`, `CallingMethod`, sink API family) and explicitly report whether SAST and IAST evidence align.
10. If SAST correlation is missing, perform an IAST-correlation-first assessment: collect correlated IAST issues and extract `Path`, `Element`, `Location`, `CallingMethod`, `SourceFile`, and sink API hints.
11. For IAST-only correlation, map runtime sink clues to editable source with repository lookup and verify candidate lines with a small tolerance window of +/-5 lines when exact line is unavailable.
12. In IAST-only correlation mode, classify mapping confidence as high/medium/low and require at least one concrete source-to-sink chain before remediation is declared.
13. Perform manual repository lookup only if IAST clues are missing, ambiguous, generated-only, or cannot be mapped to an editable source line with high confidence.
14. In the write-up, explicitly call out each contextual clue used and how it changed or narrowed analysis.
15. Apply the remediation to the local source file at the verified editable location as soon as data gathering, correlation, and remediation guidance are sufficient, unless the user requested analysis-only mode.
16. Do not echo full response bodies unless explicitly requested.
17. When DAST evidence is available and remediation is proposed or applied, generate a minimal patch verification script from fix verification data (method, path, mutated parameter, payload marker, and response marker).
18. Hard Gate: compile/boot/runtime verification is always in scope once a remediation patch is applied. Verification MUST run through the `AltoroJ-boot` skill before any compile or boot command is executed.
19. After remediation is applied, invoke the `AltoroJ-boot` skill and execute its workflow in order: resolve compatible tooling (environment first, `.tools` fallback), download missing compatible tools into `.tools` when needed, compile, boot on an available local port, and verify localhost reachability while leaving the app running.
20. If the hard gate is not satisfied, stop execution and mark verification as `Invalid - AltoroJ-boot gate not executed`.
21. If `AltoroJ-boot` reports successful boot, point the patch verification script to the local application host it reported (for example, `http://localhost:<detectedPort>`).
22. If `AltoroJ-boot` reports successful boot, run the patch verification script against that local host and assess pass/fail outcomes as fix verification evidence.
23. If `AltoroJ-boot` reports compile or boot failure, explicitly mark the runtime verification stage as blocked and continue with static/correlation-based verification only.
24. If user prompt is `yes fix the fix groups`, iterate through each issue in the correlation group and execute the same correlation-aware workflow used for individual fixes (SAST verification or IAST fallback per issue).
25. Print step status updates as each stage completes (do not wait until the end). Use dynamic status markers with the actual step name and outcome.
26. At the end, provide a concise process summary that includes only steps that were actually executed and their outcomes.
27. For remediation theory, select one issue from the correlation group in this priority order: SAST, then DAST, then IAST.
28. Retrieve `get_issue_details` for the selected issue and extract the `How to fix` content when present.
29. Build a `Security Theory and References` section from that issue-details payload with subheaders for Cause, Fix Recommendations, CWEs, and External References.

## Standard Output
Return the following sections in this order:
1. `Unified Evidence Window` (single compact object window that consolidates issue metadata, correlation, contextual clues, and source mapping evidence)
2. `Contextual Clues Used` (IAST source/sink/call-stack hints, generated-file references, and mapping confidence)
3. `Issue Snapshot` (severity, status, type, location, scanner, timestamps)
4. `AppScan Data Used` (each dataset pulled, the key fields used, and what remediation decision it informed)
5. `DAST Patch Verification Script` (conditional: include only when DAST evidence is available and script data can be derived)
6. `Local Runtime Verification` (conditional: include only when compile/boot verification is attempted and results are available)
7. `Remediation Basis` (explicit source-to-sink evidence chain plus embedded security theory and references from issue-details `How to fix` data)
8. `Process Summary` (dynamic status-marker recap of only executed steps and outcomes; place this section immediately before Next Steps Suggestion)
9. `Next Steps Suggestion` (include `FixGroupId` and deep link: `https://cloud.appscan.com/main/myapps/<AppID>/fixgroups/<FixGroupID>`)

## Rules
- Prefer concise structured output over long prose.
- Always show the dataset provenance explicitly. For each dataset, include:
	- dataset name
	- retrieval method/tool
	- fields or evidence extracted
	- how the dataset influenced triage, fix selection, or verification
- Always include source code location evidence when available:
	- file path
	- line number
	- source expression (tainted input)
	- sink expression (unsafe render/write)
- Always attempt clue-first mapping before manual lookup and disclose outcome:
	- clue sufficient -> no manual lookup required
	- clue partial -> manual lookup used to confirm editable source line
	- clue insufficient -> manual lookup required for localization
- If SAST correlation exists, per-member location verification is mandatory before declaring remediation complete:
	- retrieve each SAST member by Id
	- validate reported file+line in local source with +/-3 line tolerance
	- classify each SAST member as exact, near-match, or mismatch
	- cross-check each SAST member against IAST clues and explicitly report alignment/misalignment
- If SAST correlation is missing but IAST correlation exists, IAST fallback verification is mandatory before declaring remediation complete:
	- collect IAST correlated members and sink/source runtime clues
	- localize editable source candidates using `Path`, `Element`, `CallingMethod`, and sink API family
	- validate localized lines with +/-5 line tolerance when exact lines are not present in IAST metadata
	- produce at least one explicit source-to-sink chain and confidence rating (high/medium/low)
- In `Contextual Clues Used`, include impact statements such as:
	- which clues narrowed file candidates
	- which clues identified source/sink API families
	- whether generated servlet line references were mapped to authored JSP/templated source
- Redact or truncate session cookies/tokens by default.
- Treat stale cookies as non-portable and call this out.
- If replay script text is not available via MCP, generate an equivalent minimal patch verification script when DAST fix verification data exists.
- After data gathering, correlation verification, and remediation-theory extraction are complete, apply remediation to local source at the verified editable patch location unless user intent is analysis-only or remediation is blocked.
- Hard Gate: After any local remediation patch is applied, compile/boot/runtime verification is required and must invoke `AltoroJ-boot` before any compile/boot command. Direct verification commands outside `AltoroJ-boot` are prohibited unless user explicitly requests bypass.
- Compliance Check: If `AltoroJ-boot` evidence is missing (`compileResult`, `bootResult`, `detectedPort`, `localBaseUrl`, reachability outcomes), mark verification status as `Invalid - AltoroJ-boot gate not executed` and rerun via `AltoroJ-boot`.
- Fail Closed: If the hard gate fails or is skipped, do not report runtime verification success.
- If `AltoroJ-boot` reports compile and boot succeed, run runtime verification locally and include detected port plus localhost target used by the script.
- If `AltoroJ-boot` reports compile or boot fails, include the failure point and reason in `Local Runtime Verification` and mark it as blocked.
- When correlated issues exist, prefer one shared remediation narrative over repeating issue-by-issue prose.
- If user prompt is `yes fix the fix group issues`, process all issues in the correlation group in sequence and report per-issue verification outcomes.
- Source remediation theory from `get_issue_details` using this selection priority: SAST, then DAST, then IAST.
- Embed remediation theory directly inside `Remediation Basis`.
- In `Remediation Basis`, include these exact subheaders:
	- `Cause`
	- `Fix Recommendations`
	- `CWEs`
	- `External References`
- If a subheader cannot be populated from issue details payload, state `Not provided in issue details` for that subheader.
- Include `DAST Patch Verification Script` only when DAST-derived request/payload evidence exists.
- Include `Local Runtime Verification` for all remediation runs. If verification could not execute, report blocked status and reason.

## Dynamic Status Markers
Status markers are conditional and must reflect the real triage flow.
Print each step marker when that step completes during the workflow (live updates), not only in the final write-up.

Marker format:
- `**<icon> [<Step Name>]**`
- `<Outcome sentence>`

Visual presentation rules:
- Put each status update on its own separate block (blank line before and after each update).
- Keep the step name on a dedicated bold line.
- Put the outcome on the next line (not the same line as the step name).
- Use an icon that matches the status type.

Icon mapping:
- Dataset retrieval or collection: `🗂️`
- Contextual clue identification: `🧭`
- Source-to-sink mapping: `🔗`
- Correlation verification: `🧪`
- Patch applied: `🛠️`
- Verification success: `✅`
- Blocked/error: `⛔`
- Skipped/not applicable: `⏭️`

Examples:
- `**🗂️ [Issue Dataset Retrieval]**`
- `Collected successfully`
- `**🧪 [SAST Correlation Verification]**`
- `Completed (near-match, aligned with IAST)`
- `**⛔ [Runtime Verification]**`
- `Blocked (compile failure)`

Rules:
- Only print markers for steps actually attempted.
- Do not print a full checklist of all possible steps.
- Skip `Fallback` marker when fallback was not used.
- In the final `Process Summary`, include only executed steps and outcomes.

Recommended dynamic markers for fix-theory enrichment:
- `*[Fix-Theory Source Selection]* - Source issue selected`
- `*[How-To-Fix Retrieval]* - Collected successfully` or `Blocked/Unavailable`
- `*[Security Theory Synthesis]* - Completed` or `Partial`
- `*[Local Remediation Application]* - Patch applied` or `Blocked/Skipped by user intent`

## Object Window Format
Render one consolidated evidence window as a compact JSON object in a fenced `json` block.
Required window:
- `UnifiedEvidenceWindow`

Visual rendering rules for the window:
- Print the section heading `Unified Evidence Window` as a standalone bold header line.
- Immediately follow it with exactly one fenced `json` block containing the `UnifiedEvidenceWindow` object.
- Do not split this object across multiple windows or multiple JSON blocks.
- Do not place status markers inside the JSON block.
- Keep the JSON self-contained so markdown renderers display it as one standalone window.

Required fields for `UnifiedEvidenceWindow`:
- `window`
- `source`
- `retrievedVia`
- `issueSnapshot`
- `correlationSummary`
- `contextualClues`
- `sourceLocationEvidence`
- `sastIastAlignment`
- `keyFindings`
- `decisionImpact`

Required subfields inside `sourceLocationEvidence`:
- `sourceFile`
- `sourceLine`
- `sourceExpression`
- `sinkFile`
- `sinkLine`
- `sinkExpression`
- `confidence`

Required subfields inside `contextualClues`:
- `clueType`
- `clueValue`
- `clueOrigin`
- `mappingOutcome`
- `mappingConfidence`

Required subfields inside `sastIastAlignment`:
- `sastIssueId`
- `reportedFile`
- `reportedLine`
- `lookupWindow`
- `lookupResult`
- `lineTolerance`
- `toleranceClassification`
- `iastCrossCheck`
- `iastAlignment`
- `fallbackUsed`

Required fields for `LocalRuntimeVerificationWindow`:
- `window`
- `source`
- `retrievedVia`
- `compileAttempted`
- `compileResult`
- `bootAttempted`
- `bootResult`
- `detectedPort`
- `localBaseUrl`
- `scriptExecuted`
- `scriptResult`
- `blockedReason`
- `keyFindings`
- `decisionImpact`

Notes:
- Use only one evidence window in the write-up for vulnerability evaluation.
- Do not emit separate IssueMetadata/Correlation/Context/SAST windows.
- Keep `LocalRuntimeVerificationWindow` separate only when local compile/boot verification is attempted.
- Remove `fieldsUsed` from evidence windows to keep output succinct.

## Compact Field Set
Use this exact compact set unless user asks for more:
- `Id`, `IssueType`, `Severity`, `Status`, `DiscoveryMethod`, `Location`
- `Cvss`, `CvssVector`, `Cwe`, `Scanner`, `ScanName`
- `DateCreated`, `LastUpdated`, `LastFound`
- `ReplayScriptFrameworks`, `RemediationId`, `DiffResult`, `CorrelationGroupId`

## Dataset Provenance Template
Use this exact compact format when reporting AppScan data usage:
- `Issue Metadata`: retrieved with `get_issues`; used fields: `Id`, `ApplicationId`, `IssueType`, `Severity`, `Status`, `DiscoveryMethod`, `Location`, `CorrelationGroupId`; informed deep link creation, scanner/source classification, and correlation check.
- `Correlation Dataset`: retrieved with `get_issues` filtered by `CorrelationGroupId`; used fields: `Id`, `IssueType`, `Severity`, `DiscoveryMethod`, `Path`, `Element`, `SourceFile`, `CallingMethod`; informed shared fix scope and source-to-sink grouping.
- `SAST Correlation Detail Dataset`: retrieved with `get_issues` per SAST `Id`; used fields: `Location`, `Line`, `SourceFile`, `SourceFileUri`, `Context`, `Api`, `CallingMethod`; informed exact patch target and per-member completion criteria.
- `IAST Correlation Fallback Dataset`: retrieved with `get_issues` for correlated IAST members; used fields: `Path`, `Element`, `Location`, `CallingMethod`, `SourceFile`, sink API hints; informed localization and source-to-sink verification when SAST location data is unavailable.
- `Evidence XML`: retrieved with `get_issue_details`; used evidence: request method, path, mutated parameters, payload marker, reflected response marker, reasoning text; informed repro script generation and verification criteria.
- `Contextual Clues Dataset`: retrieved from IAST fields and issue details; used evidence: sink/source type hints, runtime call stack, source file/class references, generated servlet locations; informed clue-first localization strategy.
- `Local Code Dataset`: retrieved from repository search and targeted file reads only when needed; used evidence: source parameter reads, render sinks, shared utility methods; informed exact editable code patch location.
- `How-To-Fix Dataset`: retrieved with `get_issue_details` from one prioritized correlated issue (SAST, else DAST, else IAST); used evidence: cause text, remediation/fix recommendations, CWE values, and external references; informed `Security Theory and References` and strengthened remediation rationale.

## Remediation Basis Enriched Format
Security theory and references must be embedded in `Remediation Basis`.

Use this structure:
- `Source Issue for How-To-Fix`: include issue id and discovery method.
- `Cause`: summarize issue-details cause/problem mechanism.
- `Fix Recommendations`: summarize actionable fix guidance from issue-details remediation text.
- `CWEs`: include CWE IDs and names when available.
- `External References`: include links returned in issue details. If none are present, state `Not provided in issue details`.

Selection priority when a correlation group exists:
1. SAST issue details
2. DAST issue details
3. IAST issue details

If no correlation group exists, use the primary issue id.

Do not describe priority rationale in the final write-up.

## DAST Patch Verification Script Requirements
When `DiscoveryMethod` includes DAST or DAST correlation evidence is present:
- Generate a minimal verification script (default Python `requests`) that includes:
	- target base URL
	- HTTP method and path from evidence
	- mutated parameter and payload marker value
	- response verification assertions for vulnerable marker absence and expected safe behavior
- Redact or omit stale session cookies by default.
- Include script inputs as variables so payload and endpoint can be reused for regression checks.
- If response marker from evidence is available, use it as explicit `must_not_contain` assertion.
- If local boot succeeds, set target base URL to localhost with the detected runtime port for script execution.
- In the write-up, report whether the script was executed locally and summarize the result or blocked reason.

## Next Steps Suggestion Format
When providing next steps after assessment/remediation:
- If a fix group exists, present it as one combined next-step item using this wording pattern:
	- `A fix group is available for this issue: https://cloud.appscan.com/main/myapps/<AppID>/fixgroups/<FixGroupID>`
- Keep the fix-group statement and deep link in the same bullet/line so they read as one issue.
- If no fix group exists, provide the App-level link only and avoid mentioning internal field names.
- If remediation and local verification succeed, include a suggested next step to open a Pull Request titled with fix type and issue id in this format: `Fix(<IssueType>): <IssueId>`.
- If `Local Runtime Verification` indicates the application was booted and is still running, add a conditional cleanup next step that advises stopping the running process (include the known terminal/process id when available).
- End with a natural follow-up prompt when appropriate, for example: `Shall we fix the rest next?`

## Example Invocation Phrases
- "Verify AppScan fix for issue <id> with minimal output"
- "Get replay capability and repro request for <id>"
- "Summarize DAST issue <id> for triage, token-lean mode"
