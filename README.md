# Dev Push

Dev Push is an AI skill for safely deploying selected, already-committed frontend and backend changes to development branches with Codex.

It focuses on deployment safety: selecting only intended commits, isolating local work, checking conflicts and database dependencies, validating changes, protecting against remote races, verifying CI/deployment state, and preserving rollback information.

## Repository structure

```text
dev-push/
├── SKILL.md
├── agents/
│   └── openai.yaml
└── assets/
    └── icon.svg
```

## Installation

Copy the `dev-push` folder into your Codex skills directory, then customize the placeholders in `SKILL.md` for your repositories and branch names.

The skill uses placeholders such as:

- `<frontend-repo>`
- `<backend-repo>`
- `<frontend-dev-branch>`
- `<backend-dev-branch>`
- `<frontend-feature-base>`
- `<backend-feature-base>`
- `<frontend-port>`
- `<backend-port>`

## License

MIT
