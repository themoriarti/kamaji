# Network Policies for Tenant Control Planes

Tenant Control Plane pods run in the management cluster, next to the Kamaji controller, the DataStore and any other workload you host there. Without network policies, every pod in the management cluster can open connections to every Tenant Control Plane and to the DataStore, and a Tenant Control Plane pod can open connections anywhere.

This guide lists the traffic a Tenant Control Plane needs and gives a starting set of `NetworkPolicy` objects that allow only that traffic. Adapt them to your exposure method and CNI before applying them in production.

The examples assume:

- Kamaji and the `kamaji-etcd` DataStore are installed in `kamaji-system`;
- the Tenant Control Plane `tenant-00` lives in the namespace `tenant-a`, with the API server on port `6443` and Konnectivity on port `8132`;
- the management cluster uses the pod CIDR `10.244.0.0/16`.

## Traffic of a Tenant Control Plane

Kamaji labels every Tenant Control Plane pod with `kamaji.clastix.io/component: deployment` and `kamaji.clastix.io/name: <tenant control plane name>`. The policies below select pods by these labels.

| Direction | Peer | Port | Why |
|-----------|------|------|-----|
| Ingress | Worker nodes and other clients of the tenant cluster | API server port (`6443`) | kubelet, kube-proxy, the CNI and users reach the tenant API through the Service, Ingress or Gateway |
| Ingress | Konnectivity agents on the worker nodes | Konnectivity port (`8132`) | agents open the tunnel the API server uses to reach kubelets, webhooks and aggregated APIs |
| Ingress | Kamaji controller | API server port | the controller runs the kubeadm phases and the addons against the tenant API |
| Ingress | Cluster API controllers | API server port | when you use the Cluster API Kamaji provider, Cluster API reads Nodes from the tenant cluster |
| Ingress | Ingress controller or Gateway | API server and Konnectivity ports | only when the Tenant Control Plane is exposed through an Ingress or a Gateway |
| Ingress | Any client of a port in `spec.controlPlane.service.additionalPorts` | that port | for example a sidecar added through `additionalContainers` |
| Egress | DataStore | `2379` for etcd, `3306` MySQL, `5432` PostgreSQL, `4222` NATS | the API server (or the kine sidecar) stores the tenant state |
| Egress | Cluster DNS | `53` UDP and TCP | the API server resolves the DataStore endpoints |
| Egress | Worker nodes and tenant webhooks | `10250` and the webhook ports | **only without Konnectivity**: the API server then dials kubelets, webhooks and aggregated APIs directly |
| Egress | OIDC issuer, KMS, audit webhook | as configured | only when your API server configuration uses them |

## Ingress to the Tenant Control Plane pods

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: tenant-control-plane-ingress
  namespace: tenant-a
spec:
  podSelector:
    matchLabels:
      kamaji.clastix.io/component: deployment
  policyTypes:
    - Ingress
  ingress:
    # Worker nodes and external clients, through the Service, Ingress or Gateway.
    - from:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.244.0.0/16
      ports:
        - port: 6443
        - port: 8132
    # Management cluster components that talk to the tenant API.
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kamaji-system
          podSelector:
            matchLabels:
              app.kubernetes.io/name: kamaji
              app.kubernetes.io/component: controller-manager
        # Cluster API core, in the namespace clusterctl installs it to.
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: capi-system
      ports:
        - port: 6443
```

The first rule admits clients outside the management cluster and keeps out the pods of the management cluster, which must be allowed one by one. If you expose the Tenant Control Plane through an Ingress or a Gateway, add the namespace of your ingress controller or Gateway implementation to the second rule, with port `8132` as well. Restrict who can reach the API from outside with `spec.networkProfile.loadBalancerSourceRanges` or your firewall, not with this rule (see the notes below).

## Egress from the Tenant Control Plane pods

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: tenant-control-plane-egress
  namespace: tenant-a
spec:
  podSelector:
    matchLabels:
      kamaji.clastix.io/component: deployment
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kamaji-system
          podSelector:
            matchLabels:
              app.kubernetes.io/name: kamaji-etcd
      ports:
        - port: 2379
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
```

With Konnectivity enabled, this is all a Tenant Control Plane needs: kubelets, webhooks and aggregated APIs are reached through the tunnel that the agents open towards port `8132`. Without Konnectivity, add the worker node network on port `10250` and the addresses of the tenant webhooks. For a DataStore outside the cluster, replace the first rule with an `ipBlock` of its endpoints and the port of its driver.

## Ingress to the DataStore

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: kamaji-etcd-ingress
  namespace: kamaji-system
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: kamaji-etcd
  policyTypes:
    - Ingress
  ingress:
    # etcd peers, the kamaji-etcd Jobs, the Kamaji controller and the DataStore migration Job.
    - from:
        - podSelector: {}
    # Tenant Control Plane pods, in any namespace.
    - from:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              kamaji.clastix.io/component: deployment
      ports:
        - port: 2379
```

The Kamaji controller connects to the DataStore too, to create the user, the role and the prefix of every Tenant Control Plane, and the DataStore migration Job runs in the Kamaji namespace. When `kamaji-etcd` shares that namespace, the first rule covers them. If you install it elsewhere, allow the Kamaji namespace explicitly. Add your Prometheus if it scrapes the etcd metrics port.

To allow only some Tenant Control Planes, for example those assigned to a given DataStore, add your own label through `spec.controlPlane.deployment.podAdditionalMetadata.labels` and select on it in the second rule.

## Notes

- **Address rewriting.** When traffic reaches a Tenant Control Plane through a LoadBalancer or NodePort Service, its source address may be rewritten to a node address, and whether `ipBlock` sees the original or the rewritten address depends on the network plugin. Do not rely on `ipBlock` to restrict external clients. Check what your CNI sees before narrowing the first ingress rule.
- **Node traffic.** Traffic from the node a pod runs on, which includes kubelet liveness and readiness probes, is always allowed and needs no rule.
- **Additional ports.** Ports from `spec.controlPlane.service.additionalPorts` are not covered by the examples. Add them to the ingress rule for the clients that need them.
- **Test with your CNI.** `NetworkPolicy` is enforced by the network plugin, and some plugins do not enforce it at all. Apply the policies to a test Tenant Control Plane first, then check that worker nodes join, `kubectl logs` and `kubectl exec` work, and the Tenant Control Plane stays `Ready`.
