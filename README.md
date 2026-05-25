# ai-rules

A collection of `AGENTS.md` and `copilot-instructions.md` templates for various stacks.

Drop the relevant files into your project and adapt to fit your conventions.

---

## What's in here

Each stack folder contains up to three files:

| File                      | Tool           | Purpose                                            |
| ------------------------- | -------------- | -------------------------------------------------- |
| `AGENTS.md`               | Claude, Codex  | Agentic rules, STOP gates, architectural contracts |
| `copilot-instructions.md` | GitHub Copilot | Inline generation: tests, JSDoc, DTOs              |

---

## Stacks

```
/
├── nextjs/
│   ├── AGENTS.md
│   ├── copilot-instructions.md
│   └── AI-TOOLS.md
├── ...
```

> More stacks added over time. Check the folder list for what's available.

---

## How to use

1. Find the folder that matches your stack.
2. Copy the files you need into your project root:
   - `AGENTS.md` → repo root
   - `copilot-instructions.md` → `.github/copilot-instructions.md`
3. Review every rule before committing. These are starting points, not drop-in defaults.
4. `AI-TOOLS.md` → This is a guide to which AI tool to use

---

## Before you commit these files

- **STOP gates** — `AGENTS.md` contains hard interrupt rules for agentic runners. Read them. Confirm your team understands that approval is required at each gate before an agent proceeds.
- **Stack-specific assumptions** — each template is opinionated (e.g. the Next.js template assumes App Router, Zod, TanStack Query, RTK). Remove or replace rules that don't match your setup.
- **Copilot instructions** — the `.github/copilot-instructions.md` path is required for Copilot to pick up workspace rules. Any other path is ignored.

---

## Contributing

Adding a new stack:

1. Create a folder named after the stack.
2. Include at minimum an `AGENTS.md`. Add `copilot-instructions.md` if the stack has clear inline generation conventions.
3. Keep rules grounded in the stack's actual defaults and tooling — avoid generic advice that applies everywhere.
4. Open a PR with a one-line summary of what the stack covers and what's intentionally omitted.
