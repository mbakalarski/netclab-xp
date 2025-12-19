---
marp: true
theme: default
paginate: true
backgroundColor: '#1E293B'  # dark slate
color: '#F8FAFC'             # light text
---

# 🚀 Introducing [**netclab-xp**](https://github.com/mbakalarski/netclab-xp)
*Declarative Router Configuration with Crossplane*  
<br>
Manage physical, virtual, and containerized routers *just like cloud resources* — with **Kubernetes-native GitOps workflows**.

---

# 🔧 Why netclab-xp Matters
<br>

- **Declarative & GitOps-ready** → manage routers like cloud resources  
- **OpenConfig foundation** → vendor differences handled automatically  
- **RESTCONF support** → starting with Arista EOS / cEOS  
- **Kubernetes-native state** → real-time reconciliation vs. Terraform  
- **Fast onboarding** → add new routers or services easily

---

# ⚙️ Abstractions
<br>

***High-Level Services*** <span style="color:#60A5FA">`FabricIP` / `EvpnService`</span>  
      ▼  
***Mid-Level Router Abstractions*** <span style="color:#60A5FA">`Router`</span>  
      ▼  
***Low-Level XRDs*** <span style="color:#60A5FA">`Loopback` / `RoutedInterface` / `BgpGlobal` / `BgpNeighbor`</span>  
      ▼  
***RESTCONF Provider*** <span style="color:#60A5FA">`provider-http`</span>

---

# RoutedInterface

---

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
  ipv4PrefixLength: 24
EOF
```

---

# Check device

```bash
kubectl exec -ti ceos01 -- Cli
```

```console
ceos01>en
ceos01#show run int Ethernet1
interface Ethernet1
   no switchport
   ip address 10.10.10.1/24
```

---

# Two routers with BGP

---

```bash
cat <<EOF | kubectl apply -f -
apiVersion: eos.netclab.dev/v1alpha1
kind: Router
metadata:
  name: ceos01
spec:
  nodePort: ceos01.default.svc.cluster.local:6020
  creds: YXJpc3RhOmFyaXN0YQ==
  asn: 65001
  routerId: 10.0.0.1
  routedInterfaces:
  - name: Ethernet1
    ipv4Address: 10.1.2.1
    ipv4PrefixLength: 24
  bgpNeighbors:
  - networkInstance: default
    neighborAsn: 65002
    neighborIp: 10.1.2.2
EOF
```

---

```bash
cat <<EOF | kubectl apply -f -
apiVersion: eos.netclab.dev/v1alpha1
kind: Router
metadata:
  name: ceos02
spec:
  nodePort: ceos02.default.svc.cluster.local:6020
  creds: YXJpc3RhOmFyaXN0YQ==
  asn: 65002
  routerId: 10.0.0.2
  routedInterfaces:
  - name: Ethernet1
    ipv4Address: 10.1.2.2
    ipv4PrefixLength: 24
  bgpNeighbors:
  - networkInstance: default
    neighborAsn: 65001
    neighborIp: 10.1.2.1
EOF
```

---

# Check resources

```bash
kubectl get routers
```

```bash
NAME     NODE-PORT                               ASN     ROUTER-ID   SYNCED   READY   COMPOSITION              AGE
ceos01   ceos01.default.svc.cluster.local:6020   65001   10.0.0.1    True     True    router.eos.netclab.dev   8m14s
ceos02   ceos02.default.svc.cluster.local:6020   65002   10.0.0.2    True     True    router.eos.netclab.dev   35s
```

---

# Check BGP on ceos01

```bash
kubectl exec ceos01 -- Cli -p15 -c "show ip bgp summary"
```

```console
BGP summary information for VRF default
Router identifier 10.0.0.1, local AS number 65001
Neighbor Status Codes: m - Under maintenance
  Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc PfxAdv
  10.1.2.2 4 65002             12        16    0    0 00:01:11 Estab   0      0      0
```

---

# Verify more SYNCED and READY

```bash
kubectl get netclab
```

```bash
NAME                                            NODE-PORT                               ASN     ROUTER-ID   SYNCED   READY   COMPOSITION                  AGE
bgpglobal.eos.netclab.dev/ceos01-2eedd4b1cab4   ceos01.default.svc.cluster.local:6020   65001   10.0.0.1    True     True    bgpglobals.eos.netclab.dev   3m52s

NAME                                              NODE-PORT                               NI        REMOTE-AS   REMOTE-IP   SYNCED   READY   COMPOSITION                    AGE
bgpneighbor.eos.netclab.dev/ceos01-3e0f1822de8d   ceos01.default.svc.cluster.local:6020   default   65002       10.1.2.2    True     True    bgpneighbors.eos.netclab.dev   3m52s

NAME                                            NODE-PORT                               SYNCED   READY   COMPOSITION                  AGE
iprouting.eos.netclab.dev/ceos01-8c1ecd5acc68   ceos01.default.svc.cluster.local:6020   True     True    iproutings.eos.netclab.dev   3m52s

NAME                                                    NODE-PORT                               NAME        SYNCED   READY   COMPOSITION                          AGE
loopbackinterface.eos.netclab.dev/ceos01-06a18fc6671a   ceos01.default.svc.cluster.local:6020   Loopback0   True     True    loopbackinterfaces.eos.netclab.dev   3m52s

NAME                                                  NODE-PORT                               NAME        IPV4-ADDRESS   IPV4-PREFIX   SYNCED   READY   COMPOSITION                        AGE
routedinterface.eos.netclab.dev/ceos01-95e43dc583a0   ceos01.default.svc.cluster.local:6020   Ethernet1   10.1.2.1       24            True     True    routedinterfaces.eos.netclab.dev   3m52s
routedinterface.eos.netclab.dev/ceos01-e2781936df2b   ceos01.default.svc.cluster.local:6020   Loopback0   10.0.0.1       32            True     True    routedinterfaces.eos.netclab.dev   3m52s

NAME                            NODE-PORT                               ASN     ROUTER-ID   SYNCED   READY   COMPOSITION              AGE
router.eos.netclab.dev/ceos01   ceos01.default.svc.cluster.local:6020   65001   10.0.0.1    True     True    router.eos.netclab.dev   3m52s
```

---

# 🎯 Get Started

[https://github.com/mbakalarski/netclab-xp](https://github.com/mbakalarski/netclab-xp)

- Try **examples** for Arista EOS / cEOS  
- Integrate with your GitOps workflows  
- Contribute & extend for other vendors  

<br><br>
#Kubernetes #Crossplane #network-automation #cloud-native #yang #Arista #restconf #infrastructure #control-plane #OpenConfig #IaC #DevOps #GitOps #NetDevOps
