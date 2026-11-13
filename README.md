# sharkops-netpol

A Helm chart that renders a default-deny ingress `NetworkPolicy` and, where
needed, a Cilium ingress `CiliumNetworkPolicy` for a Kubernetes namespace.

One release of this chart per namespace. `values.yaml` takes a `policies`
list so a single release can cover every workload in that namespace (for
example, a secrets-store namespace may need the server and its snapshot
`CronJob`; `monitoring` needs several).

## Scope

Ingress by default. Egress is opt-in per `policies` entry through its `egress`
key, and renders only into the `CiliumNetworkPolicy`: `egress.enabled: true`
makes the selected pods default-deny for egress, so DNS to kube-dns is always emitted
with it. See the commented example in `values.yaml`.

## Why a `CiliumNetworkPolicy` in addition to `NetworkPolicy`

Cilium ingress traffic arrives with the reserved `ingress` identity, which a
vanilla `NetworkPolicy` cannot name. Anything reachable on a public hostname
needs `allowCiliumIngress: true` on its policy entry, which renders a
`CiliumNetworkPolicy` instead of (or alongside) a `NetworkPolicy` ingress
rule. Without it, a public app returns HTTP 000 and times out.

`helm template` with the default values produces empty output, because
`policies` defaults to `[]`.

## Usage

```yaml
policies:
  - name: wikijs
    # Standard Kubernetes label selector. `{}` selects EVERY pod in the
    # namespace.
    podSelector:
      matchLabels:
        app.kubernetes.io/name: wiki

    # Without this the namespace is simply never scraped by Prometheus.
    allowPrometheus: true

    # Required for anything reachable on a public hostname.
    allowCiliumIngress: true

    # Optional. Omit to allow the ingress proxy on every port.
    ciliumIngressPorts:
      - port: "3000"
        protocol: TCP

    # Raw NetworkPolicy ingress rules, appended verbatim. Renders into the
    # VANILLA NetworkPolicy, which has no concept of ICMP.
    extraIngress: []

    # Raw CILIUM ingress rules, appended verbatim to the CiliumNetworkPolicy.
    # The mirror of egress.extraEgress, and the only way to allow inbound ICMP.
    extraCiliumIngress:
      - fromEntities:
          - cluster
        icmps:
          - fields:
              - type: 3
                family: IPv4
              - type: 1
                family: IPv6
```

### `extraCiliumIngress` and inbound ICMP

`extraIngress` renders into the vanilla `NetworkPolicy`, which cannot express
ICMP, and every `icmps:` hook in this chart is on the **egress** side. So until
0.12.0 there was no way to allow an inbound ICMP type — while the automatic
egress rule allowed five of them. That asymmetry was the bug.

For example, making CoreDNS default-deny on ingress also denies inbound
**Destination Unreachable** (IPv4 type 3, IPv6 type 1) from clients answering a
*late* DNS reply — they already had their answer from the other replica, so the
response hit a closed socket. Harmless, ignored by CoreDNS, and enough to keep a
policy-drop alert firing continuously. DNS never breaks and no pod restarts: the
policy is correct and still has to come out.

**Setting this alone makes the endpoint default-deny for ingress**, exactly as
any egress content makes it default-deny for egress. Adding an ICMP allowance to
a workload that has no ingress policy does not "also allow ICMP" — it denies
everything else. Pair it with the real ingress rules.

## Installation

```bash
helm install <release-name> oci://ghcr.io/sharkops-io/charts/sharkops-netpol \
  --namespace <namespace> \
  -f values.yaml
```

## License


Apache License 2.0. See [LICENSE](LICENSE).
