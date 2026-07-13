---
title: "Demo: untamperable egress governance"
linkTitle: "Demo"
date: 2026-07-07
weight: 2
summary: "Two sessions, same bring-your-own agent: a live denial, a bypass attempt killed by the routing lock, and evidence the agent couldn't forge."
ShowToc: false
ShowReadingTime: false
hidemeta: true
---

One run, two sessions, same plain-busybox agent: one `enforced`, one `audit-only`.
You'll see a denied request rejected live, a bypass attempt die in the kernel, and all
of it recorded as evidence the agent couldn't forge.

{{< details summary="0 · Prerequisites" >}}
A [quickstart](/docs/getting-started/quickstart/) cluster that ended `VERIFIED`, with
internet egress (the demo fetches `example.com`).
{{< /details >}}

{{< details summary="1 · Run it" >}}
```sh
make demo
```

Applies one hardened `RuntimeProfile`, two `AgentPolicy` objects differing **only in
`mode`**, and two sessions running the same busybox agent — nothing in the image
cooperates with enforcement. Each agent probes three paths and prints what it
experienced (~2 minutes):

| probe | `demo-enforced` | `demo-audit` |
|---|---|---|
| `example.com` via the proxy (allowlisted) | `SUCCEEDED` | `SUCCEEDED` |
| `example.net` via the proxy (not allowlisted) | `BLOCKED` | `SUCCEEDED`, recorded as `dry-run` |
| direct DNS, skipping the proxy (bypass attempt) | `BLOCKED` | `BLOCKED` |

Notice: the proxy env is a convenience, not the control — the default-deny
NetworkPolicy is why the bypass dies. And it dies in **both** modes: `audit-only`
relaxes blocking, never the routing lock, or the observations couldn't be trusted.
{{< /details >}}

{{< details summary="2 · Read the evidence" >}}
`make demo` ends by printing `status.policyDecisions` for both sessions.

- **action** — `deny` (enforced) vs `dry-run` (audit) for `example.net`. Mode changed
  what *happened*, never what was *seen*.
- **assurance** — every entry is `observed`: reported by the proxy pod under its own
  identity. The agent has no path to inject or launder evidence.
  ([Why.](/docs/concepts/core-concepts/#evidence-assurance))
- **what's absent** — the bypass attempt left no entry. The CNI drops it silently;
  recording attempts unforgeably is a tracked roadmap item, stated rather than hidden.

Dig further:

```sh
kubectl get agentsession demo-enforced -o yaml
kubectl get events --field-selector involvedObject.name=demo-enforced
kubectl get pods -l scrutineer.sh/session
```
{{< /details >}}

{{< details summary="3 · Honest boundaries" >}}
- External TLS is tunneled: filtering is by domain, not request bodies.
- Tool and file governance have no enforcement backend yet — removed rather than
  shipped as advisory; they return as out-of-pod chokepoints.
- The guarantee assumes an enforcing CNI (proved by the gate) and an uncompromised
  node — spelled out in the
  [design docs](https://github.com/grantbarry29/scrutineer/tree/main/docs/design).
{{< /details >}}

{{< details summary="4 · Clean up" >}}
```sh
make demo-down        # remove the demo sessions/policies/profile
make quickstart-down  # delete the kind cluster entirely
```
{{< /details >}}

Next: [install on your cluster](/docs/getting-started/install/) — the same story,
every command visible, on your own cluster.
