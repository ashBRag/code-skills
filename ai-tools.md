# AI Tools — When to Use What

This document describes how we use AI tooling in this codebase, which tool owns which task, and where each tool's configuration lives.

## Tool Overview

| Tool               | Role                                                        | Config                            |
| ------------------ | ----------------------------------------------------------- | --------------------------------- |
| **Claude / Codex** | Agentic tasks — multi-file changes, refactors, architecture | `AGENTS.md`                       |
| **GitHub Copilot** | Inline generation — tests, JSDoc, DTOs                      | `.github/copilot-instructions.md` |

## Decision Guide

```
Is the task contained to one file and the output shape already known?
├── Yes → Copilot
└── No
    ├── Does it touch auth, session, or cross-cutting architecture?
    │   └── Claude / Codex + expect a STOP gate for security review
    └── Is it a feature, refactor, or multi-file change?
        └── Claude / Codex
```

---

## Config File Index

| File                              | Read by        | Purpose                                              |
| --------------------------------- | -------------- | ---------------------------------------------------- |
| `AGENTS.md`                       | Claude, Codex  | Agentic rules, STOP gates, architectural contracts   |
| `.github/copilot-instructions.md` | GitHub Copilot | JSDoc style, DTO conventions, test structure         |
| `env.ts`                          | Runtime        | Single source of truth for all environment variables |

---

## Onboarding Notes

- Read `AGENTS.md` before using Claude or Codex on this codebase. The STOP gate rules are enforced — do not approve past a gate without understanding what you're confirming.
- Copilot suggestions for tests and DTOs should still be reviewed against the contracts in `AGENTS.md` (AppError shape, Server Action return shape, what to mock).
- Neither tool replaces code review. AI-generated code follows the same review standards as human-authored code.
