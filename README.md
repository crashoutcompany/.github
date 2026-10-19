# crashoutcompany/.github

Shared GitHub Actions workflows for the crashoutcompany apps
(Project-RDC, Project-Z, Project-blue-jeans, Project-View).

| Workflow | What it does |
|---|---|
| [`ci.yml`](.github/workflows/ci.yml) | lint + typecheck, unit tests, `vercel build`, Playwright e2e on a Neon branch |
| [`neon-branches.yml`](.github/workflows/neon-branches.yml) | create/delete a `preview/pr-*` Neon branch per PR (close on `pull_request_target`, see the file header) |
| [`neon-sweep.yml`](.github/workflows/neon-sweep.yml) | scheduled safety net: delete `preview/pr-*` branches of closed PRs and leaked `ci/e2e-*` branches |
| [`react-doctor.yml`](.github/workflows/react-doctor.yml) | advisory React Doctor scan: PR comment, inline review, commit status |

Each app calls them from short caller workflows (`main.yml`, `neon-branches.yml`,
`react-doctor.yml`); `ci.yml`, `neon-branches.yml` and `neon-sweep.yml` use `secrets: inherit`. Inputs and the secrets each workflow reads are documented
at the top of the workflow file.

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
