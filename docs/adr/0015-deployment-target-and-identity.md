# Deployment target is on-prem for now; identity federates to existing enterprise SSO

## Status

Accepted — 2026-09-15

## Context

Two deployment-level decisions, made together because they both bound the deployment architecture without over-committing to infrastructure specifics beyond what's actually settled.

## Decision

1. **On-prem, for now.** CBUAE has not committed to a specific on-prem platform (VMs vs. Kubernetes/OpenShift), but has decided the pipeline stays on-prem rather than moving to any cloud tenancy — a deliberate, explicit call, not a placeholder. The deployment architecture is still designed around a generic container-orchestration boundary (a "region" in deployment terms) rather than platform-specific primitives, so a future move to Kubernetes/OpenShift (or, if it ever changes, to a cloud tenancy) doesn't require redesigning the shape — only this ADR needs revisiting.
2. **Identity federates to CBUAE's existing SSO** (Azure AD / on-prem AD — whichever CBUAE already runs) via OIDC/SAML, rather than the pipeline maintaining its own user store. The Examiner/Lead-Approver RBAC roles (decision 35) are authorization claims layered on top of federated identity, not new accounts.

## Consequences

Both are worth recording because a future engineer might otherwise default to a specific platform's managed services (locking in primitives the on-prem decision doesn't support) or stand up a local user/password store (duplicating identity CBUAE already manages) — either would be a real regression from what's intended here. The specific on-prem platform (VMs vs. K8s/OpenShift) and the specific IdP remain open items, tracked in `docs/decision-log.md`.
