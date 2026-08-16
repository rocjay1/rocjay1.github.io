# CI/CD trust and environment model

This standard separates validation, live production inspection, and mutation.
The central rule is that pull-request code must not receive production access,
while a production apply must use a freshly generated, safety-checked plan.

## Environments describe targets; identities describe capability

Create a GitHub environment only when it represents a durable trust or
deployment boundary:

- **`production-read`:** optional boundary for a trusted live production plan.
  It may obtain a repository-specific read-only planning identity, but it may
  not mutate resources or state.
- **`production`:** required boundary for production mutation. It obtains the
  repository-specific deployment identity only after its protection rules pass.
- **`development`:** create this only when a real development stack exists with
  its own state, resources, secrets, deployment lifecycle, and rollback path.
  Do not use “development” as a label for reading production.

A repository without a deployed development stack normally needs
`production-read` and `production`, not `development` and `production`.

## Pipeline stages

| Stage | Trigger and ref | GitHub environment | Cloud identity | May mutate? | Evidence produced |
| --- | --- | --- | --- | --- | --- |
| Pull-request validation | `pull_request` | none | none | no | formatting, validation, policy tests, and configuration-shape plan where feasible |
| Live production preview | trusted manual run or trusted `main` workflow | `production-read` | repository planning identity | no | current-state plan summary and safety result |
| Production apply | `workflow_dispatch` on `main` | `production` | repository deployment identity | yes | fresh saved plan, safety decision, apply result |
| Post-apply verification | same serialized production run | `production` | deployment identity or narrower verifier | no further mutation | service checks and drift result |

These stages are distinct states. A successful pull-request plan does not prove
current production state, and an applied Terraform plan does not prove that the
application or its persistent data is healthy.

## Pull-request validation

Pull-request workflows must remain safe for untrusted code:

- Use `permissions: contents: read` and do not grant `id-token: write`.
- Do not reference a production environment or receive production secrets.
- Format and validate every Terraform root with the backend disabled.
- Regenerate provider locks for every supported platform and fail on a dirty
  diff.
- Run policy tests for ownership, IAM, deletion protection, state protections,
  and other repository invariants.
- When configuration permits, run a credential-free speculative plan with
  syntactically valid placeholders, `-backend=false`, `-refresh=false`, and
  `-lock=false`.

Credential-free planning is preferred but not universal. Provider data sources
and remote-state dependencies can require live read access during planning. Do
not grant an identity to ordinary pull-request code to rescue such a plan.
Retain the validation and policy stages, then run the live preview from trusted
`main` code.

## Live production preview

Use a separate planning identity when a useful review depends on current
production state:

- Run only trusted code from `main`; enforce this in both the job condition and
  the `production-read` environment deployment-branch policy.
- Grant `id-token: write` only to the preview job and authenticate after local
  validation succeeds.
- Read the real backend and provider metadata with refresh enabled. Disable
  state locking when the backend lock requires write access; the result is an
  advisory preview and may become stale.
- Run the same protected-resource and destructive-action policies used by the
  production job.
- Publish only a deliberately sanitized summary: commit SHA, root, action
  counts, policy result, and a canonical inventory of resource addresses and
  actions when those addresses are safe to disclose.
- Delete the local plan and working data when the job ends.

Do not print full plan JSON, state, secret values, or sensitive provider output
to logs. Do not upload an unencrypted saved plan as a GitHub Actions artifact.
Repository readers can download workflow artifacts, and Terraform plan files
can contain the full configuration, state, and sensitive values.

The planning identity is useful evidence, not production authorization. A
successful preview never applies infrastructure and never turns a pull request
into a deployment decision.

### Trust bootstrap stays outside the workflow it authorizes

A workflow identity must not create or modify the federation provider, trust
condition, service-account impersonation binding, or other IAM grant that
authorizes that same workflow. Those resources belong to an operator-controlled
bootstrap boundary. If a least-privilege workflow correctly fails on such a
resource, do not widen its role to make the plan green: reconcile or import the
bootstrap-owned resource outside the workflow, then rerun the preview.

Enumerate the provider operations needed for read-only planning instead of
assuming that object read access is sufficient. A remote-state backend can
require both object read/list permissions and bucket metadata access such as
`storage.buckets.get`. Grant any missing capability at the narrowest resource
scope, then prove both the required reads and representative denied writes with
effective-permission tests.

## Production apply

The production workflow must:

1. Require `workflow_dispatch` and enforce
   `github.ref == 'refs/heads/main'` on every production-capable job.
2. Require the `production` environment before requesting OIDC credentials.
3. Confirm that a successful live preview exists for the same commit when the
   repository uses `production-read`.
4. Serialize production runs and never cancel an apply in progress.
5. Authenticate as the repository deployment identity only after inputs and
   the checked-out commit are validated.
6. Initialize the real backend with locking enabled and create a fresh saved
   plan.
7. Run destructive-action, protected-resource, ownership, and policy gates on
   that plan.
8. Compare its non-sensitive action inventory with the reviewed preview. Stop
   for a new review when the actions materially differ.
9. Apply that exact saved plan file in the same job and runner context.
10. Verify the service, persistent-data boundary, and expected resource state;
    run a post-apply drift check when it is reliable.

Creating and applying the saved plan in one job avoids transferring a sensitive
plan between trust boundaries. If a repository instead separates those steps,
it must use a separately reviewed protected-artifact mechanism that preserves
confidentiality, integrity, runner architecture, filesystem layout, Terraform
version, and provider versions. Ordinary artifacts in a public repository do
not meet that requirement.

## Post-apply service evidence

Post-apply verification must test the deployed service and its persistent-data
boundary, not only Terraform convergence.

- Use a health path the hosting platform will deliver to the application. On
  Cloud Run, avoid every path ending in `z`; use `/health` and keep the
  application route, container probes, tests, and deployment check aligned.
  A platform 404 that is absent from request logs is not application evidence;
  confirm that a non-reserved endpoint reaches the container.
- Run an automated check from a network vantage that can traverse the real
  edge policy. Do not weaken WAF or bot protections merely to admit GitHub-hosted
  runners. When those runners are intentionally challenged, use a bounded
  provider-side check such as a monitoring probe or browser run.
- Require evidence newer than the apply start or completion cutoff. Poll with a
  bounded timeout and verify the minimum stable status, body, data, and drift
  invariants. Document provider quotas and cost when the verifier is billable.
- For OAuth-protected or privacy-sensitive services, keep CI assertions to
  non-sensitive metadata and boundaries. Complete a manual acceptance check
  through an existing authorized grant when necessary: verify the exact tool or
  interface inventory, one minimal real operation, refresh continuity, and log
  privacy. Record only status, schema, and cardinality—not payloads, locations,
  library contents, health records, or tokens.

## Identity boundaries

Create repository-specific identities; do not share a deployment identity
across applications.

| Capability | Planning identity | Deployment identity |
| --- | --- | --- |
| Remote state | read objects; no create, update, delete, or lock writes | read and write only the repository's production state |
| Managed resources | read only the metadata required by providers and data sources | only the read/write permissions required by the declared Terraform roots |
| Secrets | no secret-payload access unless an explicit, reviewed plan requirement proves it unavoidable | access only to secrets whose lifecycle or deployment requires it |
| IAM | cannot change policies or impersonate the deployment identity | only narrowly required IAM changes; no broad project administration |
| Workload Identity | impersonable only from the trusted preview job | impersonable only from the protected production job |

Restrict Workload Identity Federation with immutable repository ownership or
repository identifiers plus the expected branch, environment, and workflow
claims where the provider supports them. The GitHub environment restriction and
cloud trust condition should independently reject an unexpected ref.

Verify least privilege with negative tests. The planning identity must fail to
write state, mutate a representative resource, read prohibited secret payloads,
or impersonate the deployment identity. The production identity must fail
outside the repository's declared resources and state.

## GitHub environment protections

For `production-read`:

- allow only `main`;
- store only the non-secret coordinates needed to select the planning identity;
- require a reviewer when production inventory is sensitive enough to justify
  the additional gate.

For `production`:

- allow only `main`;
- require reviewers when the repository's operating model has an independent
  reviewer;
- prevent self-review and administrative bypass when an independent approval
  is required and the GitHub plan supports those controls;
- keep production secrets and identity coordinates scoped to the environment.

Credentials must be repository-specific and stage-specific. A read-only
preview must not inherit the deployment credential, and an application must
not fall back to a broad repository-level credential when its protected
environment value is absent. During rotation, retain the old credential until
the replacement passes the protected apply, service checks, and no-drift check;
then revoke it and verify that no active workflow still references it.

In a single-operator repository, an environment approval may still provide a
deliberate pause, but it is not independent review. Document that limitation
rather than claiming separation of duties.

## Real development stacks

When development is a real deployment target, give it independent state,
secrets, identities, concurrency, verification, and rollback. A development
deployment identity may write only development resources; it must not read or
write production state. Promote immutable application artifacts between
environments rather than rebuilding different artifacts for each target.

Development success is useful evidence, but it does not replace a fresh
production plan or production approval.

## Workflow invariants

Every workflow should:

- pin all externally sourced actions to full commit SHAs with version comments;
- declare minimum token permissions explicitly;
- use bounded timeouts and concurrency appropriate to the stage;
- install from committed lockfiles;
- include workflows, policy scripts, lockfiles, and infrastructure roots in
  relevant path filters;
- distinguish built, published, previewed, approved, applied, and verified
  states in names and summaries;
- keep plans, state, credentials, and sensitive command output out of caches,
  logs, artifacts, and version control.

When a repository must diverge, document the unmet constraint, compensating
control, acceptance evidence, and rollback owner in its deployment or
operations guide.

## Implementation references

- [GitHub deployment environments and protection rules](https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments)
- [GitHub OIDC claims and trust conditions](https://docs.github.com/en/actions/reference/security/oidc)
- [Google Cloud Workload Identity Federation for deployment pipelines](https://docs.cloud.google.com/iam/docs/workload-identity-federation-with-deployment-pipelines)
- [Cloud Run reserved URL paths](https://docs.cloud.google.com/run/docs/known-issues#url)
- [Cloud Storage IAM roles](https://docs.cloud.google.com/storage/docs/access-control/iam-roles)
- [Cloudflare Bot Fight Mode limitations](https://developers.cloudflare.com/bots/get-started/bot-fight-mode/#limitations)
- [Terraform automation and saved-plan boundaries](https://developer.hashicorp.com/terraform/tutorials/automation/automate-terraform)
- [Terraform plan-file sensitivity](https://developer.hashicorp.com/terraform/cli/commands/plan)
- [GitHub workflow artifact access](https://docs.github.com/en/actions/how-tos/manage-workflow-runs/download-workflow-artifacts)
