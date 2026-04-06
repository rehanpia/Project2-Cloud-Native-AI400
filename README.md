# Kubernetes Deployment Planning — Panaversity Cloud Native AI Training

> **Course:** Cloud Native AI | **Project:** 2 — Kubernetes Deployment Planning  
> **Student:** Panaversity Student  
> **Version:** 3.0 | **Date:** 2026-04-04

---

## Project Overview

This repository contains production-grade Kubernetes deployment plans for two real-world AI application scenarios, plus a reusable K8 Planning Skill that can generate such plans for any future project.

---

## Repository Structure

```
k8s-deployment-plans/
├── README.md                          ← This file
├── plan1-ai-native-task-manager.md    ← Scenario 1: AI Native Task Manager
├── plan2-ai-employee-openclaw.md      ← Scenario 2: AI Employee (OpenClaw)
└── k8-planning-skill.md               ← Reusable Agent Skill for K8 Planning
```

---

## Scenario 1 — AI Native Task Manager

An AI-powered task management application with four services communicating according to the following rules (from the original project diagram):

| Rule | Description |
|---|---|
| 1 | UI Interface connects with Backend APIs to manage tasks |
| 2 | UI Interface connects **directly** with Todo Agent to manage tasks |
| 3 | Todo Agent connects with Backend API to manage tasks |
| 4 | Notification Service connects with both UI Interface and Backend API |

**Components:** UI Interface · Backend APIs · Todo Agent · Notification Service · PostgreSQL

**Plan covers:** Deployments, StatefulSet, Services (LoadBalancer + ClusterIP + Headless), ConfigMaps, Secrets, RBAC, NetworkPolicies matching all 4 communication rules, HPAs, PodDisruptionBudgets.

---

## Scenario 2 — AI Employee (OpenClaw)

A Personal AI Employee with strong security considerations, answering specifically:
- How many ConfigMaps and Secrets are needed?
- How is RBAC structured?
- **What happens when a secret expires, is compromised, or the agent attempts unauthorized access?**

**Components:** Auth Gateway · Orchestrator · Tool Executor (sandboxed) · Memory Service · Credential Vault Proxy · Audit Logger · User Dashboard

**Plan covers:** 4-namespace security isolation, mTLS via Istio, Vault-based secret management, full Secret Failure Scenario playbook (expiry / compromise / agent unauthorized access), PodDisruptionBudgets.

---

## K8 Planning Skill

A reusable AI agent skill with:
- Input schema for any application
- 5-phase reasoning framework
- 11 validated YAML snippet templates
- 4 decision reference cards (including the critical cross-namespace NetworkPolicy AND/OR rule)
- Pre-submission self-check (18 items)
- 2 worked example invocations

---

## Key Design Decisions

| Decision | Rationale |
|---|---|
| One `ServiceAccount` per workload | No shared identities — limits blast radius of a compromised pod |
| `resourceNames` in all Roles | Pods can only access the exact secrets they need |
| Deny-all NetworkPolicy first | Explicit allow > implicit allow; reduces attack surface |
| AND logic in cross-namespace NetworkPolicies | Separate `from` list items = OR (insecure); same item = AND (correct) |
| `startupProbe` on all services | Prevents premature liveness failures during slow startup |
| Images pinned to exact versions | `:latest` makes rollbacks unpredictable |
| Vault sidecar for critical secrets | Secrets as in-memory files, never environment variables |
| `fsGroup` on all pods | Required for correct file ownership on mounted volumes |

---

## Submission Checklist

- [x] Plan 1 — AI Native Task Manager (Markdown)
- [x] Plan 2 — AI Employee OpenClaw (Markdown)
- [x] K8 Planning Skill (Markdown)
- [x] All 4 communication rules from original diagram addressed
- [x] Secret failure scenarios documented (expiry / compromise / agent access)
- [x] GitHub-ready repo structure

---

*Panaversity Cloud Native AI Training Program*
