---
title: "Overview"
description: "What Scrutineer is, how it works, and where to start."
---

**Scrutineer** governs autonomous AI agents on Kubernetes: per-session policy, human
approvals, and runtime evidence produced **outside the agent's trust domain**. A
compromised agent can't bypass the rules or edit the record.

Not an agent framework. You bring your own agent image. Scrutineer wraps it in
governance it can't reach.

## How it works

Each run is an `AgentSession`. The controller starts the agent pod, a **per-session
Envoy proxy**, and a default-deny NetworkPolicy that makes the proxy the agent's
**only** way out is enforced in the kernel, outside the agent's reach.

```mermaid
flowchart LR
  agent["agent pod<br/>unprivileged · credential-empty"]
  proxy["per-session Envoy proxy<br/>own pod · own identity"]
  up["allowed upstreams"]
  rep["evidence reporter"]
  drop["dropped in the kernel<br/>default-deny NetworkPolicy"]

  agent -->|governed traffic| proxy
  proxy --> up
  proxy -->|observed evidence| rep
  agent -.->|bypass attempt| drop

  classDef dead stroke:#e5534b,stroke-dasharray:4 3
  class drop dead
```

Every decision lands in the session's status, labeled by source: **`observed`**
(the proxy) or **`self-reported`** (the agent). Neither pretends to be the other.

## Quick start

```sh
git clone https://github.com/grantbarry29/scrutineer.git
cd scrutineer
make quickstart   # kind cluster + verified enforcement (~5 min)
make demo         # a denial, a dead bypass, the evidence (~2 min)
```

## Start here

{{< cards >}}
{{< card href="/docs/getting-started/quickstart/" title="Quickstart" >}}
One command to a verified install on kind.
{{< /card >}}
{{< card href="/docs/getting-started/demo/" title="Demo" >}}
A live denial, a dead bypass, and the evidence.
{{< /card >}}
{{< card href="/docs/getting-started/install/" title="Install on your cluster" >}}
Every command visible, on your own cluster.
{{< /card >}}
{{< card href="/docs/concepts/core-concepts/" title="Core concepts" >}}
Sessions, policies, the two locks, evidence.
{{< /card >}}
{{< /cards >}}

## Learn more

{{< cards >}}
{{< card href="https://github.com/grantbarry29/scrutineer" title="GitHub" >}}
Source, issues, and reference docs.
{{< /card >}}
{{< card href="https://github.com/grantbarry29/scrutineer/tree/main/docs/design" title="Design docs" >}}
Threat model and every stated boundary.
{{< /card >}}
{{< card href="/blog/agents-forge-audit-trails/" title="Why this exists" >}}
Your agent can edit its own audit log.
{{< /card >}}
{{< card href="/docs/faq/" title="FAQ" >}}
Why not just a firewall or a CNI?
{{< /card >}}
{{< /cards >}}
