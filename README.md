# crashoutcompany/.github

Shared GitHub Actions workflows for the crashoutcompany apps
(Project-RDC, Project-Z, Project-blue-jeans, Project-View).

| Workflow | What it does |
|---|---|
| [`ci.yml`](.github/workflows/ci.yml) | lint + typecheck, unit tests, `vercel build`, Playwright e2e on a Neon branch |
| [`neon-branches.yml`](.github/workflows/neon-branches.yml) | create/delete a `preview/pr-*` Neon branch per PR and, with `VERCEL_TOKEN`, its Vercel preview env vars (close on `pull_request_target`, see the file header) |
| [`neon-sweep.yml`](.github/workflows/neon-sweep.yml) | scheduled safety net: delete `preview/pr-*` branches of closed PRs and leaked `ci/e2e-*` branches |
| [`react-doctor.yml`](.github/workflows/react-doctor.yml) | advisory React Doctor scan: PR comment, inline review, commit status |

Each app calls them from short caller workflows (`main.yml`, `neon-branches.yml`,
`react-doctor.yml`); `ci.yml`, `neon-branches.yml` and `neon-sweep.yml` use `secrets: inherit`. Inputs and the secrets each workflow reads are documented
at the top of the workflow file.

## Neon preview branches and Vercel

`neon-branches.yml` gives every same-repo PR its own `preview/pr-<n>-<ref>` Neon
branch. With `VERCEL_TOKEN` set it also hands that branch's URLs to Vercel, so
the PR's preview deploys stop reading the shared `DATABASE_URL`:

- `create` upserts encrypted, preview-only env vars scoped to the PR's head ref
  (`gitBranch`): the pooled URL into `DATABASE_URL` and the direct URL into
  `DIRECT_URL`. `delete` removes them when the PR closes.
- Secrets or variables (from the calling repo): `VERCEL_TOKEN` (secret; without
  it the Vercel steps log a notice and skip, so existing callers keep working),
  `VERCEL_PROJECT_ID`, and `VERCEL_TEAM_ID` for team projects (not
  `VERCEL_ORG_ID`, which `ci.yml` uses).
- Inputs: `vercel-database-url-key` (default `DATABASE_URL`) and
  `vercel-direct-url-key` (default `DIRECT_URL`; an empty string skips it).
- **Redeploy once.** Vercel starts the first preview build of a newly opened PR
  alongside the `create` job, so it can run before the env vars exist and read
  the fallback value. Redeploy that preview once (or push again) after the job
  passes; later builds pick the branch URLs up.
- `NEON_CI_PARENT_BRANCH` (or the `neon-parent` input) must be a scrubbed/seeded
  branch such as `development`, never `main` or production: PR code and preview
  deploys run against copies of it.

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
