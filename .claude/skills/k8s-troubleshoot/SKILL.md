---
name: k8s-troubleshoot
description: Guides the resolution of a problem reported in the homelab (k3s cluster) and, once it is resolved, writes a detailed incident record in ~/devops/troubleshooting. Use whenever the user reports a problem or error in the infrastructure, for example "I have a problem", "it's throwing an error", "it doesn't work", "the pod won't start", "the service is gone", "help me fix this" (or the same in Portuguese: "tenho um problema", "está dando erro", "não funciona", "o pod não sobe", "o serviço sumiu", "me ajude a resolver"), or asks to record/document a problem that was already solved.
---

# Troubleshooting with an incident record

Goal: solve the problem the user reported and, at the end, leave a markdown record in `~/devops/troubleshooting/`. That folder is versioned on GitHub and is the history of the problems faced on the way to a 100% working infrastructure.

The user often writes in Portuguese. Understand reports in either language, and reply in the language the user writes in. Files written by this skill are always in English.

## 1. Ask for the namespace and a description

Always start by asking two things, before running any command:

1. **Which namespace** the problem is in.
2. **A short description** of the problem: what happens and what should happen.

Use the `AskUserQuestion` tool for the namespace. List the project namespaces that exist as options (check with `kubectl get ns`); the user can type another one under "Other". Ask for the description in the same question or as free text.

If the user's report already includes the namespace or the description, do not ignore it: state what you understood and ask the user to confirm or correct it (for example, "I understood the problem is X in namespace Y. Is that right?").

If the user does not know the namespace, or the problem affects the whole cluster, record it as "cluster" and move on.

## 2. Investigate

Use the `k8s-diagnose` skill to investigate, focusing on the namespace given. It has the commands and the table of symptoms and causes. Throughout the investigation, keep what will go into the record:

- the commands that were decisive for the diagnosis;
- the exact error messages and events (copy the text, do not paraphrase);
- the hypotheses ruled out, and why.

## 3. Fix

- The investigation is read-only. **Before any change** to the cluster or the manifests (`apply`, `patch`, `delete`, `edit`, `scale`, `rollout restart`, `helm`), show the user exactly what will change and wait for their approval.
- If the user wants to follow along in real time, give them the `watch`/`-w` commands to run in another terminal before applying.
- Prefer fixing the manifests in `~/devops/manifests/` over fixing only the cluster, so the fix is not lost. Resources managed by ArgoCD are reverted if changed only in the cluster.
- After applying, check that the original symptom is gone.

## 4. Confirm the resolution

The problem is only resolved when the user confirms it. After the verification, ask whether it can be considered resolved.

- If the user confirms, go on to the record.
- If the user gives up or postpones, ask whether to record it anyway, with status "open" and what has been found so far.

## 5. Write the incident record

Create the file in `~/devops/troubleshooting/` named `YYYY-MM-DD-<short-summary>.md`, using today's date and a kebab-case summary (for example, `2026-11-07-argocd-resource-quota.md`). Before creating it, list the folder: if a record of the same problem already exists, update it instead of creating another.

Write the record in English, even when the conversation was in Portuguese. Write it in detail, so the user understands the problem when rereading it months later. Explain the Kubernetes concepts involved as if to someone who is learning, instead of only naming them. Use the template below and drop sections that do not apply. For the expected level of detail, see `2026-11-07-argocd-resource-quota.md` in the same folder.

```markdown
# <Title that describes the problem>

- **Date:** YYYY-MM-DD
- **Namespace:** `<namespace>`
- **Component:** <affected app or resource, with version if known>
- **Status:** resolved | open

## Symptom

What the user saw, with the commands and outputs that showed the problem.

## Diagnosis

The commands that led to the cause, in order, and the exact error messages and events.
Include the hypotheses ruled out, and why.

## Cause

The root cause, in one or two sentences.

## Understanding the problem

A detailed explanation of the concepts involved (for example: what a ResourceQuota is,
how requests and limits work) and of why they caused the symptom. Use tables
and examples with the real numbers from the case.

## Solution

What was changed (files, values before → after, commands applied) and why.
How it was verified that the fix worked.

## Lessons

What to check or do differently to avoid the same problem.
```

### Sensitive data

The file goes to GitHub. Never put passwords, tokens, keys, Secret values, `kubeconfig` contents or database credentials in it. If a command output contains such data, replace the value with `<redacted>`. Local network IPs, resource names and Kubernetes error messages are fine.

When done, give the user the full path of the file and summarize in a few lines what was recorded.
