# ada

Given your actual setup:

* Scanner: Java executable JAR.
* Runtime: Custom jlink runtime.
* Target: Windows only.
* Current VSIX size: Approximately 43 MB.
* Existing implementation: Working VS Code extension with static ADA/WCAG scanning, HTML reporting, scan commands and interactive findings.

I would not rewrite your existing implementation. I would evolve it incrementally.

My recommendation is a TypeScript VS Code extension + standalone Java scanner + embedded Windows runtime architecture.

The objective is to achieve four things:

* Reduce the VSIX size without requiring users to install Java.
* Make the Java scanner independent of VS Code.
* Improve scanning performance, reliability and maintainability.
* Prepare the extension for future CI/CD integration and advanced accessibility checks.

1. Final recommended architecture

ADA VS Code Extension

TypeScript · VS Code Extension API

Commands · Diagnostics · Tree View · Webview · Settings

JSON over standard input/output

Scanner Client / Process Manager

Java runtime resolution · Process lifecycle · Cancellation · Progress

Java Accessibility Scanner

Executable JAR · Java 17/21 (use your existing version)

Static Analysis

HTML · Angular · React · CSS · WCAG rules

Rule Engine

Rule metadata · Severity · Confidence · Findings

Common Result Model

Normalized JSON findings · HTML report · SARIF · VS Code Problems

Windows Runtime

Custom jlink image · java.exe · Required Java modules

The key architectural principle:

The Java scanner should not have any dependency on VS Code.

Your scanner should be independently executable. The extension should simply orchestrate it and present its results.

This also means you can reuse the scanner later in:

* Jenkins pipelines.
* GitHub Actions.
* Azure DevOps.
* Command-line accessibility assessments.
* Other IDE integrations.

2. Recommended project structure

I would gradually organize your existing project like this.

ada-vscode-extension/
│
├── src/
│   ├── extension.ts
│   │
│   ├── commands/
│   │   ├── scanProject.ts
│   │   ├── scanFile.ts
│   │   ├── scanChangedFiles.ts
│   │   └── showReport.ts
│   │
│   ├── scanner/
│   │   ├── scannerClient.ts
│   │   ├── runtimeResolver.ts
│   │   ├── processManager.ts
│   │   ├── scannerProtocol.ts
│   │   └── scannerConfiguration.ts
│   │
│   ├── diagnostics/
│   │   ├── diagnosticProvider.ts
│   │   └── codeActionProvider.ts
│   │
│   ├── reports/
│   │   ├── reportProvider.ts
│   │   └── reportManager.ts
│   │
│   └── models/
│       ├── finding.ts
│       └── scanResult.ts
│
├── resources/
│   ├── scanner/
│   │   └── ada-scanner.jar
│   │
│   └── runtime/
│       └── win32-x64/
│           ├── bin/
│           │   ├── java.exe
│           │   └── ...
│           ├── conf/
│           ├── lib/
│           └── release
│
├── media/
├── package.json
├── tsconfig.json
├── .vscodeignore
└── README.md

This is a target structure, not a requirement to move every existing file immediately.

Important: Keep the Java scanner in its own repository or build module if possible. The VS Code extension repository should consume its versioned build artifact.

That gives you independent releases:

* Extension version: 1.5.0
* Scanner version: 2.3.0
* Protocol version: 1

3. Optimize your 43 MB VSIX

Since you already use a custom jlink runtime, I would focus on measuring and removing unnecessary components.

First, understand where the 43 MB is going.

Run this PowerShell script from your extension project:

Get-ChildItem .\resources -Recurse -File |
    ForEach-Object {
        [PSCustomObject]@{
            File = $_.FullName
            SizeMB = [math]::Round(
                $_.Length / 1MB, 2
            )
        }
    } |
    Sort-Object SizeMB -Descending |
    Format-Table -AutoSize

Also measure the complete packaged VSIX, not just the resources folder.

My optimization priorities

Component	Optimization
JRE	Remove unused Java modules
Scanner JAR	Remove unused dependencies
Node modules	Bundle only production dependencies
Source files	Exclude tests, maps and development assets
Reports	Generate at runtime, don’t package sample reports
Documentation	Exclude unnecessary development documents

3.1 Optimize the jlink runtime

Use jdeps to identify required Java modules.

For example, assuming Java 17:

jdeps `
  --multi-release 17 `
  --print-module-deps `
  --class-path "target\lib\*" `
  "target\ada-scanner.jar"

Use the actual Java version and dependency layout of your scanner.

Then generate the runtime:

jlink `
  --module-path "$env:JAVA_HOME\jmods" `
  --add-modules <MODULES_FROM_JDEPS> `
  --bind-services `
  --strip-debug `
  --no-man-pages `
  --no-header-files `
  --compress=2 `
  --output "resources\runtime\win32-x64"

Replace the module placeholder with the actual dependency list.

A few important considerations:

* Include jdk.crypto.ec if your scanner needs the relevant TLS cryptographic provider.
* Include jdk.localedata if your scanner needs additional locale data.
* Include modules required through reflection or service loading, even if static dependency analysis misses them.
* Test the resulting runtime with your complete scanner and all its rules.

Do not blindly remove Java modules to reduce size.

Also, if your executable JAR is a Spring Boot fat JAR or contains nested dependencies, make sure dependency analysis accounts for those nested libraries. jdeps does not automatically analyze every nested JAR layout.

3.2 Package only Windows x64

Since you have confirmed Windows-only support, there is no reason to package Linux and macOS runtimes.

Build the Windows runtime on a compatible Windows build environment using the target JDK.

Package using:

npx @vscode/vsce package --target win32-x64

This produces a Windows x64-specific VSIX.

If you later need Windows ARM64, build and distribute that as a separate target.

My size recommendation: First establish the size of the runtime and JAR separately. Then optimize. A 43 MB self-contained offline extension is not inherently unreasonable; reliability and ease of enterprise deployment are more important than achieving an arbitrary size target.

4. Runtime resolution — make it enterprise-ready

Currently, your extension presumably invokes the bundled Java executable.

I recommend introducing a dedicated runtimeResolver.ts.

The runtime resolution priority should be:

* Configured Java runtime, if the user explicitly provides one.
* Bundled custom jlink runtime.
* Clear error if neither is available.

Because you want a self-contained extension, the bundled runtime should remain the default.

Example:

import * as vscode from "vscode";
import * as path from "path";
import * as fs from "fs";
export function resolveJavaRuntime(
    context: vscode.ExtensionContext
): string {
    const configuredRuntime =
        vscode.workspace.getConfiguration("ada")
            .get<string>("javaExecutable");
    if (configuredRuntime) {
        if (!fs.existsSync(configuredRuntime)) {
            throw new Error(
                "Configured Java executable does not exist."
            );
        }
        return configuredRuntime;
    }
    const bundledRuntime = path.join(
        context.extensionPath,
        "resources",
        "runtime",
        "win32-x64",
        "bin",
        "java.exe"
    );
    if (!fs.existsSync(bundledRuntime)) {
        throw new Error(
            "Bundled Java runtime is missing."
        );
    }
    return bundledRuntime;
}

Add a setting in package.json:

{
  "contributes": {
    "configuration": {
      "title": "ADA Scanner",
      "properties": {
        "ada.javaExecutable": {
          "type": "string",
          "default": "",
          "description": "Optional path to a Java executable. Leave empty to use the bundled runtime."
        }
      }
    }
  }
}

The important detail is that you should never rely on the user’s system PATH or assume Java is installed.

Also validate the Java version before launching the scanner.

5. Improve Java scanner communication

This is probably the most important engineering improvement.

Avoid tightly coupling TypeScript to Java command-line output.

I recommend a versioned JSON Lines protocol.

Extension sends a scan request

{
  "protocolVersion": 1,
  "requestId": "scan-1001",
  "operation": "SCAN_PROJECT",
  "projectPath": "C:/workspace/my-angular-app",
  "configuration": {
    "wcagVersion": "2.1",
    "conformanceLevel": "AA",
    "includeManualCriteria": true
  }
}

Scanner sends progress

{
  "type": "PROGRESS",
  "requestId": "scan-1001",
  "percentage": 45,
  "message": "Analyzing Angular templates"
}

Scanner sends findings

{
  "type": "FINDING",
  "requestId": "scan-1001",
  "finding": {
    "ruleId": "IMG_ALT_001",
    "severity": "HIGH",
    "file": "src/app/home.html",
    "line": 15,
    "column": 5,
    "wcagCriteria": ["1.1.1"],
    "message": "Image is missing alternative text"
  }
}

Scanner completes

{
  "type": "COMPLETED",
  "requestId": "scan-1001",
  "summary": {
    "totalFindings": 12,
    "critical": 2,
    "high": 5,
    "medium": 5
  }
}

The extension should consume these messages and update the UI as the scan progresses.

Java process invocation

Use Node.js child_process.spawn().

import { spawn } from "child_process";
import * as path from "path";
const javaPath = resolveJavaRuntime(context);
const scannerJar = path.join(
    context.extensionPath,
    "resources",
    "scanner",
    "ada-scanner.jar"
);
const process = spawn(
    javaPath,
    [
        "-Dfile.encoding=UTF-8",
        "-jar",
        scannerJar,
        "--stdio"
    ],
    {
        cwd: workspacePath,
        shell: false,
        windowsHide: true,
        stdio: ["pipe", "pipe", "pipe"]
    }
);

The Java application should read requests from standard input and write protocol messages to standard output.

Important implementation rules:

* Standard output must contain only protocol JSON.
* Send application logs to standard error.
* Use a buffered JSON Lines parser in TypeScript.
* Validate requestId and protocolVersion.
* Handle malformed responses and unexpected process termination.
* Implement cancellation and timeouts.
* Limit message sizes and avoid buffering the entire scan output in memory.
* Do not invoke a shell to execute the scanner.

I would initially run one Java process per scan. This avoids long-running process lifecycle problems, memory leaks and stale scanner state.

6. Introduce a centralized accessibility rule engine

Your screenshots show that you already have a distinction between automated and manual verification.

I would formalize this rather than redesigning the existing scanning logic.

Use a central rule definition model.

For example:

public class AccessibilityRule {
    private String ruleId;
    private String title;
    private String description;
    private String wcagCriteria;
    private String severity;
    private String verificationType;
    private String confidence;
    private String framework;
    private String remediation;
}

Organize rules into separate packages:

scanner/
  rules/
    html/
    angular/
    react/
    javascript/
    css/
    wcag/
  analysis/
    parser/
    ast/
    template/
  engine/
    RuleEngine.java
    RuleRegistry.java
  models/
    Finding.java
    ScanResult.java
    AccessibilityRule.java

One important improvement: confidence classification

Not every static finding is equally reliable.

For example:

Finding	Confidence
Image missing alt attribute	High
Empty button accessible name	High, subject to context
Dynamic ARIA attribute	Medium
Color contrast inferred from source CSS	Medium or low
Keyboard accessibility	Manual/runtime verification

This helps reduce false positives and improves developer trust.

Do not treat a finding as a definite WCAG failure if the scanner cannot establish the relevant context.

Also, keep the rule definitions versioned against the WCAG criteria they actually test.

7. Make native VS Code Problems integration a priority

Your existing HTML report is useful, but developers should not have to open it for every finding.

Implement DiagnosticCollection.

The flow should be:

Java scanner
     |
     v
Structured findings
     |
     v
TypeScript Diagnostic Provider
     |
     v
VS Code Problems panel
     |
     v
Click finding
     |
     v
Navigate to source line

Example:

const diagnostics =
    vscode.languages.createDiagnosticCollection("ada");
function publishFinding(
    finding: Finding
): void {
    const uri = vscode.Uri.file(finding.file);
    const range = new vscode.Range(
        finding.line - 1,
        finding.column - 1,
        finding.line - 1,
        finding.column
    );
    const diagnostic = new vscode.Diagnostic(
        range,
        finding.message,
        vscode.DiagnosticSeverity.Warning
    );
    diagnostic.code = finding.ruleId;
    diagnostic.source = "ADA Scanner";
    diagnostics.set(uri, [
        ...diagnostics.get(uri) ?? [],
        diagnostic
    ]);
}

This is illustrative; your implementation should also validate source ranges, convert project-relative paths to absolute paths and avoid duplicate diagnostics.

My recommendation is to retain both:

* Native Problems panel for day-to-day development.
* HTML report for comprehensive analysis and sharing.

8. Packaging and release strategy

For your current Windows-only implementation, I recommend this build pipeline:

Build Java scanner

Compile · Test · Package executable JAR

Generate custom JRE

jdeps · jlink · Windows x64

Build VS Code extension

TypeScript compilation · Production dependencies

Package Windows VSIX

Scanner JAR + Custom runtime + Extension assets

Release and validation

Clean Windows VM · Offline test · Installation test

Recommended release checks

Test	Expected result
Windows machine without Java	Scanner works
No internet connection	Static scanning works
Workspace path containing spaces	Scanner works
Large repository	No excessive memory consumption
Cancel scan	Java process terminates cleanly
Invalid Java runtime	Clear error message
Missing scanner JAR	Clear diagnostic
Upgrade extension	Existing settings preserved

For enterprise deployment, also consider code-signing your VSIX and publishing checksums for the release artifacts.

One limitation to document: a Windows-specific extension with an embedded Windows runtime does not automatically support execution inside Linux-based WSL, remote SSH or Linux Dev Containers. Those require a runtime matching the extension host platform.

9. Features I would defer

To keep the implementation manageable, I would not introduce these immediately.

Feature	Priority
AI-generated code remediation	Later
GraalVM Native Image migration	Later
Runtime download mechanism	Not needed initially
Cloud-based scanning	Not needed
Multi-platform runtime	Not needed
Playwright dynamic scanning	Phase 4
SARIF export	Phase 3

Your existing Java scanner and bundled runtime are a perfectly reasonable foundation.

10. Exact implementation priority for your existing codebase

Here is the order I would give Claude Code to work through your existing implementation.

Priority	Change	Expected outcome
P0	Analyze current 43 MB VSIX	Identify size contributors
P0	Optimize jlink runtime	Smaller offline distribution
P0	Extract runtime resolution logic	Maintainable runtime management
P1	Separate scanner process management	Reliable Java invocation
P1	Introduce JSON protocol	Decoupled scanner communication
P1	Improve error handling and cancellation	Production reliability
P2	Native VS Code diagnostics	Better developer experience
P2	Centralized rule registry	Easier scanner maintenance
P2	Incremental scanning	Better performance
P3	SARIF and CI/CD integration	Enterprise adoption
P3	Dynamic accessibility testing	Expanded coverage

11. Prompt for Claude Code

Since Claude Code has already developed your extension, I would first ask it to inspect the existing implementation and propose changes rather than immediately rewriting files.

Task: Improve Existing ADA VS Code Extension Architecture and Packaging

You have already developed the ADA VS Code extension. The extension is currently working as expected.

I want to improve its architecture, packaging, performance, reliability and maintainability without breaking existing functionality.

Current implementation

* VS Code extension written in TypeScript.
* Accessibility scanner implemented as an executable Java JAR.
* Custom Java runtime generated using jlink.
* Windows-only distribution.
* Current VSIX size approximately 43 MB.
* Static accessibility scanning.
* WCAG 2.1 success criteria evaluation.
* Automated and manual verification classification.
* HTML scan report generation.
* VS Code commands for repository and file scanning.
* Existing scan results and report functionality.

Target architecture

Maintain this architecture:

1. TypeScript VS Code extension.
2. Independent Java accessibility scanner.
3. Embedded custom Windows x64 jlink runtime.
4. Structured JSON communication between extension and scanner.
5. Common accessibility findings model.
6. Native VS Code diagnostics and HTML reporting.

The Java scanner must remain independently executable without requiring VS Code.

Implementation requirements

1. Analyze current packaging

Inspect the existing VSIX and identify:

* JRE size.
* Scanner JAR size.
* Third-party dependency sizes.
* Node.js production dependencies.
* Unnecessary packaged files.

Recommend specific optimizations.

Do not remove dependencies without validating scanner functionality.

2. Optimize Java runtime

Inspect the existing jlink configuration.

* Identify required Java modules.
* Use jdeps where applicable.
* Remove unnecessary modules.
* Preserve all scanner dependencies.
* Generate a Windows x64 runtime.
* Ensure the runtime works without system-installed Java.

Do not replace the Java scanner with another technology.

3. Introduce runtime resolution

Create a dedicated runtime resolver.

* Support configured Java executable.
* Default to the bundled runtime.
* Validate runtime availability.
* Validate Java version compatibility.
* Provide meaningful error messages.

4. Improve scanner process management

Introduce a dedicated scanner client/process manager.

* Use child_process.spawn.
* Avoid shell execution.
* Support cancellation.
* Support execution timeouts.
* Capture standard error separately.
* Handle unexpected process termination.
* Handle invalid scanner responses.
* Prevent concurrent scan conflicts where necessary.

5. Introduce structured scanner communication

Evaluate the existing scanner interface.

Where practical, implement versioned JSON Lines communication.

Support:

* Scan requests.
* Progress notifications.
* Accessibility findings.
* Scan completion.
* Scanner errors.
* Cancellation.

Do not mix application logs with protocol output.

6. Improve VS Code integration

Use the native VS Code DiagnosticCollection API.

Support:

* Exact file navigation.
* Line and column mapping.
* Severity mapping.
* Rule ID display.
* Diagnostic cleanup.
* Quick Fix integration where reliable.

Retain the existing HTML report.

7. Preserve scanner functionality

Do not unnecessarily redesign the accessibility rules.

Gradually improve:

* Rule organization.
* Rule metadata.
* WCAG mapping.
* Confidence classification.
* False-positive handling.
* Unit testing.

Do not classify unverified manual criteria as passed.

8. Enterprise readiness

Ensure:

* Offline operation.
* No mandatory external service dependencies.
* Secure process execution.
* Safe HTML report rendering.
* Workspace-level configuration.
* Clear logging.
* Stable scanner and protocol versioning.

Critical implementation instructions

1. First inspect the entire existing project.
2. Identify what is already implemented.
3. Do not duplicate existing functionality.
4. Do not rewrite working modules unnecessarily.
5. Provide the proposed architecture and file-level changes before implementation.
6. Identify backward compatibility risks.
7. Implement changes incrementally.
8. Run existing tests after each significant change.
9. Verify the packaged VSIX on a Windows machine without Java installed.
10. Report the before-and-after VSIX size.

Expected deliverables

* Current architecture assessment.
* Recommended target architecture.
* VSIX size breakdown.
* Proposed file-level changes.
* Optimized jlink configuration.
* Updated runtime resolution.
* Reliable scanner process management.
* Structured scanner communication.
* Native VS Code diagnostics.
* Build and packaging instructions.
* Test results and final size comparison.

Do not introduce GraalVM, runtime downloads, cloud scanning or unnecessary architectural complexity at this stage.

Primary objective: Improve the existing working extension incrementally, retain offline functionality, reduce distribution size where practical, and make the architecture suitable for enterprise deployment.

My overall assessment

Recommended direction

I would retain your current technology stack.

Component	Decision
VS Code extension	TypeScript
Scanner	Java executable JAR
Runtime	Custom jlink
Platform	Windows x64
Communication	JSON Lines
Editor integration	VS Code Diagnostics API
Report	Existing HTML report
Future integration	SARIF + CI/CD

One final architectural recommendation: Don’t try to make the Java scanner smarter by putting more responsibilities into the VS Code extension. Keep accessibility analysis, rule evaluation and finding generation inside Java. Keep editor integration, process management and user experience inside TypeScript.

That separation will give you the most maintainable foundation for the next several versions of your ADA extension.


Copy this prompt into your existing Claude Code session where your ADA VS Code extension is already developed.

Improve Existing ADA VS Code Extension – Architecture, Packaging & Performance

You have already developed my ADA VS Code extension, and its current functionality is working as expected.

I want you to improve the existing implementation incrementally. Do not rewrite the project or break existing functionality.

Current Architecture

* VS Code extension: TypeScript
* Accessibility scanner: Executable Java JAR
* Java runtime: Custom jlink runtime
* Target platform: Windows x64 only
* Current VSIX size: Approximately 43 MB
* Static accessibility scanning
* WCAG 2.1 success criteria validation
* Automated and manual verification classification
* HTML accessibility scan report
* Repository-level and file-level scanning
* Existing VS Code commands and interactive report functionality

Objectives

1. Optimize Extension Packaging

* Analyze the existing 43 MB VSIX.
* Identify the exact size of the JRE, scanner JAR, dependencies and extension assets.
* Optimize the existing jlink runtime.
* Use jdeps to identify required Java modules where applicable.
* Remove unnecessary Java modules and development dependencies.
* Retain Windows x64 offline functionality.
* Do not introduce runtime downloads or external dependencies.

2. Improve Architecture

Maintain the following separation:

TypeScript Extension

* VS Code commands
* UI and Webview
* Diagnostics
* Report presentation
* Runtime resolution
* Scanner process management

Java Scanner

* Accessibility analysis
* WCAG rule evaluation
* Framework-specific scanning
* Finding generation
* Scan summary

The Java scanner must remain independently executable without VS Code.

3. Improve Scanner Communication

Evaluate the existing communication mechanism.

If appropriate, introduce a versioned JSON Lines protocol between TypeScript and Java.

Support:

* Scan requests
* Progress notifications
* Findings
* Completion events
* Error responses
* Cancellation

Use child_process.spawn rather than shell execution.

Keep protocol output on stdout and application logs on stderr.

4. Runtime Management

Introduce or improve a dedicated runtime resolver.

Priority:

1. Explicitly configured Java executable.
2. Bundled custom jlink runtime.

Validate runtime availability and Java version compatibility.

Provide meaningful error messages.

5. Improve VS Code Integration

Use the native DiagnosticCollection API.

Support:

* Problems panel integration
* Exact source file navigation
* Line and column mapping
* Severity mapping
* WCAG rule identification
* Diagnostic cleanup
* Quick Fix suggestions where reliable

Retain the existing HTML reporting functionality.

6. Improve Scanner Maintainability

Gradually introduce:

* Centralized rule registry
* Consistent rule metadata
* WCAG mapping
* Severity and confidence classification
* Better false-positive handling
* Rule-level automated tests

Do not unnecessarily rewrite existing accessibility rules.

Never classify an unverified manual criterion as passed.

7. Enterprise Reliability

Ensure:

* Offline scanning
* Process cancellation
* Timeout handling
* Crash recovery
* Safe HTML report rendering
* Workspace-level configuration
* Consistent error handling
* No source-code upload to external services

8. Backward Compatibility

All existing functionality must continue working.

Preserve:

* Existing scan commands
* Existing accessibility rules
* Existing HTML report
* Existing scan summaries
* Existing report interactions
* Existing user configuration wherever possible

Implementation Approach

Work incrementally.

Step 1: Inspect the complete existing project and understand its current architecture.

Step 2: Provide an architecture assessment and exact VSIX size breakdown.

Step 3: Identify existing implementations that can be reused.

Step 4: Propose specific file-level changes and explain why each change is required.

Step 5: Implement changes in small, independently testable stages.

Step 6: Run existing tests and add regression tests.

Step 7: Build and package the Windows x64 VSIX.

Step 8: Verify offline installation on a clean Windows machine without system-installed Java.

Step 9: Provide before-and-after package sizes and implementation results.

Important Constraints

* Do not rewrite the existing project.
* Do not replace Java with another technology.
* Do not introduce Spring Boot if it is not already required.
* Do not introduce GraalVM.
* Do not introduce cloud dependencies.
* Do not make internet connectivity mandatory.
* Do not change working scanner behavior without justification.
* Do not remove functionality to reduce package size.
* Do not make speculative changes without inspecting the existing code.
* Prefer simple, maintainable enterprise-grade solutions.

Expected Final Deliverables

1. Existing versus proposed architecture.
2. VSIX size breakdown and optimization results.
3. Updated jlink configuration.
4. Runtime management implementation.
5. Scanner process management improvements.
6. Communication protocol improvements, if justified.
7. Native VS Code diagnostics integration.
8. Test and regression results.
9. Final Windows VSIX package.
10. Before-and-after comparison.

Start with Step 1: Inspect the existing project and report your findings. Do not modify any files until you have presented the architecture assessment and proposed implementation plan.

My suggestion: Give Claude this prompt first in Plan Mode. Review its assessment and proposed file changes before allowing implementation. This minimizes the risk of breaking your currently working extension.




