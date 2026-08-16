# Repository scaffolding and CI/CD standards

This is the reference pattern for repositories in the `rocjay1` namespace. It
was distilled from the standalone
[Email-to-RSS Bridge](https://github.com/rocjay1/email-to-rss-bridge) and
[Garmin Connect MCP](https://github.com/rocjay1/garmin-connect-mcp), then
expanded as additional repositories adopted the shared boundaries.

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
│       └── publish-image.yml          # container publication, if used
├── docs/
│   ├── deployment.md                  # deployment ownership and release flow
│   ├── infrastructure-contract.md     # repository and shared-system boundary
│   └── operations.md                  # operator runbook and recovery
├── infra/
│   ├── bootstrap/                     # operator-only state and identity root
│   └── production/                    # application infrastructure root
├── scripts/                           # narrow, repeatable operator helpers
├── src/                               # application source
├── tests/                             # behavior and policy tests
├── AGENTS.md                          # repository-specific agent guardrails
├── LICENSE
├── README.md
├── Dockerfile                         # only for containerized services
├── pyproject.toml or package.json
└── uv.lock or package-lock.json
```

Treat the tree as a menu, not a required skeleton. Application repositories
usually have `src/` and `tests/`; infrastructure-only repositories may have no
root package manager; documentation sites may have no Terraform; and local
CLI/data pipelines may keep generated operational evidence in dedicated
ignored directories.

Small repositories may keep the production Terraform root directly in
`infra/` rather than `infra/production/`. A service that separates state or
provider lifecycles may use roots such as `infra/bootstrap`, `infra/gcp`, and
`infra/cloudflare`. A shared foundation repository may keep documented
top-level provider roots and does not need to invent a bootstrap root for
itself. Choose roots by independent state, apply, ownership, and rollback
boundaries. See [Infrastructure repository architecture](infrastructure-repositories.md)
for the current ecosystem boundaries.

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

## Documentation as code

When a repository contains `mkdocs.yml` or another documentation build:

- Declare and lock the documentation toolchain.
- Build documentation on pull requests and relevant `main` changes using the
  strictest supported mode, such as `mkdocs build --strict`.
- Keep site name, repository name, repository URL, and navigation current.
- Put deployment, operations, infrastructure-contract, and recovery documents
  in navigation when operators are expected to find them through the site.
- Validate repository-relative and cross-repository links.
- Ignore generated site output.

After extracting or renaming a repository, audit workflows, scripts,
devcontainers, IDE configuration, documentation metadata and navigation,
cross-repository links, ownership assertions, and generated/local residue.
An extraction is not complete while the old repository name or an extracted
application remains an active local or CI dependency.

## Continuous integration baseline

Application CI should:

- Run on pull requests and on relevant pushes to `main`.
- Default to `permissions: contents: read`.
- Cancel superseded runs for the same branch or pull request.
- Use a bounded job timeout.
- Pin all externally sourced actions to full commit SHAs, with a version comment
  where useful.
- Install from the committed lockfile (`npm ci` or `uv sync --locked`).
- Run locked installation, tests, and every applicable static check. Do not add
  a formatter, linter, or type checker merely for symmetry when the project has
  not adopted one.
- Avoid production credentials and cloud identity.

Path filters are useful for expensive, specialized workflows. The general
application workflow may remain broad when the repository is small and every
change should prove the application still builds.

### Terraform CI

Infrastructure validation belongs in a separate, path-filtered workflow. The
target pattern is:

1. Check `terraform fmt -check -recursive infra`.
2. Initialize bootstrap and production roots with `-backend=false`.
3. Run `terraform validate` for each root.
4. Regenerate provider locks for every supported runner/developer platform,
   then run an explicit dirty-diff check so CI fails if the committed lockfiles
   change.
5. Run repository-specific infrastructure policy tests.
6. When configuration permits, create a credential-free speculative production
   plan on pull requests with refresh and locking disabled.

The speculative plan must use syntactically valid placeholders, omit the
remote backend, and receive no OIDC identity, remote-state access, or real
provider secrets. It proves configuration shape; it is not evidence of current
production state.

Some provider data sources and remote-state dependencies require live read
access during planning. Keep ordinary pull-request validation credential-free;
run any live preview from trusted `main` code with a separate read-only planning
identity. See [CI/CD trust and environment model](ci-cd.md) for the complete
pipeline and identity boundaries.

Policy tests should encode invariants that Terraform alone cannot express
clearly, such as exact resource ownership, deletion protection, least-privilege
IAM, bounded secret retention, immutable image requirements, or protected
state inventory.

Path filters must include the safety scripts, policy tests, lockfiles, and
workflow files that influence validation. Pull-request CI must use
`-backend=false`, placeholders, and no production identity or secrets. A live
remote-state plan is a distinct production-capable operation and must not be
presented as ordinary pull-request validation.

## Continuous delivery baseline

Production infrastructure applies are deliberately separate from pull-request
validation:

- Trigger with `workflow_dispatch` from `main`.
- Enforce `github.ref == 'refs/heads/main'` in the production job. A boolean
  dispatch confirmation is additive and is not a branch guard.
- Require a GitHub `production` environment restricted to `main`; configure
  required reviewers, self-review, and bypass rules to match the repository's
  actual approval model.
- Set `contents: read` and add `id-token: write` only for the deployment job.
- Authenticate with GitHub OIDC and a repository-specific service account;
  do not store cloud service-account keys.
- Serialize production applies and do not cancel an apply in progress.
- Validate required inputs before authentication or planning.
- Create a fresh saved plan inside the gated production job, review it through
  automated safety gates, and apply that exact plan file in the same job.
- Fail closed on deletion or replacement of protected resources.
- Finish with a drift check when the provider and service support a reliable
  immediate post-apply plan.

Merging a pull request is not equivalent to applying infrastructure. A manual
dispatch is an independent production decision.

When a repository uses a separate read-only live preview, require a successful
preview for the same `main` commit before production dispatch. Do not pass an
unencrypted saved plan through ordinary workflow artifacts; plans can contain
state and sensitive values. The production job must re-plan, compare the safe
action inventory with the preview, and stop for a new review if it materially
changed. The detailed model is defined in
[CI/CD trust and environment model](ci-cd.md).

### Multiple production roots

When one workflow owns multiple independently stateful roots, create and
safety-check every saved plan before applying the first one. Document the
apply order, cross-root interface, partial-failure behavior, and recovery path.
Afterward, verify every root independently. Terraform applies across roots are
not atomic.

### Scheduled production operations

Scheduled drift detection and narrowly controlled maintenance may be valid
exceptions to manual-only production execution. They must use the production
environment, repository-specific identity, an explicit timeout, and serialized
non-cancelling production concurrency.

A scheduled mutation additionally requires an exact saved plan, a narrow
resource/action allowlist, protected-state checks, and post-operation service
and persistent-data health verification. Terraform convergence alone does not
prove that the application or its data boundary is healthy. Drift workflows
must never apply and should fail visibly on both planning errors and detected
drift without exposing sensitive plan content.

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

### Container delivery

Containerized services should pin base images by digest, install locked
production dependencies, run as a non-root user, and maintain a narrow
`.dockerignore`. Pull-request CI should build the image without pushing it,
then execute that finished image in a credential-free acceptance check. The
check should verify:

- the effective runtime version matches the repository's declared version;
- the process runs as the intended non-root user;
- a representative application import or entrypoint executes successfully;
- package managers, compilers, or other tooling that the runtime-image
  contract says must be absent are actually unavailable.

A successful build is not sufficient evidence that the image can start. A
package manager may silently install a fallback runtime, generate an
inaccessible interpreter link, or leave version-specific build tooling behind.
Keep the acceptance check narrow: it should validate the artifact without
requiring production secrets or contacting production services.

Publication should use a commit-derived tag, resolve the pushed artifact to an
immutable digest, and deploy only that digest.

For Cloud Run services, use a health endpoint that does not end in `z`, such as
`/health`. Cloud Run reserves some paths ending in `z` and recommends avoiding
all of them. Keep the application route, startup/readiness probes, tests, and
post-deploy acceptance check on the same safe path. When diagnosing a platform
404, confirm whether the request reached container logs and test a non-reserved
path before changing application code or invocation IAM.

## Terraform and identity boundaries

For repositories that own cloud infrastructure, prefer two roots:

- **Bootstrap root:** operator-controlled; creates the dedicated state bucket,
  repository-scoped deployment identity, and their protections. Its state may
  live in a reusable foundation backend. GitHub Actions never applies it.
- **Production root:** owns only the application's declared resources. It uses
  the dedicated backend and repository-specific identity.

Keep state buckets and identities repository-specific even when a shared
Workload Identity provider or foundation bucket is reused. A repository that
needs a live production preview should use separate planning and deployment
identities: the planning identity reads only the repository's state and
required resource metadata, while the deployment identity retains only the
read/write permissions required by its roots. The planning identity must not
write state, mutate resources, change IAM, read unnecessary secret payloads, or
impersonate the deployment identity.

The bootstrap boundary must create federation trust and the IAM bindings that
authorize CI; the workflow being authorized must not manage those controls.
Treat a correct permission denial as evidence of the boundary, not a reason to
grant the workflow project-wide IAM administration. For remote state, validate
the provider's actual read calls: object read/list permissions may need a
narrow bucket metadata permission such as `storage.buckets.get`, while object
write/delete, IAM mutation, and impersonation remain denied.

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

Keep provider credentials specific to both repository and stage. Separate
`production-read` from `production`, avoid repository-level fallback secrets,
and rotate by proving the replacement through protected apply, service
acceptance, and no drift before retiring the old credential.

Do not commit personal or operator-specific values as Terraform defaults.
Require them as explicit inputs, store non-secret examples separately, and
keep secrets out of examples, plans, logs, and command history.

## GCP budget and cost ownership

Every GCP project must have an explicit budget decision when it is created and
before it incurs billable use. The repository that owns the project also owns
its project-scoped budget, notification path, verification, and response
procedure. Shared foundation infrastructure must not silently own application
budgets; it owns the budget for its own management project.

Commit the stable, non-secret decision in Terraform:

- a specified amount and calendar period, normally a monthly USD amount;
- project scope by default, with service filters only when the ownership and
  unmonitored remainder are explicitly understood;
- actual-spend thresholds that provide early, near-limit, and at-limit warning,
  such as 50%, 80% or 90%, and 100%;
- a 100% forecasted-spend threshold when the budget period supports it;
- a named notification channel and accountable recipient;
- the operational response at each threshold, including who may accept higher
  spend, reduce usage, or stop a service.

A service-filtered budget does not satisfy the project requirement unless the
repository documents why all other project spend is zero, separately budgeted,
or intentionally accepted.

The budget amount is a reviewed operating decision, not a mutable CI default.
Changes to it require the same review as other production policy changes. Keep
the billing-account identifier and notification recipient in the repository's
established sensitive or environment-specific configuration boundary.

An alerts-only Cloud Billing budget does not cap usage or spending. Treat an
automated billing disablement, quota reduction, or supported spend cap as a
separate availability-affecting control with an explicit owner, recovery path,
and acceptance test. See
[Google Cloud budget behavior](https://docs.cloud.google.com/billing/docs/how-to/budgets).

Bootstrap may establish the minimum permission needed to manage the budget,
but the application infrastructure root should own the budget resource. Scope
it to the exact project and prevent the repository identity from managing
unrelated project budgets wherever the platform permits. Policy tests should
verify the project filter, amount, thresholds, and notification configuration.
After apply, inspect the live budget and confirm alerts reach the intended
recipient; Terraform convergence alone does not prove notification delivery.

## Cross-repository baseline

Use these defaults unless a repository documents a reason to diverge:

| Concern | Standard |
| --- | --- |
| Commits | Conventional Commits; imperative summary of 72 characters or fewer |
| Runtimes | Declare the supported version and commit the dependency lockfile |
| GitHub Actions | Pin all externally sourced actions to full SHAs |
| Permissions | Declare the smallest workflow or job permissions explicitly |
| Credentials | Scope provider credentials by repository and protected stage; rotate with verified overlap |
| Dependencies | Weekly Dependabot updates; group minor/patch updates per lifecycle and keep majors separate |
| Secret scanning | Pull-request and `main` scanning, plus a scheduled full-history scan when appropriate |
| GCP cost ownership | Every project has a committed budget amount, thresholds, notification owner, and response procedure |
| Generated/local files | Ignore virtual environments, dependencies, build output, Terraform working data, plans, state, and local secret files |
| Infrastructure | Format, validate, policy-test, and plan before apply |
| Releases | Deploy immutable artifacts where the platform supports them |
| Verification | Distinguish built, published, applied, and verified states |

Keep runtime declarations consistent across package metadata, version files,
CI, containers, and deployment platforms. Repository ignores must be portable:
do not rely on a developer's global Git excludes for `.DS_Store`, virtual
environments, tool caches, Terraform plans/state, or local secret files.

If a skill or configuration file must be mirrored for more than one consumer,
declare one canonical source and add a deterministic equality check. For
security automation, document whether scanning is owned by a committed
workflow or by a GitHub-native repository setting and make its current status
verifiable.

### Dependabot configuration

Commit `.github/dependabot.yml` in every maintained repository. Configure one
weekly update entry for GitHub Actions and one entry for each package ecosystem
and lifecycle actually present in the repository. Use the native updater for
the committed lockfile: for example, `npm` for `package-lock.json`, `uv` for
`uv.lock`, `docker` for a `Dockerfile`, and `terraform` for Terraform roots.

Use these grouping rules:

- Put all routine GitHub Actions minor and patch updates in a stable
  `github-actions` group.
- Group minor and patch updates within each repository lifecycle, such as
  application, documentation, container, or Terraform dependencies.
- Keep major updates as individual pull requests so migrations, compatibility
  checks, and live-validation requirements remain explicit.
- Do not combine ecosystems merely to reduce pull-request count. A runtime
  upgrade and a production-provider upgrade have different failure and rollback
  boundaries even when both are maintained in the same repository.
- Use `build(deps)` as the Dependabot commit-message prefix so generated commits
  and pull-request titles follow the Conventional Commits baseline.

Cover every manifest location. Use `directory: "/"` for GitHub Actions and
root-level package managers. Use `directories` to enumerate all independently
maintained Terraform or monorepo roots; adding a new root is incomplete until
Dependabot covers it. Keep group identifiers stable and descriptive because
they appear in generated branch names and pull-request titles.

A repository with npm application dependencies and two Terraform roots should
follow this shape:

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    groups:
      github-actions:
        patterns: ["*"]
        update-types: ["minor", "patch"]
    commit-message:
      prefix: "build(deps)"

  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    groups:
      npm-dependencies:
        patterns: ["*"]
        update-types: ["minor", "patch"]
    commit-message:
      prefix: "build(deps)"

  - package-ecosystem: "terraform"
    directories:
      - "/infra/bootstrap"
      - "/infra/production"
    schedule:
      interval: "weekly"
    groups:
      terraform-dependencies:
        patterns: ["*"]
        update-types: ["minor", "patch"]
    commit-message:
      prefix: "build(deps)"
```

Adapt the ecosystems, group names, and directories to the repository rather
than retaining unused template entries. Validate the file as YAML before
publication, but treat GitHub's successful Dependabot processing on the default
branch as the acceptance check; a successful YAML parse does not prove that
GitHub accepts the option layout or can resolve every manifest.

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
| Dependency/security automation | grouped weekly Dependabot | grouped weekly Dependabot | keep ecosystem coverage current and add a verifiable secret-scanning owner |

These differences are not all defects. Converge on the safety and ownership
contracts first; standardize filenames and workflow presentation when doing so
does not obscure a real platform difference.

## New-repository checklist

- [ ] Write the README entry point and choose the required `/docs` files.
- [ ] Commit runtime metadata, a lockfile, `.gitignore`, and an explicit license
      decision.
- [ ] Add focused source and test directories.
- [ ] If documentation has a build system, lock it and run a strict build in
      pull requests.
- [ ] Add application CI with read-only permissions, concurrency, a timeout,
      locked dependencies, and SHA-pinned actions.
- [ ] For a production container, build and execute the finished image in CI;
      verify its runtime, non-root identity, application import or entrypoint,
      and removal of prohibited tooling.
- [ ] For Cloud Run, use a health path that does not end in `z` and align the
      route, probes, tests, and post-deploy verification.
- [ ] Add weekly Dependabot entries for GitHub Actions and every native package
      ecosystem; group minor/patch updates per lifecycle and keep majors
      separate.
- [ ] Confirm GitHub accepts the Dependabot configuration and recognizes every
      intended manifest directory.
- [ ] Add secret scanning appropriate to the repository.
- [ ] If infrastructure is owned here, define the ownership contract before
      authoring Terraform.
- [ ] For every owned GCP project, commit the decided budget, thresholds, alert
      recipient, and threshold-response procedure; verify the live budget and
      notification path.
- [ ] Separate operator-only bootstrap from production infrastructure.
- [ ] Keep federation trust and its authorizing IAM bindings outside the
      workflow they authorize; test required reads and representative denied
      mutations.
- [ ] Add policy tests and a credential-free pull-request plan where the
      configuration permits one; otherwise keep PR validation credential-free
      and use a trusted read-only live preview.
- [ ] Ensure CI path filters include every safety script, policy test, lockfile,
      and workflow that influences the result.
- [ ] Keep production identity and secrets out of pull requests.
- [ ] Scope provider credentials by repository and protected stage; verify the
      replacement before retiring an old credential.
- [ ] If live planning needs production access, separate the read-only planning
      identity from the read/write deployment identity and test the denied
      permissions.
- [ ] Restrict production jobs in both workflow conditions and GitHub
      environment deployment-branch rules.
- [ ] For multiple production roots, plan and gate all roots before applying
      any root.
- [ ] Give scheduled production mutations an allowlist and application/data
      health verification.
- [ ] Run post-apply checks from a vantage that traverses the real edge policy;
      require fresh, bounded evidence without weakening WAF or bot controls.
- [ ] For sensitive authenticated services, verify the exact interface and a
      minimal real operation without recording sensitive payloads.
- [ ] Document the application deployer, infrastructure deployer, migrations,
      verification, rollback, and destructive stop conditions.
- [ ] Run the local commands from the README and the same checks CI will run.
