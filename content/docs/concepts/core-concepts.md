---
title: "Core concepts"
date: 2026-07-07
weight: 1
summary: "Sessions, policies, profiles, the two locks, and evidence assurance."
ShowToc: false
ShowReadingTime: false
hidemeta: true
---

Five objects, two locks, one rule about evidence. This is the mental model. The
[design docs](https://github.com/grantbarry29/scrutineer/tree/main/docs/design) carry
the detail.

{{< details summary="The object model" id="the-object-model" >}}
Kubernetes CRDs, API group `scrutineer.sh/v1alpha1`:

- **`AgentSession`** -- one governed run: the agent image, the task, references to
  policy and profile. All runtime evidence lands in its `status`.
- **`AgentPolicy`** -- the rules, plus the **mode**: `enforced` (violations blocked) or
  `audit-only` (recorded as `dry-run`, nothing blocked).
- **`RuntimeProfile`** -- how the workload runs: container hardening and which
  enforcement backends are on.
- **`ApprovalPolicy` / `ApprovalRequest`** -- scoped human approvals for actions that
  need a person in the loop.
{{< /details >}}

{{< details summary="Bring your own agent" id="bring-your-own-agent" >}}
The container image *is* the agent -- reasoning loop, model calls, tools. Scrutineer
schedules it and governs what it can reach; nothing in the image needs to cooperate.
The demo agent is plain busybox.
{{< /details >}}

{{< details summary="The two locks" id="the-two-locks" >}}
Enforcement and credentials both live outside the agent's trust domain:

**Routing lock.** A default-deny egress NetworkPolicy makes the per-session Envoy
proxy the agent's *only* network path. The `HTTP_PROXY` env is a convenience; the
kernel-level deny is the control. Raw sockets and direct DNS die at the CNI.

**Capability lock.** The agent pod is credential-empty. Secrets that authorize
governed actions live outside its reach. A compromised agent has nothing to
exfiltrate that would let it act ungoverned.
{{< /details >}}

{{< details summary="Verified or refused" id="verified-or-refused" >}}
The routing lock is only real if the CNI enforces NetworkPolicy. Some don't.
Scrutineer proves it with a canary probe instead of assuming it. Fail the probe and
enforced sessions are **refused** with a loud `EgressLockVerified=False` condition.
Nothing silently degrades to advisory.
{{< /details >}}

{{< details summary="Evidence assurance" id="evidence-assurance" >}}
Every decision in `status.policyDecisions` carries a provenance label:

- **`observed`** -- reported by the egress-reporter in the proxy pod, authenticated as
  *that pod's own identity* (token review + ownership checks).
- **`self-reported`** -- anything from inside the agent's trust domain.

The label is derived from the caller's authenticated identity, never from the
payload. A report can *claim* `observed`, and the server overwrites the claim with
what the caller's token proves. The doctrine throughout: a control is either
untamperable, or it is labeled for what it is.
{{< /details >}}
