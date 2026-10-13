# Infrastructure repository architecture

The former `rocjay1-monorepo` has been decomposed into repositories with
independent application, deployment, secret, state, and rollback lifecycles.
The retired repository name must not be reused.

| Repository | Ownership boundary | GCP project and state |
| --- | --- | --- |
| [`core-infra`](https://github.com/rocjay1/core-infra) | Default-project foundation, shared GitHub federation, Cloudflare account and zone policy, shared DNS, and WAF | Default project; foundation bucket prefixes `core-infra/gcp` and `core-infra/cloudflare` |

## Shared interfaces

`core-infra` is currently the only infrastructure repository. It owns the
shared Cloudflare WAF, shared DNS, and the shared GitHub Workload Identity
Federation pools. No application repository currently has a provider in those
pools, and the WAF carries no application-specific allowances. See the
[`core-infra` documentation](https://github.com/rocjay1/core-infra/blob/main/docs/index.md)
for the current boundary.

When an application repository is added, any value it shares with `core-infra`
must be an explicit, reviewed cross-repository interface. `core-infra` must not
read another repository's state, and the application repository owns its own
application DNS records and runtime resources unless this page records
otherwise.

An opaque value shared across repositories—such as a verification header or
origin token—is a versioned interface even when both repositories use the same
secret name. Document its producer, consumers, rotation order, rollback owner,
and downstream acceptance check. During rotation, compare one-way fingerprints
without printing the value, update every owner in a coordinated change, and
verify a fresh provider-side health signal. Name equality is not evidence that
the stored values match.

GitHub Actions deployments authenticate through the shared Workload Identity
pool using immutable GitHub repository IDs. Each application repository that
joins the pool receives a repository-specific deployment identity and may
manage only its declared project resources and state. Operator-only bootstrap roots create projects,
state buckets, and deployment identities; production workflows do not apply
bootstrap state.

Provider credentials are repository- and stage-specific. Keep read-only
planning credentials separate from deployment credentials, store them in the
owning GitHub environment, and do not preserve a broad shared token as a
repository-level fallback. A rotation is complete only after the replacement
passes the protected workflow and the retired credential is no longer used.

## Change rule

Move a resource only when its lifecycle and rollback owner move with it. A
cross-repository dependency must be documented as a narrow interface, not
implemented by granting one repository broad access to another repository's
state or project.
