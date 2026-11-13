# Jeitinho Vibe org workflows

Shared GitHub Actions workflows for Jeitinho Vibe projects on Cloudflare.

## `pr-preview.yml`

One preview pipeline, two profiles. Both publish `pr-<number>.<production hostname>`,
comment a single link on the PR, smoke-test it before commenting, and remove
everything when the PR closes.

| | `stateless` | `isolated` |
| --- | --- | --- |
| Use when | the app is files only, no bindings | the app has state (D1, KV, …) and previews must not touch production data |
| Mechanism | Cloudflare Previews (`wrangler preview`) | one Worker + custom domain per PR |
| DNS / cert | minted once by `previews_enabled` on the production route | created and removed per PR |
| Cleanup | `wrangler preview delete` | `wrangler delete` (removes domain + DNS) |

### Caller requirements

- Cloudflare config committed as `wrangler.jsonc`, with `account_id` and a
  custom-domain route. The production hostname is read from that route.
- Repo secret `CLOUDFLARE_API_TOKEN` with:
  - Account · Workers Scripts · Edit
  - Account · D1 · Edit (isolated only)
  - Zone · Workers Routes · Edit
  - Zone · Zone · Read
- A workflow with the trigger, concurrency, and permissions below.

```yaml
name: PR preview

on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review, closed]

concurrency:
  group: preview-${{ github.event.pull_request.number }}
  cancel-in-progress: true

permissions:
  contents: read
  pull-requests: write

jobs:
  preview:
    uses: jeitinhovibe/.github/.github/workflows/pr-preview.yml@main
    with:
      profile: isolated
      preview_database: my-app-preview
    secrets: inherit
```

### Inputs

| Input | Default | Meaning |
| --- | --- | --- |
| `profile` | `isolated` | `isolated` or `stateless` |
| `app` | `name` from `wrangler.jsonc` | Worker name stem |
| `production_hostname` | first custom-domain route | Base host, overrides config |
| `preview_database` | `<app>-preview` | Shared D1 database for isolated previews |
| `prepare_command` | empty | Run before deploy, e.g. `./deploy.sh stage` |
| `note` | empty | Extra sentence in the PR comment |
| `smoke_path` | `/` | Path polled until it answers 200 |
| `node_version` | `22` | Node version for build/publish |

Secrets: `CLOUDFLARE_API_TOKEN` (required).

### Notes

- The preview database is shared across PRs and is never deleted on close, so
  migrations are fast and previews keep working across pushes.
- Isolated previews use a per-PR custom domain, so deleting the Worker removes
  its DNS record too — no wildcard to maintain.
- Comments from older per-repo workflows (`<!-- halfsies-preview -->`,
  `<!-- pr-preview-link -->`) are recognised and reused, then rewritten with the
  `<!-- preview -->` marker.
