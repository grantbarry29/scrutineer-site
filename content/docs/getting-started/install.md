---
title: "Install on your cluster, step by step"
linkTitle: "Install on your cluster"
date: 2026-07-12
weight: 3
summary: "Every command visible, on your existing cluster: install, deny one domain, run an agent, read the proof."
ShowToc: false
ShowReadingTime: false
hidemeta: true
---

The [quickstart](/docs/getting-started/quickstart/) is one command that hides the
steps. Here they all are, on your own cluster. You'll install Scrutineer, deny one
domain, run an agent that tries it, and read proof of the block.

{{< details summary="0 · Prerequisites" >}}
- A cluster and `kubectl`.
- A CNI that enforces egress NetworkPolicy (Calico, Cilium, and kind's default all
  do). Unsure? Scrutineer checks for you in step 2.
- Nodes that can pull from `ghcr.io` and Docker Hub.
- Internet egress from pods -- the audit-mode run in step 7 really fetches
  `example.com`.

The install is just CRDs, RBAC, and one controller. No cert-manager or webhooks.
{{< /details >}}

{{< details summary="1 · Install the CRDs and controller" >}}
```sh
git clone --depth 1 --branch v0.2.0 https://github.com/grantbarry29/scrutineer.git
cd scrutineer
kubectl apply -k config/crd
kubectl apply -k config/default
kubectl -n scrutineer-system rollout status deployment/scrutineer-controller-manager
```

Want to inspect before applying? `kubectl kustomize config/default | less`.
{{< /details >}}

{{< details summary="2 · Check the enforcement verdict" >}}
Scrutineer tests your CNI with canary pods instead of trusting it:

```sh
kubectl -n scrutineer-system logs deployment/scrutineer-controller-manager \
  | grep 'lock probe verdict' | tail -1
```

`Verified` -- enforcement is real. `Refused` -- your CNI ignores NetworkPolicy, so
Scrutineer will only run `audit-only` sessions rather than fake enforcement.
{{< /details >}}

{{< details summary="3 · Create a runtime profile" >}}
How governed agents run. The `envoy` entry gives each session its own proxy pod and
locks the agent's network to it:

```yaml
# runtimeprofile.yaml
apiVersion: scrutineer.sh/v1alpha1
kind: RuntimeProfile
metadata:
  name: enforced-egress
  namespace: default
spec:
  container:
    runAsNonRoot: true
    runAsUser: 65532        # busybox runs as root by default; remap it
    runAsGroup: 65532
    readOnlyRootFilesystem: true
    allowPrivilegeEscalation: false
    capabilities:
      drop: ["ALL"]
  pod:
    seccompProfile:
      type: RuntimeDefault
  enforcement:
  - name: envoy
    type: envoy
    enabled: true
```

```sh
kubectl apply -f runtimeprofile.yaml
```
{{< /details >}}

{{< details summary="4 · Write a rule" >}}
Deny one domain, enforced:

```yaml
# policy.yaml
apiVersion: scrutineer.sh/v1alpha1
kind: AgentPolicy
metadata:
  name: deny-example
  namespace: default
spec:
  mode: enforced
  deniedDomains:
  - example.com
```

```sh
kubectl apply -f policy.yaml
```
{{< /details >}}

{{< details summary="5 · Run a governed session" >}}
The agent is plain busybox. It doesn't know Scrutineer exists. It tries the denied
domain:

```yaml
# session.yaml
apiVersion: scrutineer.sh/v1alpha1
kind: AgentSession
metadata:
  name: first-session
  namespace: default
spec:
  task:
    description: Try one denied egress and exit
    prompt: noop
  model:
    provider: openai
    name: gpt-4.1
  runtime:
    orchestrator: kubernetes-job
    image: busybox:latest
    command:
    - sh
    - -c
    - |
      wget -q -O /dev/null http://example.com/ \
        && echo "egress succeeded" \
        || echo "egress blocked"
      exit 0
  runtimeProfileRef:
    name: enforced-egress
  policyRefs:
  - kind: AgentPolicy
    name: deny-example
```

```sh
kubectl apply -f session.yaml
kubectl get agentsessions -w        # Pending → Running → Succeeded
```

While it runs, `kubectl get pods,networkpolicies` shows the agent pod, the
`first-session-egress` proxy pod, and two NetworkPolicies: the routing lock on the
agent, a backstop on the proxy.
{{< /details >}}

{{< details summary="6 · Read the evidence" >}}
```sh
kubectl get agentsession first-session -o jsonpath='{.status.policyDecisions}' \
  | jq '.[] | select(.phase=="runtime")'
```

```json
{
  "type": "network",
  "target": "example.com",
  "action": "deny",
  "assuranceLevel": "observed",
  ...
}
```

(The reporter batches; if the output is empty, wait a moment and re-run. The `select`
skips `merge`-phase entries -- control-plane records of how your policies were
resolved -- and keeps what the proxy *observed*.)

`observed` = reported by the proxy under its own identity. The agent can't forge or
suppress it; anything the agent submits is stamped `self-reported` instead.
([Why.](/docs/concepts/core-concepts/#evidence-assurance))

The block itself, at the proxy:

```sh
kubectl logs first-session-egress -c envoy | grep example.com   # → 403
```
{{< /details >}}

{{< details summary="7 · Flip the rule to audit-only" >}}
One field turns enforcement into rehearsal -- traffic flows, the denial is recorded as
`dry-run`. Flip the policy, then run a **second** session under it:

```sh
kubectl patch agentpolicy deny-example --type merge -p '{"spec":{"mode":"audit-only"}}'
sed 's/name: first-session/name: first-session-audit/' session.yaml > session-audit.yaml
kubectl apply -f session-audit.yaml
kubectl get agentsessions -w
```

Same agent, same rule: this time the wget goes through. Read the new record:

```sh
kubectl get agentsession first-session-audit -o jsonpath='{.status.policyDecisions}' \
  | jq '.[] | select(.phase=="runtime")'
```

`"action": "dry-run"` -- still `observed`. And `first-session` still holds its `deny`:
re-run step 6 and compare them side by side. Mode changed what *happened*, never what
was *seen*.

A second session (not a delete-and-rerun) on purpose: the session object **is** the
audit record, and today the evidence lives only in its `status`. Delete the session
and the record is gone. Rerun by creating a new session; keep the old one.
{{< /details >}}

{{< details summary="8 · Clean up" >}}
```sh
kubectl delete -f session.yaml -f session-audit.yaml -f policy.yaml -f runtimeprofile.yaml
kubectl delete -k config/default
```

(`config/default` includes the CRDs, so the second command removes everything,
including any sessions still on the cluster, and their evidence with them.)
{{< /details >}}

**What you just saw:** governance from outside the agent's trust domain (stock
busybox, zero cooperation) · enforcement verified against your CNI, not assumed ·
a block that really happened, with a record the agent couldn't forge.

Next: the [demo](/docs/getting-started/demo/) adds a live bypass attempt;
[core concepts](/docs/concepts/core-concepts/) explains the objects you just used.
