# Bitbucket CLI Commands

Bitbucket CLI (`bkt`) commands for the CodeRabbit Autofix skill.

## Prerequisites

**Bitbucket CLI (`bkt`):**
- Install: `brew install avivsinai/tap/bitbucket-cli`
- Authenticate: `bkt auth login`
- Verify: `bkt auth status`

Ensure an active context is configured (`bkt context list`) with the correct host, project/workspace, and repo.

## Platform Notes

- **Bitbucket Cloud:** `bkt pr comments --state unresolved` filters server-side.
- **Bitbucket Data Center:** The DC API does not expose comment resolution status. Fetch all comments and filter client-side.
- **Reactions:** `bkt pr reaction` is DC only; not available on Cloud.

## Core Operations

### 1. Find Pull Request

`bkt pr list` does not support filtering by source branch directly. List open PRs as JSON and filter with `--jq`:

```bash
bkt pr list --state OPEN --json \
  --jq "[.[] | select(.source.branch.name == \"$(git branch --show-current)\")] | .[0] | {id, title}"
```

### 2. Fetch Unresolved Comments

**Bitbucket Cloud:**

```bash
bkt pr comments <id> --state unresolved --json
```

**Bitbucket Data Center:**

DC does not expose resolution status. Fetch all comments and filter client-side:

```bash
bkt pr comments <id> --json
```

Filter criteria:
- Comment author is one of: `coderabbitai`, `coderabbit[bot]`, `coderabbitai[bot]`

On Cloud, the `--state unresolved` flag handles resolution filtering server-side. On DC, exclude comments that have been resolved by other means (e.g. task completion).

### 3. Post Summary Comment

```bash
bkt pr comment <pr-id> --text "$(cat <<'EOF'
## Fixes Applied Successfully

Fixed <file-count> file(s) based on <issue-count> unresolved review comment(s).

**Files modified:**
- `path/to/file-a.ts`
- `path/to/file-b.ts`

**Commit:** `<commit-sha>`

The latest autofix changes are on the `<branch-name>` branch.

EOF
)"
```

Post after the push step (if pushing) so branch state is final.

### 4. Acknowledge Review (DC Only)

```bash
bkt pr reaction add <pr-id> <comment-id> --emoji thumbsup
```

This is only available on Data Center. On Cloud, skip this step.

### 5. Create PR (if needed)

```bash
bkt pr create --title '<title>' --source $(git branch --show-current) --target main
```

## Error Handling

**Missing `bkt` CLI:**
- Inform user and provide install instructions (`brew install avivsinai/tap/bitbucket-cli`)
- Exit skill

**No active context:**
- Run `bkt context list` to check
- If none, prompt user to create one with `bkt context create`
- Exit skill

**API failures:**
- Log error and continue
- Don't abort for comment posting failures

**Getting repo info:**
```bash
bkt repo view --json
```
