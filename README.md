<br>

***Extend Kubernetes to manage any resource anywhere*** ([crossplane.io](https://www.crossplane.io/))

<br>

# netclab-xp

**Declarative Crossplane Configuration Package for Router Configuration Management**

`netclab-xp` enables declarative, GitOps-driven lifecycle management of physical, virtualized, and containerized routers using Crossplane.<br>
It provides a layered abstraction over device-specific RESTCONF/YANG implementations, allowing network teams to manage router configurations consistently.

---

## Overview

`netclab-xp` is organized around **layered abstractions**:

```
FabricIP/Evpn → EosRouter → Loopback/RoutedInterface/BgpGlobal/BgpNeighbor → provider-http (RESTCONF)
```

1. **Lowest-level:** RESTCONF requests via `provider-http`.
2. **Low-level:** Vendor-specific XRDs representing device YANG models, e.g., `BgpGlobal`, `LoopbackInterface`, `RoutedInterface`, `BgpNeighbor`.
3. **Mid-level:** Router abstractions combining low-level XRDs, e.g., `EosRouter`.
4. **High-level:** Network service abstractions, e.g., `FabricIP`, `EvpnService`.

This layered design hides vendor-specific complexity while enabling reusable and declarative network configuration.

> **Note:** Currently, only **RESTCONF-enabled devices** are supported.

---

## OpenConfig-based XRDs

`netclab-xp` uses OpenConfig models as the base for low-level XRDs.
Because many vendors add **vendor-specific YANG augments**, the repository organizes these XRDs into **separate folders per vendor** to handle the differences.
This approach allows an **OpenConfig abstraction** while still supporting vendor-specific extensions for BGP, interfaces, VLANs, and other router features.

---

## Benefits

* **Declarative** – manage router configuration like Kubernetes resources.
* **GitOps-ready** – track and apply changes via version control.
* **Vendor-neutral high-level abstractions** – hide device-specific differences where possible.
* **Encapsulated device complexity** – low-level YANG differences and augments handled automatically.
* **Composable and reusable** – add new routers or services with minimal effort.

---

## Supported Devices

* **RESTCONF-enabled devices**: Currently supported devices include Arista EOS routers.
* **Other vendors**: Devices without RESTCONF support are not currently managed by `netclab-xp`. Future support may require additional Crossplane providers or could leverage JSON-RPC through `provider-http` where applicable.

---

## Getting Started

Prerequisites:

* Kubernetes cluster with Crossplane installed
* RESTCONF access to target devices

The Crossplane configuration package references all required dependencies (Providers, Functions).

---

## Example Usage

Example manifests are available in the [`examples/`](https://github.com/mbakalarski/netclab-xp/tree/main/examples) folder.

### Low-level RoutedInterface configuration (Arista EOS)

```yaml
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
```

Apply the resource:

```bash
kubectl apply -f netclab-xp/examples/eos/routedinterface-example.yaml
```

Check the synced state:

```bash
kubectl get netclab

NAME                                     NODE-PORT                               NAME        IPV4-ADDRESS   IPV4-PREFIX   SYNCED   READY   COMPOSITION                        AGE
routedinterface.eos.netclab.dev/r1e1ip   ceos01.default.svc.cluster.local:6020   Ethernet1   10.10.10.1     24            True     True    routedinterfaces.eos.netclab.dev   7s
```

From the router CLI, the configuration is applied automatically:

```
ceos01#show run int Ethernet1
interface Ethernet1
   no switchport
   ip address 10.10.10.1/24
```

---

## Extending netclab-xp

* Add **high-level services**: e.g., `FabricIP`, `EvpnService`.
* Add **mid-level router XRDs**: combine low-level resources for new vendors.
* Add **low-level vendor-specific XRDs**: represent new YANG objects.
* Create new **Compositions** and **Functions** for automation.

---

## Contributing

Contributions are welcome! You can help by:

* Adding new vendor XRDs or OpenConfig-based resources
* Defining new high-level network services
* Improving Compositions, Functions, or documentation
* Fixing bugs and adding examples

---

## License

This project is licensed under the [Apache-2.0 License](LICENSE).<br>
Copyright 2025 Michal Bakalarski and Netclab Contributors.

---
