# claude-roles

Shared Claude agent role definitions used across all personal projects.

## Roles

| File | Role | Purpose |
|------|------|---------|
| `tasker.md` | Orchestrator | Breaks down work, dispatches agents, manages review cycles |
| `coder.md` | Implementation Agent | Writes tested code to a spec |
| `reviewer.md` | Code Reviewer | 8-dimension quality review with pass/fail critical gates |
| `security-linter.md` | Security Auditor | Focused SQL injection / PII / integer overflow audit |

## Usage

Add to any project as a git submodule:

```bash
git submodule add https://github.com/andrewcostello/claude-roles.git .claude/roles
```

Reference from your project's `CLAUDE.md`:

```markdown
For role-specific instructions, see:
- `.claude/roles/tasker.md` — Orchestration agent
- `.claude/roles/coder.md` — Implementation agent
- `.claude/roles/reviewer.md` — Review agent
- `.claude/roles/security-linter.md` — Security audit agent
```

## Updating

```bash
# In any project using this submodule
git submodule update --remote .claude/roles
git add .claude/roles
git commit -m "chore: update claude-roles submodule"
```
