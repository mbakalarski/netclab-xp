# netclab-xp

> ***Extend Kubernetes to manage any resource anywhere***
> <br>Powered by [crossplane.io](https://www.crossplane.io)

**Declarative Crossplane Configuration Package for Router Configuration Management**

`netclab-xp` enables declarative, GitOps-driven lifecycle management of physical, virtualized, and containerized routers using Crossplane.
It provides a layered abstraction over device-specific RESTCONF/YANG models, allowing network teams to manage router configuration consistently and reproducibly.

---

## Overview

`netclab-xp` is organized around a set of **layered abstractions**:

```
FabricIP / Evpn
     ↓
EosRouter
     ↓
Loopback / RoutedInterface / BgpGlobal / BgpNeighbor
     ↓
provider-http (RESTCONF)
```

- **High-Level Services:** Vendor-neutral abstractions such as `FabricIP` and `EvpnService`.
- **Mid-Level Abstractions:** Router constructs composed from multiple low-level XRDs (`EosRouter`).
- **Low-Level XRDs:** Vendor-specific resources directly representing YANG models (e.g., `RoutedInterface`, `BgpNeighbor`).
- **Lowest Layer:** Raw RESTCONF operations via `provider-http`.

This model hides vendor-specific complexity while enabling reusable, declarative network configuration.

---

## OpenConfig-Based XRDs

`netclab-xp` uses **OpenConfig** as the foundation for low-level XRD definitions.
Because many vendors include **YANG augments**, the repository organizes these XRDs into vendor-specific directories and API groups.

This approach enables:

* A consistent OpenConfig-style abstraction
* Support for vendor-specific extensions (BGP, interfaces, VLANs, EVPN, etc.)

---

## Benefits

* **Declarative operations** — manage network configuration as Kubernetes resources
* **GitOps-native** — versioned, reviewable router configuration
* **Vendor-neutral abstractions** — consistent workflows across platforms
* **Encapsulated complexity** — device-specific modeling hidden behind compositions
* **Composable and reusable** — easily onboard new routers and services

---

## Supported Devices

* **RESTCONF-enabled routers**
  *Currently supported:* Arista EOS
* **Other vendors**
  Not yet supported. Additional capabilities may be added via new providers or JSON-RPC through `provider-http` where applicable.

---

## Getting Started

### Requirements

* RESTCONF access to target routers
  (here provided via [`netclab-chart`](https://github.com/mbakalarski/netclab-chart))
* A Kubernetes cluster with Crossplane installed
  (here is the same cluster that supports the netclab-chart topology)
* The `netclab-xp` configuration package bundles all required crossplane dependencies.

### Installation steps

- Install [`netclab-chart`](https://github.com/mbakalarski/netclab-chart), this gives you the KinD cluster and tools

- Install Crossplane

```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

helm install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system --create-namespace
```

- Install the netclab-xp Configuration Package

```bash
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: netclab-xp
spec:
  package: xpkg.upbound.io/netclab/netclab-xp:v0.1.1
EOF
```

- Add ProviderConfig for Http provider

```bash
cat <<EOF | kubectl apply -f -
apiVersion: http.crossplane.io/v1alpha1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: None
EOF
```

---

## Example Usage

- Start netclab-chart topology with two cEOS routers
> You would observe retrying pods for jobs generating certs on cEOS.
> It is OK as booting of cEOS take time.

```bash
cat << EOF | helm install ceos netclab/netclab --values -
topology:
  networks:
  - name: b1
  - name: b2
    type: bridge
  - name: b3
    type: bridge
  nodes:
  - name: ceos01
    type: ceos
    memory: 2Gi
    cpu: 1000m
    interfaces:
    - name: eth1
      network: b1
    - name: eth2
      network: b2
  - name: ceos02
    type: ceos
    memory: 2Gi
    cpu: 1000m
    interfaces:
    - name: eth1
      network: b1
    - name: eth2
      network: b3
  - name: h01
    type: linux
    interfaces:
    - name: e1
      network: b2
  - name: h01
    type: linux
    interfaces:
    - name: e1
      network: b3
EOF
```

- Check topology state

```bash
kubectl get pod
```

```bash
NAME                         READY   STATUS      RESTARTS       AGE
ceos01                       1/1     Running     0              2m
ceos01-generate-cert-r7kml   0/1     Completed   4              2m
ceos02                       1/1     Running     0              2m
ceos02-generate-cert-nxfp4   0/1     Completed   4              2m
```

### RoutedInterface (Arista EOS)

- Get one interface configured on ceos01:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: eos.netclab.dev/v1alpha1
kind: RoutedInterface
metadata:
  name: r1e1ip
spec:
  nodePort: ceos01.default.svc.cluster.local:6020
  creds: YXJpc3RhOmFyaXN0YQ==
  name: Ethernet1
  ipv4Address: 10.10.10.1
  ipv4Prefix: 24
EOF
```

- Check resource state:

```bash
kubectl get netclab
```

Example output:

```bash
NAME                                     NODE-PORT                               NAME        IPV4-ADDRESS   IPV4-PREFIX   SYNCED   READY   COMPOSITION                        AGE
routedinterface.eos.netclab.dev/r1e1ip   ceos01.default.svc.cluster.local:6020   Ethernet1   10.10.10.1     24            True     True    routedinterfaces.eos.netclab.dev   7s
```

On the router:

```console
kubectl exec -ti ceos01 -- Cli
ceos01>en
ceos01#show run int Ethernet1
interface Ethernet1
   no switchport
   ip address 10.10.10.1/24
```

- Remove configuration of Ethernet1

```bash
kubectl delete routedinterface r1e1ip
```

---

### Or apply all the configuration from examples

Example manifests are available in the
[`examples/`](https://github.com/mbakalarski/netclab-xp/tree/main/examples) directory.

- Clone the repo and apply
```bash
git clone https://github.com/mbakalarski/netclab-xp ; cd netclab-xp
kubectl apply -f examples/eos/
```

- Check if all are SYNCED and READY:

```bash
kubectl get netclab
```

Example output:

```
NAME                               NODE-PORT                               ASN     ROUTER-ID   SYNCED   READY   COMPOSITION                  AGE
bgpglobal.eos.netclab.dev/ceos01   ceos01.default.svc.cluster.local:6020   65001   10.0.0.1    True     True    bgpglobals.eos.netclab.dev   22s
bgpglobal.eos.netclab.dev/ceos02   ceos02.default.svc.cluster.local:6020   65002   10.0.0.2    True     True    bgpglobals.eos.netclab.dev   22s

NAME                                       NODE-PORT                               NI        REMOTE-AS   REMOTE-IP   SYNCED   READY   COMPOSITION                    AGE
bgpneighbor.eos.netclab.dev/ceos01ceos02   ceos01.default.svc.cluster.local:6020   default   65002       10.1.2.2    True     False   bgpneighbors.eos.netclab.dev   22s
bgpneighbor.eos.netclab.dev/ceos02ceos01   ceos02.default.svc.cluster.local:6020   default   65001       10.1.2.1    True     True    bgpneighbors.eos.netclab.dev   22s

NAME                               NODE-PORT                               SYNCED   READY   COMPOSITION                  AGE
iprouting.eos.netclab.dev/ceos01   ceos01.default.svc.cluster.local:6020   True     True    iproutings.eos.netclab.dev   22s
iprouting.eos.netclab.dev/ceos02   ceos02.default.svc.cluster.local:6020   True     True    iproutings.eos.netclab.dev   22s

NAME                                          NODE-PORT                               NAME        SYNCED   READY   COMPOSITION                          AGE
loopbackinterface.eos.netclab.dev/ceos01lo0   ceos01.default.svc.cluster.local:6020   Loopback0   True     True    loopbackinterfaces.eos.netclab.dev   22s
loopbackinterface.eos.netclab.dev/ceos02lo0   ceos02.default.svc.cluster.local:6020   Loopback0   True     True    loopbackinterfaces.eos.netclab.dev   22s

NAME                                          NODE-PORT                               NAME        IPV4-ADDRESS   IPV4-PREFIX   SYNCED   READY   COMPOSITION                        AGE
routedinterface.eos.netclab.dev/ceos01e1      ceos01.default.svc.cluster.local:6020   Ethernet1   10.1.2.1       24            True     True    routedinterfaces.eos.netclab.dev   22s
routedinterface.eos.netclab.dev/ceos01lo0ip   ceos01.default.svc.cluster.local:6020   Loopback0   10.0.0.1       32            True     True    routedinterfaces.eos.netclab.dev   22s
routedinterface.eos.netclab.dev/ceos02e1      ceos02.default.svc.cluster.local:6020   Ethernet1   10.1.2.2       24            True     True    routedinterfaces.eos.netclab.dev   22s
routedinterface.eos.netclab.dev/ceos02lo0ip   ceos02.default.svc.cluster.local:6020   Loopback0   10.0.0.2       32            True     True    routedinterfaces.eos.netclab.dev   22s
```

On the router:

```bash
kubectl exec ceos01 -- Cli -p15 -c "show ip bgp summary"
```

```console
BGP summary information for VRF default
Router identifier 10.0.0.1, local AS number 65001
Neighbor Status Codes: m - Under maintenance
  Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc PfxAdv
  10.1.2.2 4 65002              5         5    0    0 00:01:13 Estab   0      0      0
```

- Remove configuration

```bash
kubectl delete -f examples/eos/
```

---

## Contributing & Extending

`netclab-xp` is designed to be extensible, and contributions are encouraged.
You can help improve or extend the project by:

* Adding new vendor-specific XRDs or OpenConfig extensions
* Creating high-level network abstractions (e.g., EVPN, fabric services)
* Enhancing router compositions and automation functions
* Contributing examples, tests, or documentation improvements
* Fixing issues or proposing enhancements via pull requests

---

## License

Licensed under the [Apache-2.0 License](LICENSE).
© 2025 Michal Bakalarski and Netclab Contributors.

---
