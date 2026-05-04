# fem-cicd-service

A sample [Astro](https://astro.build) static site used to demonstrate a CI/CD pipeline on GitHub Actions, with automated deployment to AWS S3.

## Tech Stack

- **Framework:** Astro 5 (static site generator)
- **Language:** TypeScript
- **Runtime:** Node.js ≥ 22.22
- **CI/CD:** GitHub Actions
- **Hosting:** AWS S3

## Project Structure

```
fem-cicd-service/
├── src/
│   └── pages/
│       └── index.astro          # Home page
├── .github/
│   ├── actions/
│   │   └── build-astro/
│   │       └── action.yml       # Composite action: install, build, upload artifact
│   └── workflows/
│       ├── _build.yml           # Reusable build workflow (called by CI and Deploy)
│       ├── ci.yml               # PR validation — builds only, no deploy
│       └── deploy.yml           # Merge to main — builds and deploys to S3
├── package.json
└── astro.config.*
```

## Local Development

```bash
npm install
npm run dev      # start dev server at http://localhost:4321
npm run build    # production build → dist/
npm run preview  # preview the production build locally
```

## CI/CD Pipeline

### CI (`ci.yml`)

Triggered on every pull request targeting `main`. Calls the reusable `_build.yml` workflow to verify the site builds successfully. Does **not** deploy.

### Deploy (`deploy.yml`)

Triggered on every push to `main` (and manually via `workflow_dispatch`). Runs in two sequential jobs:

1. **build** — calls `_build.yml`, which uses the `build-astro` composite action to set up Node, run `npm ci`, run `npm run build`, and upload the `dist/` artifact.
2. **deploy** — downloads the artifact, assumes an AWS IAM role via OIDC, and syncs `dist/` to the target S3 bucket.

### Reusable Workflow (`_build.yml`)

A `workflow_call` workflow consumed by both `ci.yml` and `deploy.yml`. Accepts optional inputs:

| Input | Default | Description |
|---|---|---|
| `node-version` | `22.22` | Node.js version used for the build |
| `artifact-name` | `dist` | Name of the uploaded build artifact |

### Composite Action (`build-astro`)

Encapsulates the full build steps (setup Node with npm cache → `npm ci` → `npm run build` → upload artifact) so they can be reused without duplicating YAML.

## AWS Deployment

The deploy job uses [OIDC-based authentication](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services) — no long-lived AWS credentials are stored in GitHub secrets.

| Setting | Value |
|---|---|
| AWS Region | `us-west-2` |
| IAM Role | `arn:aws:iam::869549134078:role/fem-cicd-service` |
| S3 Bucket | `s3://don-fem-cicd-service/` |

The sync command uses `--delete` to remove files from the bucket that no longer exist in the build output.

## Concurrency

- **CI** — concurrent runs for the same PR are cancelled, keeping runners free.
- **Deploy** — concurrent production deploys are queued (not cancelled) to prevent partial or overlapping deployments.
