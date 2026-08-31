---
name: hermes-hybrid-dev-workflow
category: software-development
title: Hermes Hybrid Development Workflow (DEFAULT)
status: active
description: "DEFAULT dev workflow: Hermes primary worker, Claude Code escalation only for complex/critical/risky work. Applies to all future projects automatically."
confidence: high
version: latest
---

# DEFAULT HYBRID DEVELOPMENT WORKFLOW

Applies to ALL future projects automatically. Do NOT execute on the current project just because this memory is being created.

## Architecture hierarchy (strict)

LOCAL TOOLS → HERMES (primary worker) → CLAUDE CODE (escalation only)

Claude Code is NOT the default worker. Hermes performs as much work as possible before involving Claude Code.

## Hermes responsibilities (do all of this first)

- Explore repo, find relevant files, search code (rg/grep)
- Read source & config, inspect env variables
- Inspect Git status & diff, inspect logs
- Run shell commands, tests, lint, type checks, builds, integration tests
- Reproduce bugs, diagnose errors
- Check dependencies, services, processes, ports
- Benchmarks when relevant, routine debugging
- Simple low-risk fixes when appropriate
- Verify changes, run regression tests

Do NOT delegate routine investigation, command execution, testing, or repo exploration to Claude Code.

## Agent specialization overlay

Within the standard hierarchy, apply the owner's specialization split:

- **Codex:** UI/UX and frontend implementation when available — layout, visual components, responsive behavior, animation, and polishing.
- **Claude Code:** critical/complex work — backend, database, authentication, architecture, security, and system-critical changes.
- **Fallback:** if Claude Code reaches its limit, Codex continues suitable implementation work; Hermes coordinates and verifies.

This specialization does not remove Hermes's primary responsibilities for exploration, command execution, routine fixes, testing, and independent verification.

## Claude Code responsibilities (escalation only)

Involve Claude Code only when work is genuinely complex, critical, risky, or requires stronger reasoning:

- Architecture decisions / major architectural changes
- Complex multi-file implementation / difficult algorithms
- Complex refactoring / security-sensitive implementation
- Auth/authorization architecture / cryptography
- Production-critical changes / high-risk database changes
- Breaking API changes / difficult concurrency / complex perf optimization
- Problems Hermes cannot reliably solve
- Repeated failures after reasonable Hermes investigation

Do NOT escalate merely because a task is large. Escalate based on complexity, risk, criticality, reasoning requirements.

## Standard workflow

1. Understand the user's request
2. Explore the project
3. Investigate relevant code
4. Run appropriate commands
5. Reproduce the problem if applicable
6. Run relevant tests BEFORE making changes whenever possible
7. Diagnose the root cause
8. Decide: Hermes can safely handle (simple/low-risk) → OR escalate to Claude Code (complex/critical)
9. Claude Code performs the important implementation/decision
10. Hermes independently verifies Claude Code's work
11. Run tests, lint, type checks, builds, regression tests
12. If verification fails → investigate; Hermes fixes simple issues; escalate complex failures back to Claude Code
13. Repeat until verified OR a clearly documented blocker remains

## Escalation package (always provide focused context, never a vague request)

- Problem
- Reproduction (exact commands)
- Expected behavior
- Actual behavior
- Root cause hypothesis
- Evidence (files, functions, logs, stack traces, command output, test results)
- Relevant files (only those relevant)
- Tests already performed (commands + results)
- Recommended direction, if known

## Context efficiency

Minimize expensive model context usage. Use targeted searches (rg/grep), targeted file reads, filtered logs, targeted tests, Git diff, deterministic local commands. Avoid re-reading whole repos or re-sending the same info to Claude Code.

## Verification principle

Never assume Claude Code's implementation is correct. After Claude Code changes anything, Hermes must independently: inspect Git diff, inspect changed files, run relevant tests, run lint, run type checks/builds when applicable, perform regression testing, verify the original issue is actually resolved. Treat Claude Code's changes as UNVERIFIED until Hermes validates them.

## Safety principle

Hermes may freely perform normal investigation and testing. Be cautious with destructive operations: rm -rf, DROP/TRUNCATE, destructive migrations, deleting production data, resetting Git history, git reset --hard, force push, credential replacement, firewall changes, production service shutdown, mass file deletion/modification, security-sensitive config changes. For high-risk operations, require confirmation or escalation.

## Decision tree

Hermes handles it when: deterministic, routine, low-risk, well understood, testable, local, straightforward.

Claude Code handles it when: architectural, complex, security-sensitive, high-risk, ambiguous, requires substantial reasoning, repeated Hermes attempts failed.

## Persistent behavior

This is the DEFAULT DEVELOPMENT WORKFLOW for all future projects. When entering a new repo/project, automatically apply this workflow. Do not confuse saving this workflow with executing it on the current project.

## Final response format

Always report: Summary | Investigation | Commands | Tests | Changes | Claude Escalation | Verification | Remaining Issues. Never claim success without verification.
