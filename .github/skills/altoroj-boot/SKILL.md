---
name: altoroj-boot
description: "Use when building and booting AltoroJ locally with environment-first tool resolution, .tools fallback, compatibility checks, conditional tool downloads, compile verification, runtime port checks, and no cleanup. Keywords: AltoroJ boot, local Gradle, local JDK, Jetty runner, compile verify, localhost check."
---

# AltoroJ Local Boot Skill

## Purpose
Provide a repeatable workflow to compile and boot AltoroJ on localhost without modifying machine-wide settings, and without stopping the server at the end.

## Inputs
- workspaceRoot (required)
- preferredPort (optional, default: 8080)
- keepRunning (default: true)

## Tool Resolution Order
Resolve tools in this order:
1. Environment-configured tools first (JAVA_HOME, java, gradle in PATH).
2. Project-local tools in .tools second.
3. Download missing compatible tools into .tools only when no suitable tool is available.

## Compatibility Guidance For This AltoroJ Workspace
Project facts from build configuration:
- Java sourceCompatibility: 1.8
- Java targetCompatibility: 1.8

Most suitable combinations:
1. Preferred local combo: JDK 11 + Gradle 7.6.4 + Jetty Runner 9.4.x
2. Compatibility-first legacy combo: JDK 8 + Gradle 6.9.4 + Jetty Runner 9.4.x

Unsuitable combinations to avoid:
- Gradle 7.4.2 with JDK 21+ (known class file compatibility failures in this workflow).
- JRE-only runtime when compilation is required (javac missing).

Locally available tools commonly used in this repository:
- .tools/gradle-6.9.4
- .tools/gradle-7.6.4
- .tools/gradle-8.10.2
- .tools/jdk-11.0.31+11
- .tools/jetty-runner-9.4.53.v20231009.jar

## Workflow

### 1) Detect Environment Tooling
Collect and report:
- JAVA_HOME
- java -version
- gradle -v
- where java
- where gradle

Decision:
- If environment tools are compatible, they may be used.
- If environment tools are incompatible, use .tools resolution next.

### 2) Resolve .tools Tooling
Check for these local paths:
- .tools/jdk-11*/bin/java(.exe)
- .tools/jdk-11*/bin/javac(.exe)
- .tools/gradle-7.6.4/bin/gradle(.bat)
- .tools/jetty-runner-9.4.53.v20231009.jar

Decision:
- If present and compatible, use them.
- If missing, download into .tools.

### 3) Conditional Downloads To .tools (Only If Needed)
Download examples:
- JDK 11 (Adoptium API):
  https://api.adoptium.net/v3/binary/latest/11/ga/windows/x64/jdk/hotspot/normal/eclipse
- Gradle 7.6.4:
  https://services.gradle.org/distributions/gradle-7.6.4-bin.zip
- Gradle 6.9.4 (legacy fallback):
  https://services.gradle.org/distributions/gradle-6.9.4-bin.zip
- Jetty Runner 9.4.53:
  https://repo1.maven.org/maven2/org/eclipse/jetty/jetty-runner/9.4.53.v20231009/jetty-runner-9.4.53.v20231009.jar

Rules:
- Download only missing compatible components.
- Extract archives under .tools.
- Do not alter global environment variables.

### 4) Compile Verification
Use command-scoped environment overrides only:
- Set JAVA_HOME to chosen JDK for this command/session scope.
- Prepend chosen JDK bin to PATH for this command/session scope.

Run:
- gradle --no-daemon clean compileJava (or war when packaging is needed)

Success criteria:
- BUILD SUCCESSFUL
- compileJava task succeeded

Failure handling:
- If compile fails due version mismatch, switch to the other compatible combo and retry.
- If compile fails due code error, stop and report compile diagnostics.

### 5) Build WAR
Run:
- gradle --no-daemon war

Artifact expected:
- build/libs/altoromutual.war

### 6) Select Port
Port policy:
1. Try preferredPort (default 8080).
2. If unavailable, probe and choose next available from 8081-8095.
3. Report the selected port.

### 7) Boot Application (Do Not Cleanup)
Start Jetty Runner in async/background mode:
- java -jar .tools/jetty-runner-9.4.53.v20231009.jar --port <selectedPort> --path / build/libs/altoromutual.war

Startup success indicators:
- AltoroJ initialized
- No fatal stack trace on startup

Important:
- Keep process running after verification unless user explicitly requests shutdown.

### 8) Reachability Verification
Perform HTTP checks:
- GET http://localhost:<port>/
- GET http://localhost:<port>/login.jsp

Pass criteria:
- HTTP 200 on both endpoints
- Response body includes expected marker text (for example Altoro Mutual or login page markers)

### 9) Output Summary
Return:
- Toolchain source used (env or .tools)
- Exact versions selected
- Compile result
- WAR artifact path
- Selected port
- Reachability results
- Server terminal/process id
- Explicit statement: server left running (no cleanup performed)

## Example Command Pattern (PowerShell)
Use scoped environment overrides and restore afterwards:

- Save current JAVA_HOME and PATH
- Set temporary JAVA_HOME and PATH
- Run Gradle compile/war
- Start Jetty runner async
- Run localhost checks
- Restore JAVA_HOME and PATH

Note:
Restoring shell variables does not stop the booted server process.

## Rules
- Prefer environment-configured tooling only when compatible.
- Otherwise prefer .tools local toolchain.
- Download to .tools only when needed.
- Never require machine-wide env var changes.
- Never kill the runtime process at the end of this workflow.
- Only stop runtime when explicitly requested by the user.
