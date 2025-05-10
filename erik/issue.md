# Configuring IPAM InCluster networking with CAPI

# Description

I'm working to set up the latest Talos using kind on my laptop as the `capi`
mgmt cluster, deploying to Proxmox as my infra. I need to be able to utilize
the IPAM `InCluster` provider so that I can dynamically assign IPs to my nodes
from the IPAM pools. I was able to statically set the IP via the network details
in the `TalosControlPlane` patch, but hardcoding the IP into a template is not
viable for obvious reasons (I can't spawn multiple nodes doing this).

Ideally I should be able to embed a go template in the form of something like:

```
machine:
  network:
    interfaces:
      - interface: eth0
        addresses:
          - {{ index (where .Machine.Status.Addresses "type" "InternalIP") 0 "address" }}/24
```

The intent here is for the configuration template and the patch that applies
the network config for the node reads the address from the `Machine.status`,
which is where the IPAM provider writes the allocated IP:

```
status:
  addresses:
  - address: terra-workers-vhf9z-2s69m
    type: Hostname
  - address: 172.16.127.2
    type: InternalIP
```

The observed behavior is that the template string is literally rendered into
the talos machine config, which suggests to me that the bootstrap provider
doesn't support templating at all.

Am I doing something incorrectly here, or is this a defficiency of the provider
and I'd have to patch the functionality into the controller? What's do you guys
recommend?

# Component Stack

* `capi`
  - version: `v1.10.1
  - [link](https://github.com/kubernetes-sigs/cluster-api)
* `cluster-api-bootstrap-provider-talos`
  - version: `v0.6.9`
  - [link](https://github.com/siderolabs/cluster-api-bootstrap-provider-talos)
* `cluster-api-control-plane-provider-talos`
  - version: `v0.5.10`
  - [link](https://github.com/siderolabs/cluster-api-control-plane-provider-talos)
* `cluster-api-provider-proxmox` (infra provider: "capmox")
  - version: `v0.7.0`
  - [link](https://github.com/ionos-cloud/cluster-api-provider-proxmox)
* `cluster-api-ipam-provider-in-cluster`
  - version: `v1.0.1`
  - [link](https://github.com/kubernetes-sigs/cluster-api-ipam-provider-in-cluster)
* Talos Linux
  - version: `v1.10.1`
  - [link](https://github.com/siderolabs/talos)
* Kubernetes
  - version: `v1.33.0`
  - [link](https://github.com/kubernetes/kubernetes)
* `talosctl`
  - version
  - ```
    Client:
        Tag:         v1.10.1
        SHA:         52269e81
        Built:
        Go version:  go1.24.2
        OS/Arch:     linux/amd64
    ```
* Management Cluster
  - `kind v0.27.0 go1.23.6 linux/amd64`