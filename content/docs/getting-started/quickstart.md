---
title: "Quickstart"
date: 2026-07-07
weight: 1
summary: "One command to a running, lock-verified Scrutineer on a local kind cluster — about five minutes."
ShowToc: false
ShowReadingTime: false
hidemeta: true
---

One command, a local [kind](https://kind.sigs.k8s.io/) cluster, about five minutes —
and the enforcement guarantee is *proved*, not assumed, before it reports success.
(Prefer to see every step on your own cluster?
[Install on your cluster](/docs/getting-started/install/).)

{{< details summary="0 · Prerequisites" >}}
- **Docker**, **kind**, and **kubectl** on your PATH (tested with kind v0.31.0).
- Internet egress from the cluster — the sample agents fetch `example.com`.
- ~5 minutes on a first run — the images are always built from your checkout, never
  pulled, so what runs is exactly what you cloned. Repeats are faster.
{{< /details >}}

{{< details summary="1 · Run it" >}}
```sh
git clone https://github.com/grantbarry29/scrutineer.git
cd scrutineer
make quickstart
```

Creates a `scrutineer-quickstart` kind cluster, installs Scrutineer, then runs a
canary probe to prove the CNI actually enforces NetworkPolicy. It ends with the
verdict:

```text
>> routing-lock enforcement VERIFIED on this cluster (enforced sessions will run).
```
{{< /details >}}

{{< details summary="2 · If it says REFUSED" >}}
Your CNI doesn't enforce NetworkPolicy, so Scrutineer refuses to claim enforcement
rather than pretend. Retry on Calico:

```sh
make quickstart-down
make quickstart QUICKSTART_CNI=calico
```
{{< /details >}}

{{< details summary="3 · Try a sample session" >}}
```sh
kubectl apply -f config/samples/scrutineer_v1alpha1_agentsession.yaml
kubectl get agentsessions -w
kubectl get agentsession github-readme-update -o yaml   # the run's record, evidence included
```
{{< /details >}}

{{< details summary="4 · Tear down" >}}
```sh
make quickstart-down
```
{{< /details >}}

Next: the [demo](/docs/getting-started/demo/) — a live denial, a dead bypass attempt,
and evidence the agent couldn't forge, in one run.
