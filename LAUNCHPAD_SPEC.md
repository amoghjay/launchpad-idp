# Launchpad IDP: Comprehensive Project Specification & Implementation Guide

**Version**: 0.1.0-alpha
**Created**: January 21, 2026
**Status**: Ready for Implementation
**Estimated Time to MVP**: 40–50 hours
**Estimated Time to v0.1 Release**: 60–70 hours

---

## Table of Contents
1. Executive Summary
2. Project Vision & Goals
3. Architecture Overview
4. Complete Tech Stack
5. Implementation Phases
6. Directory Structure
7. Detailed Phase Breakdown
8. Security Strategy
9. Success Criteria
10. Setup Instructions
11. Common Pitfalls & Mitigations
12. Learning Resources
13. Hiring Narrative

---

## Executive Summary
Launchpad IDP is a Kubernetes-native Internal Developer Platform that automates "Day 0" (project scaffolding) and "Day 1" (deployment) operations. Developers run `lp create service <name>` to get a production-ready repo with GitOps, mTLS, and NetworkPolicies baked in. ArgoCD handles GitOps, Kargo manages promotion, cert-manager enforces mTLS, and a golden-path Helm library keeps services consistent.

---

## Project Vision & Goals
- Reduce service onboarding from days to minutes via a one-command scaffold.
- Enforce secure-by-default posture: mTLS, default-deny NetworkPolicies, non-root pods.
- Provide a validated promotion path Dev → Staging → Prod with auditable gates.
- Offer a developer-friendly CLI and templates so teams focus on business logic.
- Demonstrate platform engineering thinking for resume/portfolio.

Learning outcomes: ArgoCD ApplicationSets, Kargo promotion, cert-manager CSI mTLS, NetworkPolicies (default-deny), Helm library charts, GitHub Actions-driven GitOps.

---

## Architecture Overview
Developer runs CLI → commits scaffold to GitHub → ArgoCD syncs → Kargo promotes across stages → cert-manager injects mTLS certs → NetworkPolicies enforce least privilege.

Environments: Minikube (Dev/Staging/Prod emulation). Ingress Nginx terminates HTTP; mTLS used pod-to-pod.

---

## Complete Tech Stack
- Cluster: Minikube (docker driver, 4 CPU, 8GB RAM), Kubernetes 1.28+
- GitOps: ArgoCD v2.10+, ApplicationSets; Kargo v1.1+ for promotion; GitHub Actions
- Security: cert-manager v1.14+ + CSI driver; Trust-Manager; NetworkPolicies (Calico/Minikube CNI)
- Charts: Helm 3.12+, Golden-path library chart + per-app charts
- CLI: Python 3.11+ (Click, Jinja2) or Go (Cobra)
- Networking: Ingress Nginx v1.10+
- Observability (v0.2+): Prometheus, Grafana, OTEL Collector, Loki (optional)

---

## Implementation Phases (with estimates)
- Phase 1 (8h): Infrastructure — Minikube + Ingress, DNS, bootstrap scripts, Makefile helpers.
- Phase 2 (12h): ArgoCD control plane — Helm install, RBAC, Ingress, ApplicationSet for platform, GitHub integration.
- Phase 3 (10h): Kargo + first tenant — Warehouse, Dev/Staging/Prod stages, golden-path library chart, example FastAPI tenant chart.
- Phase 4 (12h): Security — cert-manager + CSI, self-signed CA (dev), mTLS volume mounts, default-deny NetworkPolicies + allowlists, pod securityContext.
- Phase 5 (8h): CLI (`lp`) — `init`, `create service`, optional `deploy`; Jinja2 templates; tests.
- Phase 6 (5h): Docs & release — README, ARCHITECTURE, SETUP, ONBOARDING, CONTRIBUTING, LICENSE, v0.1.0 release notes.
- Buffer (5h): Debugging/polish.

---

## Directory Structure (target)
launchpad-idp/
├── README.md
├── ARCHITECTURE.md
├── SETUP.md
├── bootstrap/ (cluster scripts)
├── platform/
│   ├── argocd/ (values, ingress, rbac, applicationsets)
│   ├── kargo/ (values, warehouse, stages)
│   └── security/ (cert-manager, issuers, trust-manager, network-policies)
├── templates/
│   ├── launchpad-app-library/ (library chart: deployment/service/ingress/networkpolicy/hpa)
│   └── example-apps/webapp-fastapi/
├── tenants/ (dev/staging/prod namespaces/apps)
├── cli/ (lp tool, templates, tests)
├── docs/ (architecture, security, troubleshooting, faq)
├── tests/ (integration: argocd, kargo, security; e2e)
└── .github/workflows/ (ci, integration, release)

---

## Detailed Phase Breakdown (highlights)

### Phase 1: Infrastructure
- Minikube start: `minikube start --cpus 4 --memory 8192 --driver docker --addons ingress`
- /etc/hosts entries: launchpad.local, argocd.local, kargo.local, myapp.local → 127.0.0.1
- Scripts: `bootstrap/minikube-setup.sh`, `bootstrap/cleanup.sh`; Makefile targets: setup/clean/test-dns.
- Success: `curl http://launchpad.local` → 404 (Ingress default), not connection refused.

### Phase 2: ArgoCD Control Plane
- Helm install with overrides (insecure mode, AppSets enabled, RBAC policy).
- Ingress at argocd.local.
- RBAC roles: readonly, developer (sync own apps), admin.
- ApplicationSet to deploy platform components.
- (Optional) GitHub Deployments + webhook for status reporting.

### Phase 3: Kargo + Tenant
- Library chart (type: library) with deployment, service, ingress, networkpolicy, hpa, configmap.
- Example app (FastAPI) chart depends on library chart; values override image/tag/env.
- Kargo Warehouse subscribes to Git commits + container tags.
- Stages: Dev (auto), Staging (approval gate), Prod (manual approval).
- Namespaces: tenants-dev/staging/prod; ArgoCD Applications per stage.

### Phase 4: Security (mTLS + NetworkPolicies)
- Install cert-manager + CSI driver; create self-signed CA (dev) and ClusterIssuer.
- Deployment template mounts CSI volume at /etc/tls; serviceAccount per app; non-root securityContext.
- NetworkPolicies: default-deny ingress/egress; allow from ingress-nginx; allow DNS egress.
- Tests: verify certs mounted, mTLS handshake, DNS allowed, random egress blocked.

### Phase 5: CLI (`lp`)
- Python package with Click entrypoint; commands: `lp init`, `lp create service <name>`, (optional) `lp deploy <env>`.
- Jinja2 scaffold templates for Chart.yaml, values, Dockerfile, main.py/main.go.
- Git operations: init repo, first commit, optional push to GitHub.
- Tests: scaffold exists, helm lint passes, basic CLI unit tests.

### Phase 6: Docs & Release
- README (value prop, quick start), ARCHITECTURE (design/diagrams), SETUP (30-min path), ONBOARDING (new service walkthrough), security-design, troubleshooting/FAQ, CONTRIBUTING, LICENSE (Apache 2.0).
- Tag v0.1.0 with release notes; note limitations and roadmap.

---

## Security Strategy
- mTLS via cert-manager CSI: pod certs from ClusterIssuer; Trust-Manager distributes CA; apps read /etc/tls.
- NetworkPolicies: default-deny; allow ingress from ingress-nginx; allow DNS; allow specific services as needed.
- Pod securityContext: runAsNonRoot, runAsUser 1000, fsGroup 1000, readOnlyRootFilesystem, no privilege escalation.
- RBAC: readonly/developer/admin roles; future Dex/OIDC + groups.

---

## Success Criteria
**MVP (Phases 1–4):**
- Minikube + Ingress up; `make setup` works.
- ArgoCD syncing platform repo; ApplicationSet deploys components.
- Kargo promotes Dev → Staging (approval) → Prod (manual) using Warehouse freight.
- Example FastAPI app deployed; reachable at myapp.local.
- mTLS certs injected; NetworkPolicies block unauthorized traffic; security tests pass.
- SETUP.md gets a new user from 0 → working platform in <30 minutes.

**v0.1 (adds Phase 5–6):**
- `lp create service` scaffolds a new app; ArgoCD deploys it automatically to Dev.
- CLI + chart lint/tests pass.
- Docs complete; release tagged v0.1.0.
- Portfolio-ready narrative prepared.

---

## Setup Instructions (quick)
```bash
make setup
helm install argocd argo/argo-cd -n argocd -f platform/argocd/values-override.yaml
helm install kargo kargo/kargo -n kargo -f platform/kargo/values-override.yaml
# add GitHub repo to ArgoCD; apply ApplicationSets; apply Kargo warehouse/stages
```
Detailed steps: see SETUP.md (to be authored).

---

## Common Pitfalls & Mitigations
- DNS not resolving: ensure /etc/hosts entries; `minikube tunnel` running.
- ArgoCD repo auth failures: PAT scope (repo + read:org); verify with `argocd repo list`.
- NetworkPolicies block everything: start with allow-all, then enable default-deny + specific allowlists.
- mTLS failures: confirm CSI driver, issuer name, and /etc/tls contents; check pod serviceAccount.
- Kargo not promoting: check Stage status/events; approvals; Warehouse freight detected.

---

## Learning Resources
- ArgoCD docs; Kargo docs; cert-manager docs; Kubernetes docs.
- Books: Kubernetes in Action; GitOps and Kubernetes; Cloud Native Security Cookbook.
- Labs: KodeKloud; A Cloud Guru; official K8s bootcamp.

---

## Hiring Narrative (use in interviews)
"I built Launchpad IDP, an open-source Kubernetes-native Internal Developer Platform. Developers run a single CLI command to scaffold services that inherit a golden-path Helm library. ArgoCD provides GitOps, Kargo handles promotion (Dev→Staging→Prod), cert-manager injects mTLS certs, and NetworkPolicies enforce default-deny. This reduces onboarding from days to minutes and standardizes security across services. I designed and implemented the architecture, security, and developer experience end-to-end."

---

## Roadmap (post-v0.1)
- v0.2: Observability stack (Prometheus, Grafana, OTEL, Loki), alerting, dashboards.
- v0.3: Advanced delivery (Flagger canary, blue/green), A/B testing.
- v0.4: Multi-cluster/federation, DR/backup, capacity planning, cost attribution.
- v0.5: SSO/RBAC with Dex/OIDC, audit logging, compliance reporting.

---

## Version History
- 0.1.0 (2026-01-21): Initial comprehensive spec for Launchpad IDP.
