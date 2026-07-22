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

**Public app** (calls this repo directly, ARN from a repo variable):

```yaml
jobs:
  deploy:
    permissions: { id-token: write, contents: read }
    uses: Lolispo/ci/.github/workflows/deploy-app.yml@main
    with:
      app-id: dominion
      role-arn: ${{ vars.AWS_DEPLOY_ROLE_ARN }}
```

**Private app**: keep calling `Lolispo/web-platform/.github/workflows/deploy-app.yml@main`
as before — that workflow is a thin passthrough to this one and fills in
`role-arn`, so private apps need no changes.

| input | required | default | notes |
|-------|----------|---------|-------|
| `app-id` | yes | — | subdomain / bucket prefix |
| `role-arn` | yes | — | OIDC role to assume; kept out of this public source |
| `install-command` | no | `npm ci` | dependency install |
| `build-command` | no | `npm run build` | build step (stage into `dist-dir`) |
| `dist-dir` | no | `dist` | directory synced to S3 |
| `node-version` | no | `22` | |
| `aws-region` | no | `eu-north-1` | |
