# ci — shared CI/CD workflows for the petterbuilds platform

Public home for reusable GitHub Actions workflows shared across Lolispo apps.

It is public on purpose: GitHub blocks a **public** repo from calling a
**private** repo's reusable workflow (public→public and private→public are both
allowed), so keeping the shared pipelines here lets **every** app — public or
private — use them. Nothing secret lives here; runtime config comes from AWS SSM
and the AWS role ARN is passed in by the caller.

## Workflows

### `.github/workflows/deploy-app.yml`

Builds a static app and deploys it to `<app-id>.petterbuilds.com` (S3 sync +
CloudFront invalidation, with the RUM analytics snippet injected into each HTML
file). Config (apps bucket, CloudFront distribution, RUM ids) is read from SSM at
deploy time.

**Public app** (calls this repo directly; pass the ARN as a masked **secret** so
its account id stays out of the public run logs):

```yaml
jobs:
  deploy:
    permissions: { id-token: write, contents: read }
    uses: Lolispo/ci/.github/workflows/deploy-app.yml@main
    with:
      app-id: dominion
    secrets:
      role-arn: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}
```

**Private app**: keep calling `Lolispo/web-platform/.github/workflows/deploy-app.yml@main`
as before — that workflow is a thin passthrough to this one and fills in
`role-arn`, so private apps need no changes.

**Onboarding a new repo**: see [`CLAUDE.md`](CLAUDE.md) for the full step-by-step
(how to pick the entry point by visibility, the build contract, and verification).

| input | required | default | notes |
|-------|----------|---------|-------|
| `app-id` | yes | — | subdomain / bucket prefix |
| `role-arn` (input) | no | `""` | OIDC role ARN for the private passthrough; public apps use the secret instead |
| `install-command` | no | `npm ci` | dependency install |
| `build-command` | no | `npm run build` | build step (stage into `dist-dir`) |
| `dist-dir` | no | `dist` | build output to sync, relative to `working-directory` |
| `working-directory` | no | `.` | subdir to install/build in and resolve `dist-dir` against |
| `inject-rum` | no | `true` | set `false` to opt a site out of the RUM snippet |
| `node-version` | no | `22` | |
| `aws-region` | no | `eu-north-1` | |

| secret | required | notes |
|--------|----------|-------|
| `role-arn` | no | masked OIDC role ARN; takes precedence over the input. Public callers should use this |
