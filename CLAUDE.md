# Steering: onboarding a repo to petterbuilds platform deploys

Follow this exactly when wiring a repo to auto-deploy to `<app-id>.petterbuilds.com`
via `.github/workflows/deploy-app.yml` in this repo. The canonical deploy logic
lives here (public `Lolispo/ci`); `Lolispo/web-platform` is a private passthrough
to it for private apps.

## 1. Pick the entry point by repo VISIBILITY — this is the critical decision

GitHub blocks a **public** repo from calling a **private** repo's reusable workflow
(public→public and private→public are both allowed). Getting this wrong makes the
run fail immediately (0s, "This run likely failed because of a workflow file
issue", no jobs — and it "cannot be retried").

Check first: `gh api repos/Lolispo/<repo> --jq .visibility` (use `gh1` for Lolispo
repos).

- **private** → call the passthrough: `Lolispo/web-platform/.github/workflows/deploy-app.yml@main`.
  You can omit `role-arn` (it defaults to the shared role). Nothing else needed.
- **public** → call THIS repo directly: `Lolispo/ci/.github/workflows/deploy-app.yml@main`,
  and pass the role ARN as a **secret** (see §3). **Never** route a public repo
  through `web-platform` — that's the public→private block above.

## 2. Caller workflow

Private app:

```yaml
name: Deploy
on: { push: { branches: [main] }, workflow_dispatch: {} }
jobs:
  deploy:
    permissions: { id-token: write, contents: read }
    uses: Lolispo/web-platform/.github/workflows/deploy-app.yml@main
    with:
      app-id: <subdomain>
```

Public app:

```yaml
name: Deploy
on: { push: { branches: [main] }, workflow_dispatch: {} }
jobs:
  deploy:
    permissions: { id-token: write, contents: read }
    uses: Lolispo/ci/.github/workflows/deploy-app.yml@main
    with:
      app-id: <subdomain>
    secrets:
      role-arn: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}
```

`permissions.id-token: write` is required (OIDC) at every level.

## 3. Role ARN — use a SECRET for public repos, never a plain variable

A public repo's Actions logs are public, and `configure-aws-credentials` prints
`role-to-assume`. A **variable** prints the ARN (with the AWS account id) in the
clear; a **secret** is masked to `***`. So for a public app, set a repo secret
(once): 

```
gh secret set AWS_DEPLOY_ROLE_ARN -R Lolispo/<repo> --body "<deploy-role-arn>"
```

The ARN is not itself sensitive (access is gated by the OIDC trust policy
`repo:Lolispo/*`, not secrecy) — masking just keeps the account id out of public
logs. Get the value from SSM `/platform/deploy-role-arn` or the `role-arn` default
in `web-platform`'s workflow. Do NOT hardcode the ARN in any public repo's source
(including this file).

Private apps need no secret — `web-platform` supplies the ARN as an input and
their logs are private.

## 4. Build contract (applies to every app)

- The S3 sync is `aws s3 sync <dist> ... --delete` with **no VCS exclusions**, so
  it must deploy a CLEAN directory — never the repo root (that would publish
  `.git/`, `scripts/`, docs). Add a `scripts/build-static.sh` that stages only the
  runtime files into `dist/`, and set `build-command` to run it. For a repo whose
  deployable site is a subfolder, set `working-directory` (install/build/cache and
  `dist-dir` resolve against it).
- `setup-node` uses `cache: npm` and the install step runs `npm ci`, so the repo
  (or `working-directory`) needs `package.json` + `package-lock.json` even for a
  no-dependency static site. Gitignore `dist/` and `node_modules/`.
- Opt a site out of the RUM analytics snippet with `inject-rum: false`.

## 5. Lolispo (hobby) repo mechanics

- Use `gh1` for GitHub API/CLI (Lolispo account), and push over SSH with the
  Lolispo key. `gh1`'s token lacks the `workflow` scope, so push workflow-file
  changes via git/SSH, not the API.

## 6. Verify — do not assume

- Config alone deploys nothing; push to `main` (or `workflow_dispatch`).
- Watch: `gh1 run list --workflow deploy.yml --limit 1`. A 0s "workflow file
  issue" means the public→private mismatch in §1.
- Confirm the sync + invalidation ran, then hit the live URL.
- Public repo: confirm the account id is masked —
  `gh1 run view <id> --log | grep -c <account-id>` should be `0`.
