# Claude Feature Pipeline Template

Automated feature development pipeline that turns Notion feature requests into GitHub PRs using Claude AI. Mark a feature as "Ready for dev" in Notion and Claude will implement it, open a PR, and keep Notion updated throughout.

## How It Works

```
Notion (Ready for dev)
  → Cloudflare Worker
    → GitHub Actions
      → Claude API (reads repo, implements feature)
        → Pull Request opened
          → Notion updated (Review + PR URL)
            → Merge PR
              → Notion updated (Done)
```

---

## Prerequisites

Before using this template you need the following already set up:

- A **Cloudflare Worker** (`notion-github-bridge`) deployed and connected to your Notion workspace. See the [bridge repo](https://github.com/stegel/notion-github-bridge) for setup instructions.
- A **Notion Feature Requests database** with the following properties:
  - `Title` — title
  - `State` — status (`Not started`, `Ready for dev`, `In progress`, `Review`, `Done`)
  - `Project` — relation to your Coding Projects database
  - `Branch name` — text
  - `PR URL` — URL
  - `Description` — text (optional, used for feature details)
- A **Notion Coding Projects database** with:
  - `Name` — title
  - `GitHub Repo` — text (format: `owner/repo-name`)
- An **Anthropic API account** with credits at [console.anthropic.com](https://console.anthropic.com)

---

## Setting Up a New Repo from This Template

### Step 1 — Create repo from template

On GitHub, click **Use this template** → **Create a new repository**.

### Step 2 — Add GitHub secrets

Go to your new repo → **Settings** → **Secrets and variables** → **Actions** → add these secrets:

| Secret | Value |
|---|---|
| `ANTHROPIC_API_KEY` | Your Anthropic API key (`sk-ant-api03-...`) |
| `NOTION_API_KEY` | Your Notion integration token (`ntn_...`) |

### Step 3 — Enable Actions permissions

Go to **Settings** → **Actions** → **General** → **Workflow permissions**:
- Select **Read and write permissions**
- Check **Allow GitHub Actions to create and approve pull requests**
- Click **Save**

### Step 4 — Add the repo to Notion

In your Notion **Coding Projects** database, add a new row:
- `Name` — your project name
- `GitHub Repo` — `your-github-username/your-repo-name`

### Step 5 — Connect Notion integration

Open your **Coding Projects** database in Notion → **⋯** → **Connections** → connect your Notion integration. Do the same for your **Feature Requests** database if not already connected.

### Step 6 — Update CLAUDE.md

Edit `CLAUDE.md` in the root of your repo to describe your project — tech stack, directory structure, rules, and any conventions Claude should follow. This is the most important thing you can do to improve output quality.

### Step 7 — Update .claude-context

Edit `.claude-context` to tell Claude which directories are most relevant and which to ignore:

```json
{
  "priority_dirs": ["src", "components", "lib"],
  "exclude_dirs": ["legacy", "archive", "fixtures"]
}
```

---

## Using the Pipeline

### Basic feature request

In your Notion Feature Requests database, create a new row:
- `Title` — describe the feature (e.g. `Add dark mode toggle`)
- `Project` — link to your project in Coding Projects
- `Description` — optional extra detail for Claude
- `State` — set to `Ready for dev`

The pipeline fires automatically when State changes to `Ready for dev`.

### Targeting a specific prototype or subdirectory

If your repo contains multiple prototypes or sub-projects, prefix the title with the path in square brackets:

```
[username/prototype-name] Add dark mode toggle
```

Claude will only see files within `src/prototypes/username/prototype-name/` plus shared library files, keeping the context focused and the output fast.

### Tracking progress

| Notion State | Meaning |
|---|---|
| `Ready for dev` | You marked it — pipeline is about to fire |
| `In progress` | Worker received it, GitHub Actions is running |
| `Review` | PR is open — check the PR URL field |
| `Done` | PR was merged |

---

## Files in This Template

| File | Purpose |
|---|---|
| `.github/workflows/claude-feature-builder.yml` | Triggered by Notion → builds feature → opens PR → updates Notion |
| `.github/workflows/pr-merged.yml` | Triggered on PR merge → updates Notion to Done |
| `scripts/claude_builder.py` | Reads repo, calls Claude API, applies file changes |
| `CLAUDE.md` | Project brief for Claude — **edit this per project** |
| `.claude-context` | Directory hints — **edit this per project** |

---

## Troubleshooting

**Notion status not updating to "In progress"**
- Check Cloudflare Worker logs with `wrangler tail`
- Verify your Notion integration is connected to the Feature Requests and Coding Projects databases
- Confirm `NOTION_API_KEY` is set correctly in Cloudflare Worker secrets

**GitHub Actions not triggering**
- Confirm the `repository_dispatch` event is firing — check the Worker logs for `GitHub dispatch status: 204`
- Verify `GITHUB_TOKEN` in Cloudflare Worker secrets has `repo` and `workflow` scopes

**Claude failing with max_tokens error**
- Break the feature into smaller tasks
- Add more specific exclusions to `.claude-context` to reduce context size

**PR created but Notion not updating to "Review"**
- Check the `Update Notion → Review` step in the GitHub Actions run
- Verify `NOTION_API_KEY` is set in GitHub repo secrets

**PR merged but Notion not updating to "Done"**
- Check the `pr-merged.yml` workflow run in the Actions tab
- Verify the PR body contains the Notion page ID in the format `**Notion feature:** <id>`

---

## Secrets Reference

| Secret | Where | Purpose |
|---|---|---|
| `ANTHROPIC_API_KEY` | GitHub repo secrets | Claude API access |
| `NOTION_API_KEY` | GitHub repo secrets | Update Notion from Actions |
| `NOTION_API_KEY` | Cloudflare Worker secrets | Read/update Notion from Worker |
| `GITHUB_TOKEN` | Cloudflare Worker secrets | Trigger repository_dispatch |
| `WEBHOOK_SECRET` | Cloudflare Worker secrets | Authenticate Notion webhook |