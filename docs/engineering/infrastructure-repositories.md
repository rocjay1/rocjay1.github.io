# Infrastructure repository architecture

The former `rocjay1-monorepo` has been decomposed into repositories with
independent application, deployment, secret, state, and rollback lifecycles.
The retired repository name must not be reused.

| Repository | Ownership boundary | GCP project and state |
| --- | --- | --- |
| [`core-infra`](https://github.com/rocjay1/core-infra) | Default-project foundation, shared GitHub federation, Cloudflare account and zone policy, shared DNS, and WAF | Default project; foundation bucket prefixes `core-infra/gcp` and `core-infra/cloudflare` |
| [`signal-studio`](https://github.com/rocjay1/signal-studio) | Signal Studio application plus its Gemini credential, secret, notification channel, budget, and production infrastructure | `signal-studio-00c111`; dedicated versioned production state bucket and operator-only foundation bootstrap state |
| [`reading-stack-infra`](https://github.com/rocjay1/reading-stack-infra) | Miniflux host, persistent storage, tunnel, runtime identity, monitoring, and backups | Miniflux project; repository-owned GCP and Cloudflare state |
| [`email-to-rss-bridge`](https://github.com/rocjay1/email-to-rss-bridge) | Email Worker, D1 storage, authenticated feed, and application delivery | Repository-owned Cloudflare infrastructure and state |

## Shared interfaces

`core-infra` owns the shared Cloudflare WAF. It consumes the reading stack's
stable public app-host IP and token-gated ingress values as an explicit,
reviewed cross-repository interface; it does not read the reading-stack state.
The reading stack owns its tunnel and application DNS records. The email
bridge owns its Worker and routing rule.

All GitHub Actions deployments authenticate through the shared Workload
Identity pool using immutable GitHub repository IDs. Each application has a
repository-specific deployment identity and may manage only its declared
project resources and state. Operator-only bootstrap roots create projects,
state buckets, and deployment identities; production workflows do not apply
bootstrap state.

## Change rule

Move a resource only when its lifecycle and rollback owner move with it. A
cross-repository dependency must be documented as a narrow interface, not
implemented by granting one repository broad access to another repository's
state or project.
