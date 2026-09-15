# Deployment target stays cloud-agnostic; identity federates to existing enterprise SSO

Two deployment-level decisions, made together because they both bound the deployment architecture without over-committing to infrastructure specifics that aren't decided yet:

1. **Cloud-agnostic deployment target.** CBUAE has not yet committed to a specific platform (on-prem VMs, on-prem Kubernetes/OpenShift, or a government-approved cloud tenancy). Rather than guess, the deployment architecture is designed around a generic container-orchestration boundary (a "region" in deployment terms) that any of those targets can satisfy. Revisit this ADR once the actual platform is chosen — it will likely stay compatible, since the design deliberately avoids platform-specific primitives.
2. **Identity federates to CBUAE's existing SSO** (Azure AD / on-prem AD — whichever CBUAE already runs) via OIDC/SAML, rather than the pipeline maintaining its own user store. The Examiner/Lead-Approver RBAC roles (decision 35) are authorization claims layered on top of federated identity, not new accounts.

Both are worth recording because a future engineer, faced with an unspecified deployment target, might default to a specific cloud vendor's managed services (locking in vendor-specific primitives) or stand up a local user/password store (duplicating identity CBUAE already manages) — either would be a real regression from what's intended here.
