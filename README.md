# poke-claude

Keep your [Claude Code](https://docs.anthropic.com/en/docs/claude-code) 5‑hour usage window alive, in the simplest way possible. Avoids both the soon-to-be deprecated `-p` flag and Agent SDK billing by using a scheduled single‑prompt heartbeat.

## Why this exists

Claude Code enforces a rolling 5‑hour usage window. If you don't send a prompt at the start of a new window, your quota resets unpredictably or gets wasted. Scheduled heartbeats fix that. Existing solutions either rely on the soon‑to‑be‑deprecated `-p` flag (which will eventually use a separate billing track in Agent SDK) or require a persistently running supervisor process.

`poke-claude` combines:

- A **single piped prompt** (`echo "Hello" | claude --model haiku`). It just works.
- **GitHub Actions** to run the heartbeat on a schedule.
- **FastCron** (or any external cron service) to trigger the workflow **exactly on time**. GitHub's built‑in scheduler drifts, but FastCron fires within seconds.
- The **`id-token: write`** permission fix that I initially missed, causing a mysterious 401 error.

## Prerequisites

- A private (or public) GitHub repository.
- A [Claude Code](https://docs.anthropic.com/en/docs/claude-code/overview) subscription.
- A free [FastCron](https://fastcron.com) account (or any service that can make HTTP requests on a schedule, but guide will use FastCron).
- A [GitHub Personal Access Token (PAT)](https://github.com/settings/tokens) with `Actions` read & write access to your repository.
- A Claude OAuth token generated with `claude setup-token`.

## Quickstart

### 1. Add the workflow file

Create `.github/workflows/poke-claude.yml` in your repository:

```yaml
name: poke-claude

on:
  workflow_dispatch: {}   # Triggered by FastCron via API
  # Optional fallback schedule (GitHub's own scheduler, may drift, not recommended)
  # schedule:
  #   - cron: '0 12,17,22,3 * * *'

jobs:
  poke:
    runs-on: ubuntu-latest
    timeout-minutes: 2
    permissions:
      id-token: write   # REQUIRED for OAuth authentication
      contents: read
    steps:
      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code

      - name: Send heartbeat
        env:
          CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_OAUTH_TOKEN }}
        run: |
          echo "Say Hello and nothing else." | claude --model haiku
        continue-on-error: true   # Gracefully handles quota exhaustion
```

If you don't want to set this up on an existing repository, you can fork this one.

**Why `continue-on-error: true`?** When your usage limit is already exhausted, the workflow won't fail. It just exits. This keeps your repository's Actions tab clean.

### 2. Add your Claude OAuth token as a GitHub secret

1. Run `claude setup-token` locally. Copy the token (starts with `sk-ant-oat01-...`).
2. In your repository, go to **Settings -> Secrets and variables -> Actions**.
3. Create a new secret named `CLAUDE_OAUTH_TOKEN` and paste the token.

### 3. Create a GitHub Personal Access Token (PAT) for FastCron

1. Go to [GitHub -> Fine‑grained tokens](https://github.com/settings/tokens?type=beta).
2. Click **Generate new token**.
3. **Repository access:** Select your repo.
4. **Permissions:** Under **Repository permissions**, set **Actions** to **Read and write**.
5. Generate and copy the token.

### 4. Set up FastCron

Create a new cron job in FastCron and set these settings:

#### Basic settings
- **URL:** `https://api.github.com/repos/YOUR_USERNAME/YOUR_REPO/actions/workflows/poke-claude.yml/dispatches` (just copy `YOUR_USERNAME/YOUR_REPO` off your address bar)
- Click `Crontab` and decide when you want to poke Claude and start your usage window. Remember to your timezone correctly.

#### Send HTTP request
- **Method:** `POST`
- **HTTP Headers:**
  - `Authorization`: `Bearer YOUR_GITHUB_PAT`
  - `Content-Type`: `application/json`
  - `Accept`: `application/vnd.github+json`
- **POST data:** `{"ref":"main"}` (use `master` if that's your default branch)
- **Username:** Just use your GitHub username.
- **Password:** Just use YOUR_GITHUB_PAT.

### 5. Test

- Trigger the workflow manually from the **Actions** tab using **Run workflow**.
- Or click **:** and **Run** in FastCron and check the log for failure or success.

## Why not use `claude -p`?

The `-p` flag is being moved to a separate billing model (Agent SDK credits). Using it in automation may cost extra in the future. This workflow pipes the prompt into an interactive Claude session, which uses your standard subscription and avoids that entire billing track.

## License

MIT. Do whatever you want with it. Have fun. If it saves you from a 401 rabbit hole, or makes your life easier, star the repo.