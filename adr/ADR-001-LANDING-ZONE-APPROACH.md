# ADR-001: Landing Zone Approach

**Status:** Accepted  
**Date:** 2026-01-27  
**Decision makers:** Platform Team  

---

## Context

LOCAL-PLUS needs an AWS multi-account strategy for a Gift Card & Loyalty Platform with SOC2, PCI-DSS, GDPR compliance requirements.

## Decision

**Hybrid approach: Control Tower + Terraform**

> Control Tower comme fondation, Terraform comme langage.

## Rationale

### The real question

> *"Qui porte la responsabilité légale, sécurité et audit ?"*

- **Org-wide, security baseline, audit** → Managed AWS (Control Tower)
- **Produit, métier, plateforme** → Terraform pur

### Why not Pure Terraform?

| Risk | Impact |
|------|--------|
| Erreur SCP | Blast radius = toute l'org |
| Oubli CloudTrail | Non-compliance, incident invisible |
| S3 log mal configuré | Audit failure |
| Migration tardive vers CT | 2-4 semaines, risque élevé |

> *"Le Terraform-only est intellectuellement pur mais stratégiquement risqué."*

### Why not Control Tower only?

| Issue | Impact |
|-------|--------|
| Pas Git-first | Platform Team friction |
| Black box | Debugging difficile |
| AFT interne | CodePipeline géré par AWS, invisible pour nous |

> *"Control Tower est imparfait mais politiquement et légalement puissant."*

### Compliance = langage commun

Un audit est un **exercice social**, pas technique.

| Question auditeur | Avec Control Tower |
|-------------------|-------------------|
| "Comment vous gérez les logs ?" | "Control Tower, Log Archive account" |
| "Vos guardrails ?" | "AWS managed controls + custom SCPs" |
| "Drift detection ?" | "AWS Config + Control Tower dashboard" |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    LAYER 0 — CONTROL TOWER                          │
│                    (Managed, Immutable, Audit)                       │
├─────────────────────────────────────────────────────────────────────┤
│  • AWS Organizations                                                 │
│  • SCPs globales (AWS managed + custom via Terraform)               │
│  • CloudTrail org-level                                              │
│  • AWS Config                                                        │
│  • Security Hub                                                      │
│  • Log Archive Account                                               │
│  • Audit Account                                                     │
│  ⛔ Aucune logique produit ici                                      │
└─────────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    AFT — ACCOUNT FACTORY                             │
│                    (GitHub Actions → Terraform → AFT)                │
├─────────────────────────────────────────────────────────────────────┤
│  • Account requests via Git PR                                       │
│  • GitHub Actions exécute Terraform                                  │
│  • Terraform appelle AFT module                                      │
│  • AFT provisionne compte + baseline                                 │
│  ⛔ Pas de logique métier                                           │
└─────────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    LAYER 1+ — TERRAFORM PUR                          │
│                    (Platform, GitOps, GitHub Actions)                │
├─────────────────────────────────────────────────────────────────────┤
│  • VPC / Networking                                                  │
│  • EKS / ECS                                                         │
│  • RDS / Kafka / Cache                                               │
│  • IAM métiers (IRSA)                                               │
│  • Observabilité                                                     │
│  • Everything business-facing                                        │
│  💯 PR review, GitHub Actions, 100% lisible                         │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Control Tower via Terraform

Control Tower controls can be managed via Terraform:

**Reference:** 
- Terraform: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/controltower_control
- AWS Docs: https://docs.aws.amazon.com/controltower/

---

## Implementation Plan

### Phase 1: Control Tower Setup (Console)

> **Voir [BOOTSTRAP-RUNBOOK](../../bootstrap/docs/BOOTSTRAP-RUNBOOK.md) pour les instructions détaillées.**

| Step | Action |
|------|--------|
| 1 | Choose setup preferences (regions, region deny) |
| 2 | Create OUs (Security, Sandbox) |
| 3 | Configure Service integrations — **créer 2 comptes** |
| 4 | Review and enable (~45 min) |

**Comptes créés dans Step 3 :**

| Service | Account | Email |
|---------|---------|-------|
| AWS Config Aggregator | **Audit** | `aws+audit@talq.xyz` |
| CloudTrail Administrator | **Log Archive** | `aws+logs@talq.xyz` |

> ⚠️ Config et CloudTrail exigent des comptes **différents**.

### Phase 2: Terraform Layer (bootstrap/)

| Component | Approach |
|-----------|----------|
| Organizations | CT-managed, read via data sources |
| OUs | CT-managed, custom via Terraform |
| SCPs | CT-managed + custom via `aws_controltower_control` |
| SSO | Terraform (`aws_ssoadmin_*`) |
| Account Factory | AFT via GitHub Actions + Terraform |
| Workload accounts | AFT baseline + Terraform customizations |

### Phase 3: Platform (platform-application-provisioning/)

- VPC, EKS, RDS, Kafka — Pure Terraform
- GitHub Actions CI/CD
- 100% GitOps

---

## What changes in bootstrap/

| Current | New |
|---------|-----|
| `organization/` creates org | CT creates org, we read via data |
| `scps/` creates all SCPs | CT SCPs + custom via `aws_controltower_control` |
| `core-accounts/` creates accounts | CT creates Log/Audit, we create others |
| `account-factory/` custom | AFT module via GitHub Actions |
| `sso/` | Stays Terraform (SSO is independent) |

---

## Decision Matrix by Stage

| Stage | Approach |
|-------|----------|
| 🟢 Early startup (1-5 accounts, no audit < 12 months) | Pure Terraform OK (but CT-compatible design) |
| 🟡 Scaling / Series A / B2B clients | Control Tower OBLIGATOIRE |
| 🔵 Enterprise / regulated | Control Tower non-negotiable |

**LOCAL-PLUS position:** 🟡 → Control Tower recommended

---

## Risks and Mitigations

| Risk | Decision |
|------|----------|
| CT opacity | Resource inventory maintenu dans cet ADR. Data sources Terraform pour lire les ressources CT. |
| AFT interne CodePipeline | AFT est un module Terraform. GitHub Actions exécute Terraform → AFT. Le CodePipeline interne est géré par AWS. |
| CT behavior changes | Provider Terraform pinné. Tests en sandbox avant promotion. |
| Expertise split | CODEOWNERS défini: `@security` pour CT, `@platform` pour Terraform. |

---

## Consequences

### Positive

- Compliance ready — auditors know Control Tower
- Reduced blast radius — AWS manages critical controls
- Future-proof — no migration pain
- Security baseline by default

### Negative

| Limitation | Resolution |
|------------|------------|
| Console setup one-time | **Accepté.** Documenté dans BOOTSTRAP-RUNBOOK. Exécuté 1 seule fois. |
| Ressources CT pas dans Terraform state | Data sources pour lire les IDs. Inventory maintenu dans cet ADR. |
| Équipe doit comprendre CT + Terraform | Formation Platform Team. Ownership clair dans CODEOWNERS. |

---

## References

- [Bootstrap Guide](../bootstrap/BOOTSTRAP-GUIDE.md)
- [Security Architecture](../security/SECURITY-ARCHITECTURE.md)
- AWS Control Tower: https://docs.aws.amazon.com/controltower/
- Terraform aws_controltower_control: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/controltower_control

---

*Document maintenu par : Platform Team*  
*Dernière mise à jour : Janvier 2026*
