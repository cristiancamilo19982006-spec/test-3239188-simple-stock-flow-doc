# Git Conventions

> **Read this document before making your first commit on the project.**

## Branch strategy
main        ← Production. Merge from release only. Always stable.
└── dev   ← Continuous integration. Merge from features.
└── feat/[description]    ← One branch per feature/user story
└── fix/[description]     ← One branch per bugfix
└── chore/[description]   ← Infrastructure, docs, dependency changes
└── hotfix/[description]  ← Urgent fixes directly to main

**Rules:**
- Nobody commits directly to `main` or `dev`
- Every task = one branch + one Pull Request
- One branch = one task (do not mix different features)
- Branches are deleted after merge

---

## Branch naming format
[type]/[description-in-kebab-case]

Examples:
feat/jwt-authentication
fix/pqr-pdf-generation-error
chore/update-dependencies
hotfix/null-token-expiration
docs/00-governance-update

---

## Commit format (Conventional Commits)
type: [lowercase description, imperative mood, no trailing period]

[optional body — explain WHY, not what]

[optional footer — issue/user story references]

**Types:**
| Type | When to use |
|------|-------------|
| `feat` | New functionality |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `style` | Formatting, whitespace (no logic change) |
| `refactor` | Code refactoring without behavior change |
| `test` | Add or modify tests |
| `chore` | Tooling, dependencies, CI |
| `perf` | Performance improvement |

**Examples:**
feat(auth): implement JWT login

fix(pqr): correct PDF template layout rendering
Closes #12

docs(governance): update agile conventions and security policy

chore(deps): upgrade Node.js dependencies

---

## Pull Request policy

- **Size:** maximum 400 lines of code (excluding tests). If larger, split it.
- **Reviewers:** minimum 1 approval before merging
- **Review time:** reviewer has a maximum of 24 business hours
- **Template:** use the template at `.github/pull_request_template.md`
- **Green CI:** merge only proceeds if all pipeline checks pass

---

## Merge policy

- Use **Squash and Merge** for features (keeps `dev` history clean)
- Use **Merge Commit** for releases to `main` (preserves full history)
- **Do not** use Rebase & Merge (creates confusion in shared history)

---

## Tags and versioning

Follow [SemVer](https://semver.org/): `MAJOR.MINOR.PATCH`

```bash
# When releasing to production
git tag -a v1.0.0 -m "Release v1.0.0: initial lexia MVP release"
git push origin v1.0.0