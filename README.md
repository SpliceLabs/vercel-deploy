# SpliceLabs / vercel-deploy

Central, reusable GitHub Actions workflow that triggers Vercel Deploy Hooks for every SpliceLabs repo.

Every working repo uses the same caller file. This workflow looks up the calling repo's name in the org secret `VERCEL_HOOKS` and triggers that repo's Vercel Deploy Hook.

## One-time setup

1. **Workflow location:** the workflow must live at `.github/workflows/deploy.yml` in the root of this repo. GitHub won't find it in a subfolder.
2. **Allow other repos to use it:** Settings → Actions → General → Access →
   "Accessible from repositories in the 'SpliceLabs' organization" → Save.
3. **Tag a release** so callers can pin to it:
```bash
   git tag v1 && git push origin v1
```
4. **Create the org secret** `VERCEL_HOOKS`: SpliceLabs → Settings → Secrets and variables → Actions → New organization secret.
   - Value: a JSON map from GitHub repo name to Deploy Hook URL:
```json
     {
       "spliceworks-marketing": "https://api.vercel.com/v1/integrations/deploy/prj_.../...",
       "another-repo": "https://api.vercel.com/v1/integrations/deploy/prj_.../..."
     }
```
   - Repository access: all repos that deploy to Vercel.

## Adding a repo

1. **Vercel:** open the project → Settings → Git → Deploy Hooks → create a hook for `main` → copy the URL.
   Keep the Git repository connected; the hook builds from it.
2. **Org secret:** update `VERCEL_HOOKS` and add a line for the repo. The key must match the GitHub repo name exactly.
3. **In the working repo**, add these two files (identical in every repo):

   `vercel.json`, which stops Vercel's automatic Git deploys:
```json
   {
     "git": {
       "deploymentEnabled": false
     }
   }
```
   If the repo already has a `vercel.json`, add the `"git"` block to it.

   `.github/workflows/deploy.yml`:
```yaml
   name: Deploy
   on:
     push:
       branches: [main]
     workflow_dispatch: {}

   jobs:
     deploy:
       uses: SpliceLabs/vercel-deploy/.github/workflows/deploy.yml@v1
       secrets: inherit
```
4. Push to `main` and check the repo's **Actions** tab. The run should end with
   `Triggered Vercel deploy for <repo-name>`.

## Updating the workflow

Change `deploy.yml` here, then move the tag:
```bash
git tag -f v1 && git push -f origin v1
```
For a breaking change, cut `v2` instead and update callers one by one.

## Troubleshooting

| Error | Cause |
|---|---|
| `workflow was not found` | The `v1` tag is missing, Access (setup step 2) isn't enabled, or the workflow isn't at `.github/workflows/deploy.yml` in the repo root. |
| `No hook for '<repo>' in VERCEL_HOOKS` | The repo name is missing or misspelled in the JSON, or the secret isn't shared with that repo. |
| Two Vercel builds for one commit | `vercel.json` is missing or doesn't have `"deploymentEnabled": false`. |

## Notes

- PRs don't get Vercel preview deployments with this setup; hooks only build the configured branch.
- Every repo with access to `VERCEL_HOOKS` can read all hook URLs in it.
