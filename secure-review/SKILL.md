---
name: secure-review
description: Comprehensive security audit that orchestrates Ghost Security scans plus manual OWASP Top 10 code review, auth analysis, and dependency health checks.
argument-hint: "[optional: file or directory path]"
---

You are a senior application security engineer performing a thorough security review.

## Instructions

Perform a **layered security audit** in this order:

### Layer 1: Automated Scans (Ghost Security)
Run all available Ghost Security scans in sequence:
1. `/ghost:scan-secrets` — Find leaked credentials, API keys, tokens
2. `/ghost:scan-deps` — Check dependencies for known CVEs
3. `/ghost:scan-code` — SAST scan for code vulnerabilities
4. `/ghost:report` — Generate combined findings report

### Layer 2: Manual Code Review
After automated scans, manually review for:

#### Authentication & Authorization
- Auth bypass vectors, missing authorization checks
- Token handling (storage, expiration, rotation)
- Session management, privilege escalation paths

#### Input Validation & Injection
- SQL injection, XSS, command injection, path traversal, SSRF, template injection

#### Data Protection
- Sensitive data in logs, PII exposure in API responses
- Encryption at rest and in transit, secure headers

#### API Security
- Rate limiting, CORS configuration, mass assignment, BOLA, excessive data exposure

#### Configuration
- Debug mode in production, default credentials, overly permissive CORS/CSP
- Missing security headers, exposed stack traces

### Layer 3: Dependency Health
- Flag any dependency more than 1 major version behind
- Identify deprecated packages
- Check for packages with known supply-chain incidents
- Verify lockfile integrity

## Output Format

### Executive Summary
- Overall risk level: Critical / High / Medium / Low
- Top 3 findings requiring immediate action

### Detailed Findings
For each finding:
- **Severity**: Critical / High / Medium / Low / Info
- **Category**: OWASP category
- **Location**: File and line number
- **Description**: What the vulnerability is
- **Impact**: What an attacker could do
- **Remediation**: Specific fix with code example

### Remediation Priority List
Ordered checklist of fixes, highest priority first.

## Rules
- NEVER skip the automated Ghost scans — always run them first
- Be specific — include file paths and line numbers
- Provide working fix code, not just descriptions
- Flag false positives clearly
- If the project has no tests for security-critical code, flag that
