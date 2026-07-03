---
name: Appscan-fix
description: "Use when validating an AppScan issue fix, generating compact triage summaries, checking replay availability, and driving clue-first remediation from MCP issue metadata. Keywords: AppScan, Appscan-fix, fix verification, replay script, DAST issue, IAST clue-first mapping, issueId, get_issue_details."
---

# AppScan Fix workflow

## Purpose
Use this skill to validate and summarize AppScan issues using a correlation-first workflow that minimizes token use while preserving reproducible evidence.

## Inputs
- `issueId` (required)
- `locale` (optional, default: `en-us`)
- `includeRawTraffic` (optional, default: `false`)
- `includeFullXml` (optional, default: `false`)

## Execution Contract (Authoritative)
Follow this contract in order. This section is the single source of truth for execution order.

Phase model (mandatory):
- Phase 0: Preflight and intent lock
- Phase 1: Evidence collection (metadata/correlation/details)
- Phase 2: Localization and remediation decision
- Phase 3: Code change
- Phase 4: Runtime verification
- Phase 5: Reporting

Hard rule:
- Never enter Phase 3 (code change) before Phase 1 is complete.
- Never run post-patch metadata backfill queries to satisfy missed gates. If a gate was missed, report `Invalid run - gate order violated` and restart evidence collection before additional code changes.

1. Metadata gate (mandatory first step):
	- Call `mcp_appscan_mcp_s_get_issues` with filter `Id eq <issueId>`.
	- Extract at minimum: `Id`, `ApplicationId`, `IssueType`, `Severity`, `Status`, `DiscoveryMethod`, `Location`, `CorrelationGroupId`, `RemediationId`.
	- Do not call `mcp_appscan_mcp_s_get_issue_details` before this gate completes.

2. Correlation gate (mandatory before details):
	- If `CorrelationGroupId` exists, call `mcp_appscan_mcp_s_get_issues` with filter `CorrelationGroupId eq <CorrelationGroupId>`.
	- Summarize shared sink/path/scanner mix before proposing a fix.
	- If correlation data retrieval is unavailable, report `Correlation check blocked - metadata endpoint unavailable` and mark confidence reduced.

3. Details retrieval (conditional):
	- Call `mcp_appscan_mcp_s_get_issue_details` only when remediation evidence, replay details, or theory extraction is required.
	- Extract only the minimum needed evidence: HTTP method, URL/path, mutated parameter(s), payload marker, response marker, and reasoning.

4. Clue-first localization:
	- Use IAST/runtime clues first when present (`SourceFile`, `CallingMethod`, source/sink hints, generated servlet references).
	- Use manual repository lookup only when clues are missing, ambiguous, or non-editable.

5. Correlated verification logic:
	- If SAST correlated issues exist, verify each reported file/line with +/-3 line tolerance and classify `exact`, `near-match`, or `mismatch`.
	- Cross-check each SAST location against IAST clues and report alignment.
	- If SAST is absent but IAST correlation exists, use IAST fallback with +/-5 line tolerance and provide at least one source-to-sink chain with confidence rating.

6. Remediation execution:
	- Apply remediation at the verified editable location unless user requested analysis-only mode.
	- If any required Phase 1 dataset is missing, do not patch; mark run invalid and stop.

7. Runtime verification hard gate:
	- After patching, runtime verification must use the `AltoroJ-boot` skill before any direct compile/boot command.
	- If hard gate evidence is missing, mark `Invalid - AltoroJ-boot gate not executed`.
	- If compile/boot succeeds, run patch verification against localhost and report pass/fail.

8. Fix-theory enrichment:
	- For correlation groups, pick one issue for theory extraction in this order: SAST, then DAST, then IAST.
	- Reuse already retrieved `get_issue_details` evidence when possible.
	- Call one additional `get_issue_details` only when required to populate Cause, Fix Recommendations, CWEs, External References.

9. Process signaling:
	- Print dynamic status markers only for steps actually executed.
	- End with a concise process summary and next steps suggestion.

## Standard Output
Return the following sections in this order:
1. `Workflow Status`
2. `Unified Evidence Window`
3. `Contextual Clues Used`
4. `Issue Snapshot`
5. `AppScan Data Used`
6. `DAST Patch Verification Script` (conditional)
7. `Local Runtime Verification` (conditional)
8. `Remediation Basis`
9. `Process Summary` (immediately before Next Steps Suggestion)
10. `Next Steps Suggestion` (include `FixGroupId` deep link: `https://cloud.appscan.com/main/myapps/<AppID>/fixgroups/<FixGroupID>`)

`Workflow Status` must include:
- Run validity (`Valid` or `Invalid`).
- Completed phases (`Phase 0` through `Phase 5`).
- Gate status (`Metadata`, `Correlation`, `AltoroJ-boot`) as `pass|blocked|skipped`.
- If invalid, a single-line reason and where execution stopped.

## Rules
- Prefer concise structured output over long prose.
- Respect the `Execution Contract (Authoritative)` ordering.
- Evidence-first is mandatory: complete metadata and correlation gates before any source edit, compile, or boot.
- No retroactive gating: once code changes begin, do not issue additional gate-completion metadata queries for the same run.
- For each dataset, always include provenance: dataset name, retrieval tool, fields/evidence used, and remediation decision impact.
- Always include source location evidence when available: file path, line, source expression, sink expression.
- Always attempt clue-first mapping and state outcome as `clue sufficient`, `clue partial`, or `clue insufficient`.
- If SAST correlation exists, verify each SAST member with +/-3 line tolerance, classify `exact|near-match|mismatch`, and cross-check against IAST clues.
- If SAST is absent but IAST correlation exists, perform IAST fallback verification with +/-5 line tolerance and provide at least one source-to-sink chain with confidence.
- In `Contextual Clues Used`, include which clues narrowed files, identified sink/source API families, and whether generated servlet references were mapped to authored source.
- Redact or truncate session cookies/tokens by default.
- Treat stale cookies as non-portable and call this out.
- If replay script text is not available via MCP, generate an equivalent minimal patch verification script when DAST fix verification data exists.
- After data gathering, correlation verification, and remediation-theory extraction are complete, apply remediation to local source at the verified editable patch location unless user intent is analysis-only or remediation is blocked.
- Runtime hard-gate behavior is defined in `Execution Contract (Authoritative)` step 7 and must be treated as mandatory and fail-closed.
- When correlated issues exist, prefer one shared remediation narrative over repeating issue-by-issue prose.
- If user prompt is `yes fix the fix group issues`, process all issues in the correlation group in sequence and report per-issue verification outcomes.
- Source remediation theory from `get_issue_details` using this selection priority: SAST, then DAST, then IAST.
- Embed remediation theory directly inside `Remediation Basis`.
- In `Remediation Basis`, include exact subheaders: `Cause`, `Fix Recommendations`, `CWEs`, `External References`.
- If a subheader cannot be populated from issue details payload, state `Not provided in issue details` for that subheader.
- Include `DAST Patch Verification Script` only when DAST-derived request/payload evidence exists.
- Include `Local Runtime Verification` for all remediation runs. If verification could not execute, report blocked status and reason.

## Minimal Tooling Reference
Use tools in this order:
1. `mcp_appscan_mcp_s_get_issues` (`Id eq <issueId>`) and optional second call for `CorrelationGroupId`.
2. `mcp_appscan_mcp_s_get_issue_details` only after metadata/correlation gates.
3. Optional one additional `mcp_appscan_mcp_s_get_issue_details` for prioritized fix-theory source (SAST, else DAST, else IAST).
4. `AltoroJ-boot` for any post-patch compile/boot/runtime verification.

## Dynamic Status Markers
Status markers are conditional and must reflect only the executed flow.
Print each marker as the step completes, not only in the final write-up.

Icon mapping:
- Dataset retrieval or collection: `🗂️`
- Contextual clue identification: `🧭`
- Source-to-sink mapping: `🔗`
- Correlation verification: `🧪`
- Patch applied: `🛠️`
- Verification success: `✅`
- Blocked/error: `⛔`
- Skipped/not applicable: `⏭️`

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

## Appendix A - Compact Field Set
Use this exact compact set unless user asks for more:
- `Id`, `IssueType`, `Severity`, `Status`, `DiscoveryMethod`, `Location`
- `Cvss`, `CvssVector`, `Cwe`, `Scanner`, `ScanName`
- `DateCreated`, `LastUpdated`, `LastFound`
- `ReplayScriptFrameworks`, `RemediationId`, `DiffResult`, `CorrelationGroupId`

## Appendix B - Dataset Provenance Template
Use this compact format when reporting AppScan data usage:
- `Issue Metadata`: retrieved with `get_issues`; fields include `Id`, `ApplicationId`, `IssueType`, `Severity`, `Status`, `DiscoveryMethod`, `Location`, `CorrelationGroupId`.
- `Correlation Dataset`: retrieved with `get_issues` filtered by `CorrelationGroupId`; fields include `Id`, `IssueType`, `Severity`, `DiscoveryMethod`, `Path`, `Element`, `SourceFile`, `CallingMethod`.
- `SAST Correlation Detail Dataset`: retrieved with `get_issues` per SAST `Id`; fields include `Location`, `Line`, `SourceFile`, `SourceFileUri`, `Context`, `Api`, `CallingMethod`.
- `IAST Correlation Fallback Dataset`: retrieved with `get_issues` for correlated IAST members; fields include `Path`, `Element`, `Location`, `CallingMethod`, `SourceFile`, sink API hints.
- `Evidence XML`: retrieved with `get_issue_details`; evidence includes request method/path, mutated parameters, payload marker, reflected marker, reasoning.
- `Contextual Clues Dataset`: retrieved from IAST fields and issue details; evidence includes sink/source hints and call-stack clues.
- `Local Code Dataset`: retrieved from repository lookup only when needed.
- `How-To-Fix Dataset`: retrieved with `get_issue_details` from prioritized source issue (SAST, else DAST, else IAST).

## Object Window Format
Render one consolidated evidence window as a compact JSON object in a fenced `json` block.
Required window:
- `UnifiedEvidenceWindow`

Rendering rules:
- Print `Unified Evidence Window` followed by exactly one fenced `json` block containing `UnifiedEvidenceWindow`.
- Do not split the object across multiple JSON blocks.
- Do not place status markers inside the JSON block.

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
