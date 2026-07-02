---
title: "Your AI agent can edit its own audit log"
date: 2026-07-02
draft: true
tags: ["ai-agents", "kubernetes", "security", "audit"]
summary: "Most agent guardrails are an honor system: enforcement and evidence live inside the agent's own trust domain. Here's what it takes to produce an audit trail the agent can't bypass, tamper with, or forge — and what I built to get there."
---

Here's an uncomfortable question for anyone running autonomous agents: if a prompt-injected agent exfiltrated a credential from your environment last Tuesday, would your logs show it?

If your logs are produced by the agent's own process, or by a sidecar sharing its pod, the honest answer is: *you don't know*. The record of what the agent did was written from inside the agent's own trust domain. An agent that has been steered off-script — by prompt injection, a malicious tool description, or a poisoned dependency — is in a position to bypass the controls, starve the reporter, or quietly write a cleaner version of history.

That's not an audit trail. It's an honor system with extra YAML.

## The cooperative-security trap

Almost every agent guardrail I've seen falls into one of three patterns:

1. **In-process controls.** Framework callbacks, permission wrappers, "safety" middleware inside the agent runtime. The agent's code enforces limits on the agent's code. A compromised agent *is* the enforcement point.
2. **In-pod controls.** A proxy or policy sidecar in the same pod. Better — it's a separate process — but it shares the agent's network namespace and often its ServiceAccount. `HTTP_PROXY` is a *suggestion*; a raw socket ignores it. An adjacent process can be bypassed, tampered with, or starved.
3. **Provider-side controls.** The model API applies filters. Useful, but blind to everything that matters at runtime: which host actually received the connection, which file was written, which tool was invoked with which arguments.

All three share a fatal property: **the evidence and the enforcement live somewhere the agent can reach.** For a demo, fine. For the thing your security team signs off on, not fine.

I've spent my career in network dataplanes, where we internalized this a long time ago: you don't ask the workload to firewall itself.

## The principle

> Integrity never comes from *which tool* you use. It comes from the enforcement point living in a **trust domain the agent has no privilege to alter**, and from making that point the **only path** for the governed traffic.

Two properties, both required:

- **Out of the agent's control** — separate pod, separate identity, separate network namespace.
- **Mandatory** — the agent's *only* working egress path, enforced by a layer the agent cannot modify.

Miss either one and you're back to the honor system. A perfect proxy the agent can route around is decoration. An unavoidable proxy the agent can reconfigure is worse.

## What this looks like in practice

[Scrutineer](https://github.com/grantbarry29/scrutineer) is an open-source Kubernetes operator that governs agent runs as `AgentSession` resources: policy resolution, runtime provisioning, approvals, and evidence collection. The part I want to walk through is the egress path, because it's where the trust-domain principle becomes concrete.

For each session, the controller provisions a **per-session Envoy proxy in its own pod** — its own ServiceAccount, its own network namespace, owner-referenced to the session and torn down with it. The agent pod gets standard `HTTP_PROXY` / `HTTPS_PROXY` / `ALL_PROXY` environment pointing at that proxy. Envoy terminates HTTP and tunnels HTTPS via `CONNECT`.

On its own, that's still a suggestion. What makes it mandatory is a **default-deny egress NetworkPolicy** on the agent pod: the only reachable destinations are the session's proxy and the evidence endpoint. It's enforced by the CNI at the pod boundary — in the kernel, outside the agent's network namespace, described by a Kubernetes object the agent has no RBAC to touch.

```
agent pod (unprivileged, nothing added but env)
   │
   ├── well-behaved traffic ──► per-session Envoy pod ──► allowed upstreams
   │                            (own identity/netns)  └──► observed evidence ──► reporter
   │
   └── bypass attempt (raw socket, unset proxy env)
                └──► dropped by default-deny NetworkPolicy (kernel/CNI)
```

Some consequences of this shape that turned out to matter:

**DNS moves out of the agent.** With `CONNECT`-style proxying, the agent hands the proxy a *hostname* and the proxy resolves it. The agent needs no direct DNS at all, so the NetworkPolicy denies DNS egress entirely — which closes DNS tunneling as an exfiltration channel and gives the policy layer clean hostnames instead of guessing from IPs.

**The agent pod stays unprivileged.** No privileged init container, no `NET_ADMIN`, no transparent-redirect tricks inside the pod. The only moving parts are injected environment variables, a NetworkPolicy, and a separate proxy pod. (Transparent interception is a planned alternative backend, but it costs a privileged init container — it's genuinely a weaker security posture than the boring explicit proxy.)

**Backstops live in the kernel, not the proxy.** Cloud metadata endpoints, cluster-internal ranges, the Kubernetes API — those denies hold *even if the proxy itself is compromised*, because they're in the CNI layer, not in Envoy config.

**Failure fails closed.** If the proxy isn't ready, the agent has no egress. An outage denies; it never bypasses.

## Evidence with a provenance label

Enforcement is half the story. The other half: when the proxy reports what happened, how do you know *that* report is trustworthy?

The proxy authenticates to the evidence endpoint with its own projected ServiceAccount token. The receiving side verifies, via TokenReview, that the caller is exactly the session's proxy — the right pod name, controller-owned by the right `AgentSession`, running as the dedicated per-session ServiceAccount. All three must hold; a lookalike pod missing any one is rejected, and an agent-adjacent process can't impersonate the proxy's identity.

Evidence that passes that check is stamped **`observed`**. Evidence from anything inside the agent's trust domain — cooperative sidecars, the agent itself — is stamped **`self-reported`**. Both are recorded; neither pretends to be the other.

This distinction sounds small. It's the whole product. When a security review asks "what did this agent do," every line in the answer carries its assurance level. You know which parts of the record could survive a hostile agent and which parts assume a cooperative one.

## What this does *not* guarantee

Security writing that omits its assumptions is marketing. The guarantee above is "adversarial-grade *for governed egress*," under stated boundaries:

- **The CNI must actually enforce NetworkPolicy.** Calico and Cilium do. Some default CNIs silently don't — on those clusters the routing lock doesn't exist, and the system should say so rather than pretend.
- **The agent pod is unprivileged** — capabilities dropped, no `NET_ADMIN`. The guarantee is "the agent can't tamper *without escaping to the node*." A node or CNI compromise is a strictly higher threat tier and out of scope.
- **Coverage is HTTP/S plus `CONNECT`-tunneled TCP** — which is most agent egress (model APIs, web, git/pip/npm, cloud SDKs). Raw non-HTTP L4 is currently denied, not observed: fail-closed until transparent interception lands.
- **Proxy-unaware tools fail closed**, not open. That's an availability cost, deliberately paid. Kernel-level syscall/file observation is future work, and file/tool governance today is cooperative — labeled `self-reported`, exactly as the model demands.

I'd rather ship a narrow guarantee that's true than a broad one that's true until someone tries.

## Why I think this matters now

Agents are crossing from "chatbot with tools" to "unattended process with credentials, a git remote, and a shell." Every serious platform team I can find is asking some version of the same question: *how do we let these things act without trusting them?* The answer isn't a smarter agent or a better system prompt. It's the same answer infrastructure has always given for untrusted workloads: put the controls where the workload can't reach them, and make the record of what happened independent of the thing being recorded.

Scrutineer is my attempt at that layer — [open source, Apache 2.0](https://github.com/grantbarry29/scrutineer), built as a Kubernetes operator with the honest parts (`observed` vs `self-reported`, stated threat-model boundaries) in the API, not the README. If you're running agents on Kubernetes and this problem is on your desk, I'd genuinely like to hear what your security review would ask that this doesn't answer: issues and mail are open.
