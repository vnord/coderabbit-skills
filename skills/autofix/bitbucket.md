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

## Response Envelopes

`bkt` wraps JSON list responses, so `--jq ".[]"` will fail. Always unwrap:

- `bkt pr list --json` → `{pull_requests: [...], repo, workspace}` — use `.pull_requests[]`
- `bkt pr comments --json` → `{comments: [...]}` — use `.comments[]`

Prefer piping to `jq` over `--jq` for multi-line filters.

## Core Operations

### 1. Find Pull Request

```bash
bkt pr list --state OPEN --json \
  | jq --arg b "$(git branch --show-current)" \
    '.pull_requests[] | select(.source.branch.name == $b) | {id, title}'
```

### 2. Fetch Unresolved Comments

**CodeRabbit author identity differs by platform:**

- **Cloud:** `user.display_name == "Code Rabbit"` and `user.nickname == "code.rabbit"` (no `coderabbitai` handle).
- **Data Center:** `user.name` / `user.slug` matches `coderabbitai`, `coderabbit[bot]`, or `coderabbitai[bot]`.

**Meta comments to exclude** (not actionable review findings):

- Walkthrough / tips / "Reviews paused" auto-generated summaries
- "Actionable comments posted: N" status stubs
- Plain bot replies to user comments (e.g. "acknowledged", thumbs-up)
- Outside-diff comments that CodeRabbit nests inside `> ` quote blocks — these are summaries of issues already posted elsewhere
- Comments with `✅ Addressed in commits <sha>..<sha>` — CodeRabbit marks these itself after a push

**Heuristic filter** that isolates actionable issues: content starts with `_` (severity header like `_⚠️ Potential issue_ | _🟠 Major_`), contains a `🤖 Prompt for AI Agents` block, and does not contain `✅ Addressed in commits`.

**Bitbucket Cloud** (resolution filter server-side):

```bash
bkt pr comments <id> --state unresolved --json | jq '
  [.comments[]
    | select(
        .user.display_name == "Code Rabbit"
        or .user.nickname == "code.rabbit"
        or ((.user.name // .user.slug // "") | test("coderabbit"; "i"))
      )
    | select(.content.raw | test("Prompt for AI Agents"))
    | select(.content.raw | test("✅ Addressed in commits") | not)
    | select(.content.raw | startswith("_"))
    | {id, content: .content.raw}]
'
```

**Bitbucket Data Center** (no server-side resolution status):

```bash
bkt pr comments <id> --json | jq '
  [.comments[]
    | select(((.user.name // .user.slug // .user.display_name // "") | test("coderabbit"; "i")))
    | select(.content.raw | test("Prompt for AI Agents"))
    | select(.content.raw | test("✅ Addressed in commits") | not)
    | select(.content.raw | startswith("_"))
    | {id, content: .content.raw}]
'
```

On DC, additionally exclude comments resolved via task completion if the host exposes task state.

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
