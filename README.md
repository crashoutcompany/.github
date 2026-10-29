# crashoutcompany/.github

Shared GitHub Actions workflows for the crashoutcompany apps
(Project-RDC, Project-Z, Project-blue-jeans, Project-View).

| Workflow | What it does |
|---|---|
| [`ci.yml`](.github/workflows/ci.yml) | lint + typecheck, unit tests, `vercel build`, Playwright e2e on a Neon branch |
| [`neon-branches.yml`](.github/workflows/neon-branches.yml) | create/delete a `preview/pr-*` Neon branch per PR and, opt-in, its Vercel preview env vars (close on `pull_request_target`, see the file header) |
| [`neon-sweep.yml`](.github/workflows/neon-sweep.yml) | scheduled safety net: delete `preview/pr-*` branches of closed PRs and leaked `ci/e2e-*` branches |
| [`react-doctor.yml`](.github/workflows/react-doctor.yml) | advisory React Doctor scan: PR comment, inline review, commit status |

Each app calls them from short caller workflows (`main.yml`, `neon-branches.yml`,
`react-doctor.yml`); `ci.yml`, `neon-branches.yml` and `neon-sweep.yml` use `secrets: inherit`. Inputs and the secrets each workflow reads are documented
at the top of the workflow file.

## Neon preview branches and Vercel

`neon-branches.yml` gives every same-repo PR its own `preview/pr-<n>-<ref>` Neon
branch. With `vercel-preview-env: true` it also hands that branch's URLs to
Vercel, so the PR's preview deploys stop reading the shared Preview
`DATABASE_URL`. It is opt-in because every caller already has `VERCEL_TOKEN`
for `ci.yml`.

- `create` upserts encrypted, preview-only env vars scoped to the PR's head ref
  (`gitBranch`): the pooled URL into `DATABASE_URL` and the direct URL into
  `DIRECT_URL`. `delete` removes them when the PR closes. Each var carries the
  comment `managed by crashoutcompany/.github neon-branches`.
- `neon-sweep.yml` with `vercel-preview-env: true` deletes those marked vars
  when their PR is no longer open or their Neon branch is gone (expired or
  swept), so previews fall back to the shared value instead of a dead endpoint.
- Secrets or variables (from the calling repo): `VERCEL_TOKEN` (secret, write
  access to the project's env vars), `VERCEL_PROJECT_ID`, and for team projects
  `VERCEL_TEAM_ID` or the `team_...` `VERCEL_ORG_ID` that `ci.yml` already uses.
- Inputs: `vercel-database-url-key` (default `DATABASE_URL`) and
  `vercel-direct-url-key` (default `DIRECT_URL`; an empty string skips it).
- `NEON_CI_PARENT_BRANCH` (or the `neon-parent` input) must be a scrubbed/seeded
  branch, never production: PR code and preview deploys run against copies of
  it. With `vercel-preview-env` the job fails if it is the Neon project's
  default branch.
- **Make the shared fallback safe.** Vercel starts the first preview build of a
  newly opened PR alongside the `create` job, so that build can read the shared
  Preview `DATABASE_URL`. Point that value at a scrubbed branch too, never
  production. Redeploy (or push) to switch the preview to the PR's own branch.
- PRs opened before you turn this on get their vars when reopened.
- Migrations: with `prisma-migrate: true` the `create` job also checks out the
  PR and runs `prisma migrate deploy` on its branch (direct URL). Add
  `synchronize` to the caller's `pull_request` types so migrations from later
  pushes land too; rerunning `create` on an existing branch is a no-op. This
  runs PR code, so it only runs on `pull_request` (same-repo, non-Dependabot)
  and stays the job's last step. `node-version` / `pnpm-version` default to
  `24` / `10.1.0`, as in `ci.yml`. Without it, a PR that changes the schema
  gets a preview on the parent's schema.

Caller:

```yaml
on:
  pull_request:
    types: [opened, reopened, synchronize]
  pull_request_target:
    types: [closed]
jobs:
  neon:
    uses: crashoutcompany/.github/.github/workflows/neon-branches.yml@<full commit sha> # v1
    with:
      vercel-preview-env: true
      prisma-migrate: true
    secrets: inherit
  sweep:
    uses: crashoutcompany/.github/.github/workflows/neon-sweep.yml@<full commit sha> # v1
    with:
      vercel-preview-env: true
    secrets: inherit
```

## Releasing

Callers should pin a full commit SHA with the version as a comment, not the
moving tag:

```yaml
uses: crashoutcompany/.github/.github/workflows/ci.yml@<full commit sha> # v1
```

The workflows run with `secrets: inherit`, so they execute with every app's
Neon, Vercel and PostHog keys. A force-moved `@v1` would silently change code
that runs with those secrets; a SHA can't move. Add the `github-actions`
ecosystem to each app's `dependabot.yml` so SHA bumps arrive as reviewable PRs.

After merging a change, cut an immutable release tag, then move the major tag
for apps that still follow it:

```bash
git tag v1.2.0 && git push origin v1.2.0      # immutable; Dependabot bumps to this
git tag -f v1 && git push -f origin v1        # non-breaking, moving major
git tag v2 && git push origin v2              # breaking; bump callers one at a time
```
