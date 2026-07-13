---
title: "FAQ"
date: 2026-07-12
weight: 99
summary: "Why not just a firewall, a service mesh, or a CNI? Honest comparisons and the reasoning behind the enforcement model."
ShowToc: false
ShowReadingTime: false
hidemeta: true
---

The questions a technical evaluator asks first — answered with the boundaries stated,
not hidden.

{{< details summary="Couldn't I build this myself with an egress proxy and a NetworkPolicy?" id="diy" >}}
The *mechanism* — yes, deliberately so. Scrutineer's data plane is commodity on
purpose: a per-session Envoy and a default-deny NetworkPolicy, portable to any
conformant cluster. A competent team can stand that up in an afternoon.

The product is what makes the mechanism a *guarantee*:

| A proxy pod + NetworkPolicy gives you | What the DIY version is missing |
|---|---|
| A policy object that *should* deny traffic | Proof that it does. On a CNI that ignores NetworkPolicy the lock is a **silent no-op**. Scrutineer probes the CNI and **refuses** enforced sessions rather than degrade. |
| One shared, long-lived proxy | Per-session isolation: a dedicated proxy per session, own identity, torn down with it. Attribution is structural; blast radius is one session. |
| Firewall *logs* | *Evidence*: records authenticated as the proxy pod's own identity, stamped `observed` server-side, landing in the session's `status` as an API object. |
| DNS egress you probably left open | No direct DNS at all — the proxy resolves. An open DNS allowance is a ready-made exfil tunnel. |
| An allow-list you edit by hand | Policy as an API: `enforced` vs `audit-only` dry-runs, fail-closed live updates, injection-safe validation. |
| Blocking | A session lifecycle: verified-or-refused admission, mid-run approvals, cancellation, credential-empty agent pods. |

Build all of that and you haven't disproven the product — you've rebuilt it.
{{< /details >}}

{{< details summary="Why not Cilium with toFQDN policies and Hubble?" id="cilium" >}}
The strongest existing alternative — and a fine CNI *under* Scrutineer; the probe will
happily verify it. The differences are the layer above:

- **Portability** — Scrutineer's guarantee is CNI-agnostic; toFQDN ties policy to one CNI.
- **Name semantics** — toFQDN maps DNS answers to IPs, with the races that implies.
  Scrutineer's proxy is handed the *name* and resolves it itself.
- **Flows vs. evidence** — Hubble exports flows. Scrutineer records
  identity-authenticated decisions bound to a session object, with assurance labels.
- **No session model** — admission gating, approvals, audit-vs-enforce, one API object
  per run.
{{< /details >}}

{{< details summary="Does the agent have to cooperate with the proxy?" id="cooperation" >}}
No — the design assumes it won't. Proxy env is injected so well-behaved HTTP stacks
route automatically; everything else gets nothing. Direct connections die at the CNI,
DNS doesn't resolve. **Through the proxy, or nowhere** — security holds at zero
cooperation; cooperation only buys the agent functionality.

One pain deliberately avoided: Envoy tunnels TLS rather than intercepting it — no MITM
CA to distribute, certificate pinning keeps working.
{{< /details >}}

{{< details summary="What about non-HTTP protocols — databases, SSH, raw TCP?" id="non-http" >}}
Covered when the client can speak an HTTP `CONNECT` tunnel (most can, or can be
wrapped): same proxy, same policy, same lock, same `observed` evidence. Recipes:
[non-HTTP egress guide](https://github.com/grantbarry29/scrutineer/blob/main/docs/egress-non-http.md).

Tools that honor no proxy config at all **fail closed** — no egress rather than
ungoverned egress. Transparent interception for those is
[future work](https://github.com/grantbarry29/scrutineer/issues/64).
{{< /details >}}

{{< details summary="Why an explicit proxy instead of transparent interception?" id="transparent" >}}
Every transparent option costs privilege somewhere sensitive: iptables in the agent
pod needs `NET_ADMIN` in the workload being governed; node-level interception needs a
privileged data plane on every node; CNI-native redirect couples the product to one
CNI. The explicit proxy adds **nothing privileged anywhere**.

It also sees destinations as clean *names* — the policy vocabulary — where transparent
interception sees addresses and reconstructs names from SNI or DNS correlation. The
trade: proxy-oblivious tools fail closed until the
[transparent backend](https://github.com/grantbarry29/scrutineer/issues/64) exists.
But the steering mechanism was never the security boundary — the lock is.
{{< /details >}}
