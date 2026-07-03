# Skills Overview

This folder contains reusable Copilot skills for the AltoroJ workspace.

## What is a skill?

A skill is a focused workflow with clear inputs, tool usage order, and output expectations.
Skills are intended to make repeated tasks consistent, faster, and easier to verify.

## Available skills

### 1) altoroj-boot

Location: `altoroj-boot/SKILL.md`

Intended use:
- Build and boot AltoroJ locally in a repeatable way.
- Resolve compatible Java/Gradle/Jetty tooling.
- Verify localhost reachability after startup.

How it works:
1. Detect environment tools (`JAVA_HOME`, `java`, `gradle`) and check compatibility.
2. Resolve local fallback tools under `.tools`.
3. Download missing compatible tools into `.tools` only if needed.
4. Compile and build the WAR file.
5. Select an available local port.
6. Start Jetty Runner in background mode.
7. Verify the app is reachable on localhost endpoints.
8. Return a structured summary (toolchain, build result, port, reachability, process id).

Typical trigger phrases:
- "Boot AltoroJ locally"
- "Compile and run AltoroJ"
- "Verify AltoroJ is running on localhost"

### 2) appscan-fix

Location: `appscan-fix/SKILL.md`

Intended use:
- Triage and remediate AppScan findings.
- Validate fixes with evidence and correlation-aware analysis.
- Generate compact security-focused reports and verification scripts.

How it works:
1. Retrieve issue metadata by issue id.
2. Collect detailed issue evidence (XML) when proof or reproduction is needed.
3. Prefer clue-first mapping from IAST/SAST/DAST correlation data.
4. Map source-to-sink in local code with confidence classification.
5. Apply remediation patch at verified editable source location (unless analysis-only).
6. Run local compile/boot verification through AltoroJ-boot gate.
7. Execute minimal verification checks/scripts when DAST evidence is available.
8. Return structured sections: evidence window, data provenance, remediation basis, process summary, and next steps.

Typical trigger phrases:
- "Appscan-fix <issue-id>"
- "Verify AppScan fix for issue <issue-id>"
- "Triage this DAST/IAST/SAST issue and propose remediation"

## How to choose the right skill

- Use altoroj-boot when your main goal is local build/startup/reachability.
- Use appscan-fix when your main goal is security issue triage, remediation, and verification.
- appscan-fix may invoke altoroj-boot as a verification gate after a code patch.

## Notes

- Keep skill workflows deterministic and evidence-driven.
- Prefer compact outputs that still include traceable provenance.
- Update each skill's `SKILL.md` if process requirements change.

Skills MD was scanned for security concerns with Backslash Security Scanner and found to have no security threats:
https://www.backslash.security/

<img width="1042" height="916" alt="image" src="https://github.com/user-attachments/assets/33865bf0-9a22-4554-9ae2-d79d0a549edc" />

<img width="1062" height="907" alt="image" src="https://github.com/user-attachments/assets/08b1ff46-b3de-4721-833e-fdc4ef8aa323" />


