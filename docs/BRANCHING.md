# Branching Strategy

## Branches

| Branch | Purpose | Who pushes |
|---|---|---|
| **nexus** | Active development. All PRs target here. | Everyone |
| **apex** | Stable releases. Tagged versions cut from here. | Maintainer via PR from nexus |

## Flow

```
feature branch → PR → nexus (review + merge) → PR → apex (release)
                                                       ↓
                                                  v0.1.0, v0.2.0, ...
```

## Rules

- **nexus**: PRs require review. CI must pass.
- **apex**: PRs from nexus only. CI guard rejects any `.private/` files. Tagged releases trigger publish.
- **Feature branches**: named `feat/<description>`, `fix/<description>`, or `chore/<description>`.
