# crashoutcompany/.github

Shared GitHub Actions workflows for the crashoutcompany apps
(Project-RDC, Project-Z, Project-blue-jeans, Project-View).

| Workflow | What it does |
|---|---|
| [`ci.yml`](.github/workflows/ci.yml) | lint + typecheck, unit tests, `vercel build`, Playwright e2e on a Neon branch |
| [`neon-branches.yml`](.github/workflows/neon-branches.yml) | create/delete a `preview/pr-*` Neon branch per PR |
| [`react-doctor.yml`](.github/workflows/react-doctor.yml) | advisory React Doctor scan: PR comment, inline review, commit status |

Each app calls them from short caller workflows (`main.yml`, `neon-branches.yml`,
`react-doctor.yml`); the first two use `secrets: inherit`. Inputs and the secrets each workflow reads are documented
at the top of the workflow file.

## Releasing

Apps pin a major tag (`@v1`). After merging a change:

```bash
git tag -f v1 && git push -f origin v1   # non-breaking
git tag v2 && git push origin v2         # breaking; bump callers one at a time
```
