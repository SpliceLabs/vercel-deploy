# SpliceLabs / vercel-deploy

Central, reusable GitHub Actions workflow that triggers Vercel deploys for SpliceLabs repos.

This file is written to be followed step by step by a person **or an AI agent**. Every step names the exact place, the exact value, and how to check it worked.

> **Security:** this repo is **public**. Never commit hook URLs, tokens, or any secret value here. Hook URLs live only in GitHub secrets named `VERCEL_HOOKS`.

---

## How it works

1. A push to `main` in a working repo (e.g. `spliceworks-marketing`) runs that repo's `.github/workflows/deploy.yml`.
2. That file calls this repo's workflow: `SpliceLabs/vercel-deploy/.github/workflows/deploy.yml@v1`.
3. This workflow reads the secret `VERCEL_HOOKS` (a JSON map of `repo-name → Deploy Hook URL`), finds the calling repo's name, and sends a `POST` to that URL.
4. Vercel builds the latest `main` of that repo.

Vercel's own Git auto-deploys are turned off in each repo with `vercel.json`, so every deploy comes through this workflow.

---

## Facts and rules (read before starting)

| Rule | Detail |
|---|---|
| One Vercel project per repo | Each working repo needs its own Vercel project and its own Deploy Hook. |
| Keep Vercel connected to GitHub | Do **not** click "Disconnect" in Vercel. Deploy Hooks build from the connected repo. |
| This repo must stay public | Public working repos cannot call a workflow in a private repo. GitHub reports this as `workflow was not found`. |
| Workflow path is fixed | The workflow must be at `.github/workflows/deploy.yml` in the **root** of this repo. |
| Callers pin to `@v1` | After changing the workflow here, move the `v1` tag (see "Updating this workflow"). |
| Org secrets only reach **public** repos on our current GitHub plan | **Private** working repos need a **repository** secret `VERCEL_HOOKS` instead (Step 3B). |
| JSON keys = exact GitHub repo names | Case-sensitive, hyphens allowed, e.g. `spliceworks-marketing`. |
| Secret **names** cannot contain hyphens | The secret is always named `VERCEL_HOOKS`. |

---

## Adding a repo (do these steps in order)

Replace `<repo>` with the GitHub repo name, e.g. `spliceworks-marketing`.

### Step 1: Create the Vercel Deploy Hook

1. Open Vercel → the project linked to `SpliceLabs/<repo>` → **Settings → Git**.
2. Confirm **Connected Git Repository** shows `SpliceLabs/<repo>`. Do not disconnect it.
3. Scroll down to **Deploy Hooks**.
4. Name: `GitHub Actions`. Branch: `main`. Click **Create**.
5. Copy the generated URL. It starts with `https://api.vercel.com/v1/integrations/deploy/`.

**Check:** the hook appears in the Deploy Hooks list.

### Step 2: Find out whether `<repo>` is public or private

Open `https://github.com/SpliceLabs/<repo>`. A **Public** or **Private** label appears next to the name.

- **Public** → do Step 3A.
- **Private** → do Step 3B.

### Step 3A (public repos): add the repo to the org secret

1. Open `https://github.com/organizations/SpliceLabs/settings/secrets/actions`.
2. Click the **pencil icon** on `VERCEL_HOOKS`.
3. Click **enter a new value** and paste the **complete** JSON, with every existing repo plus the new one:
   ```json
   {
     "spliceworks-marketing": "https://api.vercel.com/v1/integrations/deploy/prj_XXXX/YYYY",
     "<repo>": "https://api.vercel.com/v1/integrations/deploy/prj_XXXX/YYYY"
   }
   ```
   The secret value can't be viewed after saving, so keep the full JSON in the team password manager. Each update replaces the whole value.
4. Under **Repository access**, open the selector, tick `<repo>`, and click **Update selection**.
5. Click **Save changes**. Without this, nothing is saved.

**Check:** on the org secrets page, the `VERCEL_HOOKS` row's repository count went up by one.

### Step 3B (private repos): add a repository secret

1. Open `https://github.com/SpliceLabs/<repo>/settings/secrets/actions`.
2. Click **New repository secret**.
3. Name: `VERCEL_HOOKS`
4. Value (only this repo's key is needed):
   ```json
   {"<repo>": "https://api.vercel.com/v1/integrations/deploy/prj_XXXX/YYYY"}
   ```
5. Click **Add secret**.

**Check:** `VERCEL_HOOKS` is listed under **Repository secrets** in `<repo>`.

### Step 4: Add `vercel.json` to the root of `<repo>`

This stops Vercel's automatic Git deploys.

```json
{
  "git": {
    "deploymentEnabled": false
  }
}
```

If `vercel.json` already exists, add the `"git"` block to it. Don't replace other settings.

### Step 5: Add `.github/workflows/deploy.yml` to `<repo>`

This file is identical in every repo. Do not add a `with:` block.

```yaml
name: Deploy

on:
  push:
    branches: [main]
  workflow_dispatch: {}   # allows manual redeploys from the Actions tab

jobs:
  deploy:
    uses: SpliceLabs/vercel-deploy/.github/workflows/deploy.yml@v1
    secrets: inherit
```

`secrets: inherit` is required, because it passes `VERCEL_HOOKS` to this workflow.

### Step 6: Push and verify

1. Commit both files and push to `main` (or merge a PR into `main`).
2. Open `https://github.com/SpliceLabs/<repo>/actions` (signed in).
3. Open the latest **Deploy** run → job **deploy / deploy** → expand step **Trigger Vercel Deploy Hook**.
4. **Success looks like:**
   - In the `env:` section: `HOOKS: ***`
   - Last line: `Triggered Vercel deploy for <repo>`
5. In Vercel → the project → **Deployments**, a new deployment appears within seconds.
6. Confirm there is only **one** deployment for that commit. Two means `vercel.json` isn't working.

To redeploy without a code change: **Actions → Deploy → Run workflow**.

---

## Troubleshooting

Match the exact message from the run log.

| What you see | Cause | Fix |
|---|---|---|
| `workflow was not found` | This repo is private while the caller is public, the workflow isn't at `.github/workflows/deploy.yml` in this repo's root, or the `v1` tag is missing or stale. | Keep this repo public. Check the file loads at `https://github.com/SpliceLabs/vercel-deploy/blob/v1/.github/workflows/deploy.yml`. Move the tag (see below). |
| `HOOKS:` is **blank** in the `env:` section | The secret isn't reaching the run. | Public repo: tick `<repo>` in the org secret's Repository access and click **Save changes**. Private repo: add a repository secret (Step 3B). Check the secret name is exactly `VERCEL_HOOKS`, and that the caller has `secrets: inherit`. |
| `HOOKS: ***` but `No hook for '<repo>' in VERCEL_HOOKS` | The JSON has no key exactly matching the repo name. | Re-enter the JSON with the exact repo name as the key. |
| `jq: error ... parse error` | The secret value isn't valid JSON. | Re-enter it. Use double quotes, commas between entries, and no trailing comma. |
| `curl: (22) ... 404` | The hook URL was deleted or mistyped. | Create a new Deploy Hook in Vercel (Step 1) and update the secret. |
| `Invalid input, project is not defined` | The caller still has an old `with: project:` block. | Remove the `with:` block (Step 5). |
| Two Vercel deployments per commit | `vercel.json` is missing or wrong. | Redo Step 4. |
| Notice about `ubuntu-latest` migrating to Ubuntu 26 | Informational only. | Ignore it. |

---

## Updating this workflow (maintainers)

1. Edit `.github/workflows/deploy.yml` in this repo and merge to `main`.
2. Move the `v1` tag to the new commit:
   ```bash
   git pull
   git tag -f v1
   git push -f origin v1
   ```
3. Push a test commit to any working repo and verify it (Step 6).

For a **breaking** change, create a `v2` tag instead. Then update each caller's `@v1` to `@v2` one repo at a time.

### The workflow (reference copy of `.github/workflows/deploy.yml`)

```yaml
name: Vercel Deploy Hook
on:
  workflow_call: {}

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Trigger Vercel Deploy Hook
        env:
          HOOKS: ${{ secrets.VERCEL_HOOKS }}
          REPO: ${{ github.event.repository.name }}
        run: |
          HOOK_URL=$(echo "$HOOKS" | jq -r --arg r "$REPO" '.[$r] // empty')
          if [ -z "$HOOK_URL" ]; then
            echo "::error::No hook for '$REPO' in VERCEL_HOOKS"
            exit 1
          fi
          curl -fsS -X POST "$HOOK_URL"
          echo "Triggered Vercel deploy for $REPO"
```

---

## Limitations

- **No preview deployments for pull requests.** Hooks only build `main`.
- **Everyone with access to a repo that shares the org secret can use every hook URL in it.** Treat hook URLs as secrets, and rotate a hook in Vercel if one leaks.
- **Vercel plan terms:** check that the team's Vercel plan covers multiple contributors.
