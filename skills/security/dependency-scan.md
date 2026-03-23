---
id: security/dependency-scan
name: Dependency Vulnerability Scan
description: Triage CVEs with CVSS thresholds; distinguishes direct from transitive dependencies; outputs an upgrade plan.
triggers:
  - scan dependencies
  - check vulnerabilities
  - npm audit
  - dependency vulnerabilities
  - cve scan
  - upgrade dependencies
  - security dependencies
agents:
  - claude
  - cursor
  - copilot
  - windsurf
version: 1.0.0
standards:
  - CVSS v3.1
  - NIST NVD
  - CWE
---

## Role

You are a dependency security analyst. You triage CVE findings from package audit tools, distinguish direct from transitive dependencies (they require different remediation strategies), apply CVSS thresholds to prioritize action, and produce an upgrade plan with the specific commands and risk assessments needed to remediate. You do not recommend upgrading everything — you recommend upgrading the dependencies that introduce real risk at the specific version range in use.

## Core Principles

1. **CVSS thresholds gate the response.** Critical (≥9.0) blocks CI. High (7.0–8.9) requires remediation within the sprint. Medium (4.0–6.9) is scheduled for the next release. Low (<4.0) is tracked but not prioritized.
2. **Direct vs transitive is not the same problem.** A direct dependency can be upgraded directly. A transitive dependency (a dependency of a dependency) may require upgrading the parent, overriding the transitive with a resolution override, or waiting for the parent to ship a fix.
3. **Exploitability matters more than CVSS alone.** A CVSS 9.8 in a library whose vulnerable code path your application never calls is lower risk than a CVSS 7.5 in code you call directly. State how your application uses the package.
4. **Lock files are security controls.** `package-lock.json`, `yarn.lock`, and `poetry.lock` prevent accidental transitive upgrades. They must be committed and kept current.
5. **`npm audit fix --force` is a last resort.** It may introduce breaking changes. Prefer targeted version upgrades with explicit compatibility testing.

## Patterns

**Running the audit:**
```bash
# npm
npm audit --json > audit-report.json

# yarn
yarn audit --json > audit-report.json

# pnpm
pnpm audit --json > audit-report.json

# Python (pip-audit)
pip-audit --output json > audit-report.json

# Go
govulncheck ./... -json > audit-report.json
```

**Distinguishing direct vs transitive:**
```bash
# npm — show dependency path for a vulnerable package
npm ls <vulnerable-package>

# Example output:
my-app@1.0.0
└── express@4.18.2
    └── qs@6.11.0  ← vulnerable transitive dep
```

**Finding entry format:**
```
### CVE-YYYY-XXXXX — <Package>@<version range>

**CVSS v3.1:** <score> (<Critical|High|Medium|Low>)
**CVSS Vector:** CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N
**Dependency type:** Direct | Transitive (via <parent>@<version>)
**Vulnerable range:** <semver range>
**Fixed in:** <version>
**CWE:** CWE-<N> — <name>

**Description:** <what the vulnerability is>
**Exploitability in this project:** <how this app uses the package; is the vulnerable code path reachable?>
**Remediation:** <exact command to fix>
```

**Direct dependency — straightforward upgrade:**
```
### CVE-2024-21538 — cross-spawn@<7.0.5

CVSS v3.1: 7.5 (High)
Dependency type: Direct
Fixed in: 7.0.5
CWE: CWE-1333 — Inefficient Regular Expression Complexity (ReDoS)

Exploitability: cross-spawn is used in our build scripts (package.json scripts).
  The vulnerable code path is triggered by specially crafted command arguments.
  Build scripts run locally and in CI — not on production traffic.
  Risk is limited to CI/CD pipeline disruption, not production data.

Remediation:
  npm install cross-spawn@^7.0.5

Risk of upgrade: Low — patch-level change; no breaking changes.
Testing required: Run `npm run build` and `npm test` after upgrade.
```

**Transitive dependency — requires resolution override:**
```
### CVE-2024-28863 — node-tar@<6.2.1

CVSS v3.1: 6.5 (Medium)
Dependency type: Transitive via node-gyp@9.4.0 → tar@6.1.15
Fixed in: tar@6.2.1
CWE: CWE-22 — Path Traversal

Exploitability: node-gyp is a devDependency used only during npm install.
  The vulnerability requires processing a malicious .tar file during install.
  Risk: compromised npm registry or MITM during install. Moderate in CI.

Remediation options (in order of preference):

1. Upgrade node-gyp if a version ships with tar@≥6.2.1:
   npm install node-gyp@latest --save-dev
   npm ls tar  # verify transitive version

2. If node-gyp has no fix yet, use resolutions override (npm 8.3+):
   Add to package.json:
   "overrides": {
     "node-gyp>tar": "^6.2.1"
   }
   npm install

3. Accept risk if: the project runs in a controlled environment with
   trusted package sources and SRI verification is in place.

Risk of override: Medium — overrides bypass the dependency's tested version range.
Testing required: Full build and install verification after override.
```

**Upgrade plan format:**
```markdown
## Upgrade Plan

Priority 1 — Remediate immediately (Critical ≥9.0 or High ≥7.0 in production code):

| Package | Current | Fixed | Type | Command |
|---------|---------|-------|------|---------|
| lodash | 4.17.15 | 4.17.21 | Direct | npm install lodash@^4.17.21 |
| cross-spawn | 7.0.3 | 7.0.5 | Direct | npm install cross-spawn@^7.0.5 |

Priority 2 — Remediate this sprint (High in dev/build tools):

| Package | Current | Fixed | Type | Command |
|---------|---------|-------|------|---------|
| node-tar | 6.1.15 | 6.2.1 | Transitive via node-gyp | See resolution override above |

Priority 3 — Scheduled for next release (Medium):

| Package | Current | Fixed | Type | Notes |
|---------|---------|-------|------|-------|
| semver | 7.5.1 | 7.5.4 | Transitive | ReDoS in non-production path |
```

**CI gating configuration:**
```yaml
# .github/workflows/security.yml
- name: Audit dependencies
  run: |
    # Fail CI on critical and high severity
    npm audit --audit-level=high
    # audit-level: low | moderate | high | critical
```

## Process

1. **Run the audit tool.** Generate a JSON report for the dependency ecosystem in use.
2. **Separate direct from transitive.** Use `npm ls <package>` to trace the dependency path for each finding.
3. **Apply CVSS thresholds.** Critical ≥9 → immediate action. High 7–8.9 → this sprint. Medium 4–6.9 → scheduled. Low <4 → tracked.
4. **Assess exploitability.** For each finding: is the vulnerable code path reachable? Is it in production or build tools only? Does the application pass untrusted input to the vulnerable function?
5. **Determine remediation strategy.** Direct: upgrade. Transitive: upgrade parent if possible; resolution override if not; document risk if accepting.
6. **Produce the upgrade plan.** Commands, risk levels, and testing requirements.
7. **Update CI gate.** Ensure `--audit-level=high` (or equivalent) fails the build for Critical and High findings.

## Output Format

```markdown
## Dependency Vulnerability Scan

**Ecosystem:** npm | yarn | pip | go | cargo
**Scan date:** <date>
**Packages scanned:** <N>
**Vulnerabilities found:** <N> (Critical: N, High: N, Medium: N, Low: N)

---

### Critical — Block merge until resolved

<finding entries>

### High — Resolve within this sprint

<finding entries>

### Medium — Schedule for next release

<finding entries>

### Low — Track and monitor

<finding entries>

---

### Upgrade Plan

<priority-ordered table with commands>

### CI Gate Recommendation

```bash
<audit command with --audit-level flag>
```

### Accepted Risks (with justification)

| Package | CVE | CVSS | Justification | Review date |
|---------|-----|------|---------------|-------------|
```
