# Repository scaffolding and CI/CD standards

This is the reference pattern for repositories in the `rocjay1` namespace. It
was distilled from the standalone Email-to-RSS Bridge and Garmin Connect MCP
work completed in Codex sessions
`019ffd30-0e72-71f2-9daa-52d56720e4db` and
`019ff75e-8ad4-7091-804d-d4e9b93f9d9d`.

The goal is consistency at the boundaries that affect maintenance, security,
and deployment. Repositories do not need identical files when their runtimes
or deployment owners differ.

## Reference repository shape

```text
.
├── .github/
│   ├── dependabot.yml                 # recommended dependency updates
│   └── workflows/
│       ├── application-ci.yml         # application validation
│       ├── terraform-ci.yml           # infrastructure validation, if used
│       ├── terraform-cd.yml           # guarded production apply, if used
│       └── publish-image.yml           # container publication, if used
├── docs/
│   ├── deployment.md                  # deployment ownership and release flow
│   ├── infrastructure-contract.md     # repository and shared-system boundary
│   └── operations.md                  # operator runbook and recovery
├── infra/
│   ├── bootstrap/                     # operator-only state and identity root
│   └── production/                    # application infrastructure root
├── scripts/                            # narrow, repeatable operator helpers
├── src/                                # application source
├── tests/                              # behavior and policy tests
├── AGENTS.md                           # repository-specific agent guardrails
├── LICENSE
├── README.md
├── Dockerfile                          # only for containerized services
├── pyproject.toml or package.json
└── uv.lock or package-lock.json
```

Small repositories may keep the production Terraform root directly in
`infra/` rather than `infra/production/`. That is acceptable when the root is
unambiguous and `infra/bootstrap/` remains clearly separate.

Add domain-specific directories only when the application needs them. For
example, the Email bridge has ordered `migrations/`; applied migrations are
immutable and new changes receive the next sequence number.

## What each document owns

### `README.md`

The README is the short entry point, not the full runbook. It should answer:

1. What the repository does and whether it is public, private, or unofficial.
2. What its primary user-facing workflow or interfaces are.
3. What security or privacy boundary is essential to understanding it.
4. What the repository owns at a high level.
5. How to install dependencies and run the standard local checks.
6. Where to find deployment and operations details.

Keep commands copyable and based on the committed lockfile. Link to documents
with repository-relative paths so the links work locally and on GitHub.

### `docs/operations.md`

The operator runbook is the source of truth for runtime configuration, secret
handling, bootstrap procedures, production verification, rollback, incident
response, and destructive stop conditions. Separate normal operations from
exceptional recovery steps, and mark actions that require explicit review.

### `docs/deployment.md`

Use this when deployment has enough moving parts to deserve its own guide.
Document which system deploys application code, which system applies
infrastructure, branch and environment gates, migrations, acceptance checks,
and rollback ownership.

### `docs/infrastructure-contract.md`

Use this when a repository depends on shared infrastructure or has been split
from a monorepo. List resources owned here, resources supplied elsewhere,
required interface values, and the state/deployment owner. This prevents two
repositories from silently managing the same resource.

### Infrastructure READMEs

Put a short `README.md` inside an unusual Terraform root such as
`infra/bootstrap/`. Explain who may apply it, where its state lives, what it
creates, and why CI/CD must not apply it.

### `AGENTS.md`

Record repository-specific constraints that should apply to future automated
work: directory ownership, validation commands, deployment prohibitions,
secret exclusions, migration rules, runtime versions, and commit conventions.
Do not duplicate the entire operator runbook.

## Continuous integration baseline

Application CI should:

- Run on pull requests and on relevant pushes to `main`.
- Default to `permissions: contents: read`.
- Cancel superseded runs for the same branch or pull request.
- Use a bounded job timeout.
- Pin third-party actions to full commit SHAs, with a version comment where
  useful.
- Install from the committed lockfile (`npm ci` or `uv sync --locked`).
- Run the repository's formatter or linter, tests, and type checker.
- Avoid production credentials and cloud identity.

Path filters are useful for expensive, specialized workflows. The general
application workflow may remain broad when the repository is small and every
change should prove the application still builds.

### Terraform CI

Infrastructure validation belongs in a separate, path-filtered workflow. The
pattern established by both reviewed repositories is:

1. Check `terraform fmt -check -recursive infra`.
2. Initialize bootstrap and production roots with `-backend=false`.
3. Run `terraform validate` for each root.
4. Regenerate provider locks for every supported runner/developer platform and
   fail if the committed lockfiles change.
5. Run repository-specific infrastructure policy tests.
6. On pull requests, create a credential-free speculative production plan
   with refresh and locking disabled.

The speculative plan must use syntactically valid placeholders, omit the
remote backend, and receive no OIDC identity, remote-state access, or real
provider secrets. It proves configuration shape; it is not evidence of current
production state.

Policy tests should encode invariants that Terraform alone cannot express
clearly, such as exact resource ownership, deletion protection, least-privilege
IAM, bounded secret retention, immutable image requirements, or protected
state inventory.

## Continuous delivery baseline

Production infrastructure applies are deliberately separate from pull-request
validation:

- Trigger with `workflow_dispatch` from `main`.
- Require the GitHub `production` environment.
- Set `contents: read` and add `id-token: write` only for the deployment job.
- Authenticate with GitHub OIDC and a repository-specific service account;
  do not store cloud service-account keys.
- Serialize production applies and do not cancel an apply in progress.
- Validate required inputs before authentication or planning.
- Create a fresh saved plan, review it through automated safety gates, and
  apply that exact plan file.
- Fail closed on deletion or replacement of protected resources.
- Finish with a drift check when the provider and service support a reliable
  immediate post-apply plan.

Merging a pull request is not equivalent to applying infrastructure. A manual
dispatch is an independent production decision.

### Application delivery stays separate

The deployment mechanism may vary without weakening the common contract:

| Repository type | Application delivery | Infrastructure delivery |
| --- | --- | --- |
| Cloudflare Worker | Cloudflare Workers Builds from `main` | Manual Terraform CD from `main` |
| Containerized service | Manual image workflow publishes a commit-tagged image and resolves its digest | Manual Terraform CD deploys the immutable digest |
| Static documentation site | GitHub Pages workflow builds and deploys the site artifact | No Terraform workflow unless the site gains separately managed infrastructure |

This separation keeps application deployment, database migration, secret
rotation, state mutation, and infrastructure replacement as independently
reviewable actions.

## Terraform and identity boundaries

For repositories that own cloud infrastructure, prefer two roots:

- **Bootstrap root:** operator-controlled; creates the dedicated state bucket,
  repository-scoped deployment identity, and their protections. Its state may
  live in a reusable foundation backend. GitHub Actions never applies it.
- **Production root:** owns only the application's declared resources. It uses
  the dedicated backend and repository-specific identity.

Keep state buckets and deployment identities repository-specific even when a
shared Workload Identity provider or foundation bucket is reused. Grant the
GitHub repository only permission to impersonate its own service account, then
grant that account only the resource-level permissions required by its root.

Commit stable, non-secret Terraform backend coordinates such as state-bucket
names and prefixes in the backend configuration. This makes the state
destination reviewable and avoids a mutable CI variable silently selecting a
different state. Use partial backend configuration only while bootstrap must
create a not-yet-known destination, then normalize it once that destination is
stable.

Store other non-secret deployment coordinates such as project IDs, regions,
service-account emails, and provider IDs as GitHub variables when they vary by
environment or are intentionally supplied by the deployment boundary. Store
provider credentials and application secrets in the production environment or
the cloud secret manager, whichever owns their lifecycle. Pin runtime secret
references to exact versions when reproducibility or controlled rotation
matters.

## Cross-repository baseline

Use these defaults unless a repository documents a reason to diverge:

| Concern | Standard |
| --- | --- |
| Commits | Conventional Commits; imperative summary of 72 characters or fewer |
| Runtimes | Declare the supported version and commit the dependency lockfile |
| GitHub Actions | Pin third-party actions to full SHAs |
| Permissions | Declare the smallest workflow or job permissions explicitly |
| Dependencies | Weekly Dependabot updates for the ecosystems in use |
| Secret scanning | Pull-request and `main` scanning, plus a scheduled full-history scan when appropriate |
| Generated/local files | Ignore virtual environments, dependencies, build output, Terraform working data, plans, state, and local secret files |
| Infrastructure | Format, validate, policy-test, and plan before apply |
| Releases | Deploy immutable artifacts where the platform supports them |
| Verification | Distinguish built, published, applied, and verified states |

## Observed differences and convergence targets

The reviewed repositories already share the important infrastructure pattern,
but their scaffolding is not identical:

| Area | Email-to-RSS Bridge | Garmin Connect MCP | Convergence target |
| --- | --- | --- | --- |
| README | concise entry point with docs links | detailed product, auth, and privacy overview | both are valid; keep procedures in `/docs` |
| Operations docs | operations, deployment, and infrastructure contract | consolidated operations runbook | split only when ownership becomes difficult to scan |
| Agent guidance | repository `AGENTS.md` present | not present | add when repo-specific safety rules are durable |
| License | present | not present | make the public/private licensing decision explicit |
| Application CI | Node tests and type check | Python lint, tests, typing, and lock check | align permissions, timeouts, concurrency, and SHA pinning |
| Terraform CI | two-root validation, policy tests, speculative plan | same | retain as the infrastructure baseline |
| Terraform CD | exact plan, protected-resource gates, apply, drift check | exact plan and apply | add post-apply verification and destructive-plan gates where reliable |
| Application release | Cloudflare Builds | immutable container publication | document the deployment owner; do not force one mechanism |
| Dependency/security automation | not part of the reviewed session baseline | not part of the reviewed session baseline | adopt the portal's Dependabot and secret-scan baseline where appropriate |

These differences are not all defects. Converge on the safety and ownership
contracts first; standardize filenames and workflow presentation when doing so
does not obscure a real platform difference.

## New-repository checklist

- [ ] Write the README entry point and choose the required `/docs` files.
- [ ] Commit runtime metadata, a lockfile, `.gitignore`, and an explicit license
      decision.
- [ ] Add focused source and test directories.
- [ ] Add application CI with read-only permissions, concurrency, a timeout,
      locked dependencies, and SHA-pinned actions.
- [ ] Add Dependabot and secret scanning appropriate to the repository.
- [ ] If infrastructure is owned here, define the ownership contract before
      authoring Terraform.
- [ ] Separate operator-only bootstrap from production infrastructure.
- [ ] Add policy tests and a credential-free pull-request plan.
- [ ] Keep production identity and secrets out of pull requests.
- [ ] Document the application deployer, infrastructure deployer, migrations,
      verification, rollback, and destructive stop conditions.
- [ ] Run the local commands from the README and the same checks CI will run.
