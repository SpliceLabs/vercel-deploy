# SpliceLabs / vercel-deploy

Central, reusable GitHub Actions workflow that triggers Vercel Deploy Hooks for every SpliceLabs repo.

## One-time setup

1. **Create this repo** as `SpliceLabs/vercel-deploy` and push this folder.
2. **Allow other repos to use it:** Settings → Actions → General → Access →
   "Accessible from repositories in the SpliceLabs organization".
3. **Tag a release** so callers can pin to it:
   ```bash
   git tag v1 && git push origin v1
   ```

## Adding a repo

1. **Vercel:** open the project → Settings → Git → Deploy Hooks → create a hook for `main` → copy the URL.
2. **GitHub org secret:** SpliceLabs → Settings → Secrets and variables → Actions → New organization secret
   - Name: `VERCEL_HOOK_<KEY>` (e.g. `VERCEL_HOOK_FRONTEND`)
   - Value: the hook URL
   - Repository access: the repo(s) that should trigger it
3. **In the working repo**, add:
   - `vercel.json` with `"git": { "deploymentEnabled": false }`
   - `.github/workflows/deploy.yml` calling this workflow with `project: <KEY>` and `secrets: inherit`

Templates for both files are in `caller-template/` of the setup bundle.

## Caller example

```yaml
name: Deploy
on:
  push:
    branches: [main]
  workflow_dispatch: {}

jobs:
  deploy:
    uses: SpliceLabs/vercel-deploy/.github/workflows/deploy.yml@v1
    with:
      project: FRONTEND
    secrets: inherit
```

One repo deploying several Vercel projects: add one job per project key.

## Updating the workflow

Change `deploy.yml` here, then move the tag (`git tag -f v1 && git push -f origin v1`)
or cut `v2` and update callers when the change is breaking.

## Notes

- PRs don't get Vercel preview deployments with this setup (hooks only build the configured branch).
- If a run fails with "secret not found", the org secret is missing or not shared with that repo.
