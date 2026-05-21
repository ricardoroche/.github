# ricardoroche/.github

Shared GitHub Actions workflows for the `ricardoroche` organization.

## Workflows

### PR Discord Notify (`pr-discord-notify.yml`)

Reusable workflow. Posts rich embeds to `#pr-please` Discord channel when PRs are opened/closed.

**Used by:** `hermes-agent`, `oh-my-pi`, `pantheon-dev`, `minerva`, `arachne`

### Auto-Merge on Approval (`auto-merge-on-approval.yml`)

Triggers when a PR is approved. Attempts squash merge automatically. If conflicts exist, posts a comment and stops — you don't have to touch it.

**Used by:** `hermes-agent`, `oh-my-pi`, `pantheon-dev`, `minerva`, `arachne`

### Create PR with Reviewer (`create-pr-with-reviewer.yml`)

Reusable workflow for cron jobs or other automation to open PRs with reviewer pre-assigned.

```yaml
jobs:
  create-pr:
    uses: ricardoroche/.github/.github/workflows/create-pr-with-reviewer.yml@main
    with:
      branch: "feature/my-change"
      title: "feat: my change"
      body: "Auto-generated PR"
      reviewer: "ricardoroche"
    secrets:
      BOT_TOKEN: ${{ secrets.BOT_PR_TOKEN }}
```

## Bot Identity Setup

PRs created by `GITHUB_TOKEN` show as `github-actions[bot]` but **cannot trigger other workflows** (so Discord notifications won't fire).

You have three options for a proper bot identity:

### Option A: GitHub App (Recommended)

1. Go to Settings → Developer settings → GitHub Apps → New GitHub App
2. Name: `mercury-bot` (or whatever)
3. Uncheck "Active" under Webhook (we don't need it)
4. Permissions: Contents (write), Pull Requests (write), Actions (read)
5. Create → Generate private key
6. Install app on your repos
7. Store `APP_ID` and `PRIVATE_KEY` as repo/org secrets
8. Use a token generation action in workflows:

```yaml
- uses: actions/create-github-app-token@v1
  id: app-token
  with:
    app-id: ${{ secrets.BOT_APP_ID }}
    private-key: ${{ secrets.BOT_PRIVATE_KEY }}

- uses: ricardoroche/.github/.github/workflows/create-pr-with-reviewer.yml@main
  with:
    ...
  secrets:
    BOT_TOKEN: ${{ steps.app-token.outputs.token }}
```

**Pros:** Clean bot identity, no extra account, fine-grained permissions  
**Cons:** One-time setup

### Option B: Machine User (Dedicated Account)

1. Create a new GitHub account (e.g., `ricardo-bot`)
2. Verify email
3. Generate fine-grained PAT with repo access
4. Add account as collaborator to repos
5. Store PAT as `BOT_PR_TOKEN` secret

**Pros:** Simple to understand  
**Cons:** Extra account to manage, GitHub TOS discourages it

### Option C: Custom Git Config (Quick & Dirty)

Use your existing PAT but override git identity in the workflow:

```yaml
- run: |
    git config user.name "Mercury Bot"
    git config user.email "bot@ricardoroche.dev"
    git commit -m "..."
```

**Pros:** Zero setup  
**Cons:** PR still shows as created by you; commits have bot name but GitHub links them to your account

## Recommended Flow

1. **Cron/agent** creates branch, commits with bot identity
2. **Cron/agent** calls `create-pr-with-reviewer.yml` → PR opened by bot, you added as reviewer
3. **You** review and approve
4. **Auto-merge workflow** triggers → squash merges if clean
5. **Discord notify workflow** triggers → posts to `#pr-please`
6. **If conflicts:** Auto-merge workflow comments and exits — no action needed from you

## Secrets Required

| Secret | Used By | Purpose |
|--------|---------|---------|
| `DISCORD_PR_WEBHOOK_URL` | PR notify | Discord webhook for `#pr-please` |
| `DISCORD_BOT_TOKEN` | PR notify | Bot token for emoji reactions |
| `BOT_PR_TOKEN` or `BOT_APP_ID`/`BOT_PRIVATE_KEY` | PR creation | Bot identity for opening PRs |
