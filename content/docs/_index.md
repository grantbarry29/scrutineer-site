---
title: "Overview"
description: "What Scrutineer is, how it works, and where to start."
---

**Scrutineer** is an open-source, Kubernetes-native governance control plane for
autonomous AI agents: per-session policy, scoped human approvals, and runtime evidence
produced **outside the agent's trust domain** — so a prompt-injected or compromised
agent can't bypass enforcement or falsify the record of what it did.

It is **not** an agent framework, workflow engine, or inference layer. You bring your
own agent as a container image; Scrutineer wraps it in governance it cannot reach.

## How it works

Each governed run is an `AgentSession`. The controller provisions the agent workload,
a **per-session Envoy proxy in its own pod**, and a default-deny NetworkPolicy that
makes the proxy the agent's *only* egress path — enforced by the CNI in the kernel,
outside the agent's network namespace.

```text
agent pod (unprivileged, credential-empty)
   │
   ├── governed traffic ──► per-session Envoy pod ──► allowed upstreams
   │                        (own identity/netns)  └──► observed evidence ──► reporter
   │
   └── bypass attempt (raw socket, unset proxy env)
                └──► dropped by default-deny NetworkPolicy (kernel/CNI)
```

Every runtime decision lands in the session's status with a provenance label:
**`observed`** (reported by the out-of-pod proxy's own identity) or
**`self-reported`** (anything from inside the agent's trust domain). Neither pretends
to be the other.

## Quick start

```sh
git clone https://github.com/grantbarry29/scrutineer.git
cd scrutineer
make quickstart   # kind cluster + controller + verified-or-refused gate (~5 min)
make demo         # enforced vs audit-only, live denial + bypass + evidence (~2 min)
```

## Start here

{{< cards >}}
{{< card href="/docs/getting-started/quickstart/" title="Quickstart" >}}
One command to a running, lock-verified Scrutineer on a local kind cluster.
{{< /card >}}
{{< card href="/docs/getting-started/demo/" title="Demo" >}}
Watch a denial, a dead bypass attempt, and evidence the agent couldn't forge.
{{< /card >}}
{{< card href="/docs/getting-started/install/" title="Install on your cluster" >}}
The no-magic path: every command visible, on your existing cluster.
{{< /card >}}
{{< card href="/docs/concepts/core-concepts/" title="Core concepts" >}}
Sessions, policies, profiles, the two locks, and evidence assurance.
{{< /card >}}
{{< /cards >}}

## Learn more

{{< cards >}}
{{< card href="https://github.com/grantbarry29/scrutineer" title="GitHub" >}}
Source, issues, and the full reference documentation.
{{< /card >}}
{{< card href="https://github.com/grantbarry29/scrutineer/tree/main/docs/design" title="Design docs" >}}
The threat model, architecture decisions, and every stated boundary.
{{< /card >}}
{{< card href="/blog/agents-forge-audit-trails/" title="Why this exists" >}}
The essay: your AI agent can edit its own audit log.
{{< /card >}}
{{< card href="/docs/faq/" title="FAQ" >}}
Why not just a firewall, a service mesh, or a CNI? Honest comparisons.
{{< /card >}}
{{< /cards >}}
