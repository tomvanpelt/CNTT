[<< Back](../../kubernetes)

# 4. Component Level Architecture
<p align="right"><img src="../figures/bogo_sdc.png" alt="scope" title="Scope" width="35%"/></p>

## Table of Contents
* [4.1 Introduction](#4.1)
* [4.2 Host OS](#4.2)
* [4.3 Kubernetes](#4.3)
* [4.4 Container runtimes](#4.4)
* [4.5 CNI plugins](#4.5)
* [4.6 Storage components](#4.6)
* [4.7 Service meshes](#4.7)
* [4.8 Container package managers](#4.8)
* [4.9 Supplementary components](#4.9)

<a name="4.1"></a>
## 4.1 Introduction

This chapter describes in detail the CNTT Kubernetes Reference Architecture in terms of the functional capabilities and how they relate to the Reference Model requirements, i.e. how the infrastructure profiles are determined, documented and delivered.

Figure 4-1 below shows the architectural components that are described in the subsequent sections of this chapter.

<p align="center"><img src="../figures/ch04_k8s_architecture.png" alt="Kubernetes Reference Architecture" Title="Kubernetes Reference Architecture" width="65%"/></p>
<p align="center"><b>Figure 4-1:</b> Kubernetes Reference Architecture</p>

<a name="4.2"></a>
## 4.2 Host OS

In order for a Host OS to be conformant with the Reference Architecture it must meet the following requirements:
- A version of the Linux kernel that is [compatible with kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/implementation-details/#kubeadm-init-workflow-internal-design) - this has been chosen as the baseline because kubeadm is concerned with nothing other than installing and managing the lifecycle of Kubernetes, hence it is easily integrated into higher-level and more complete tooling for the full lifecycle management of the infrastructure, cluster add-ons, and other tools and applications required for the full system.
- Windows Server 2019 (this can be used for worker nodes, but be aware of the [limitations](https://kubernetes.io/docs/setup/production-environment/windows/intro-windows-in-kubernetes/#limitations)).
- Support `req.gen.cnt.02` (immutable infrastructure), which means that the Host OS must be easily reproduced, consistent, disposable, with a repeatable deployment process, and will not have configuration or artifacts that are modifiable in place (i.e. once it is in a  running state).
- The selection of Host OS shall not restrict the selection of the OS used to build container images (container base image).

Table 4-1 lists the Linux kernel versions that comply with this Reference Architecture specification.

|OS Family|Version(s)|Notes|
|---|---|---|
|Linux|3.10+||
|Windows|1809 (10.0.17763)|For worker nodes only|

<p align="center"><b>Table 4-1:</b> Conformant OS Kernels</p>


<a name="4.3"></a>
## 4.3 Kubernetes
> * The version of version range of Kubernetes and the mandatory components needed for Kubernetes (e.g.: etcd, cadvisor)
> * Which optional features are used and which optional API-s are available
> * Which [alfa or beta features](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) are used

In alignment with the [Kubernetes version support policy](https://kubernetes.io/docs/setup/release/version-skew-policy/#supported-versions), a Reference Implementation must use one of three latest minor versions (`n-2`) - e.g. if the latest version is 1.17 then the RI must use either 1.17, 1.16 or 1.15. The Kubernetes distribution, product, or installer used in the RI must be listed in the [Kubernetes Distributions and Platforms document](https://docs.google.com/spreadsheets/d/1LxSqBzjOxfGx3cmtZ4EbB_BGCxT_wlxW_xgHVVa23es/edit#gid=0) and marked (X) as conformant for the Kubernetes version that is being used.

This Reference Architecture also specifies:

- Master nodes must run the following Kubernetes control plane services:
    - kube-apiserver
    - kube-scheduler
    - kube-controller-manager
- Master nodes can also run the etcd service and host the etcd database, however etcd can also be hosted on separate nodes if desired
- In order to support `req.gen.rsl.01`, `req.gen.rsl.02` and `req.gen.avl.01` a Reference Implementation must:
    - Consist of either three, five or seven nodes running the etcd service (can be colocated on the master nodes, or can run on separate nodes, but not on worker nodes)
    - At least one master node per availability zone or fault domain to ensure the high availability and resilience of the Kubernetes control plane services
    - At least one worker node per availability zone or fault domain to ensure the high availability and resilience of workloads managed by Kubernetes
- Master node services, including etcd, and worker node services (e.g. consumer workloads) must be kept separate - i.e. there must be at least one master node, and at least one worker node
- Workloads must ***not*** rely on the availability of the master nodes for the successful execution of their functionality (i.e. loss of the master nodes may affect non-functional behaviours such as healing and scaling, but components that are already running will continue to do so without issue). This function is essential for support of Edge type architectures.
- The following kubelet features must be enabled
    - CPU Manager
    - Device Plugin
    - Topology Manager

All kubelet features can be enabled/disabled by using the `feature-gates:` section in the kubelet config file.  e.g.
```
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
feature-gates:
  CPUManager: true|false (BETA - default=true)
  DevicePlugins: true|false (BETA - default=true)
  TopologyManager: true|false (ALPHA - default=false)
```

<a name="4.4"></a>
## 4.4 Container runtimes

The chosen runtime must be conformant with the [Kubernetes Container Runtime Interface (CRI)](https://kubernetes.io/blog/2016/12/container-runtime-interface-cri-in-kubernetes/) and the [Open Container Initiative (OCI) runtime spec](https://github.com/opencontainers/runtime-spec). Examples of container runtimes that are conformant with these specification in no particular order are:
- container-d (with CRI plugin enabled, which it is by default)
- Docker CE (via the dockershim, which is currently built in to the kubelet)
- CRI-O
- Frakti
The above list is by no means intended to be a complete listing of all the possible options for container runtimes.

To support `req.inf.vir.01` the architecture specifies the use of a container runtime with the capability for Kernel isolation:
- kata-containers

These specifications cover the [full lifecycle of a container](https://github.com/opencontainers/runtime-spec/blob/master/runtime.md#lifecycle) `creating > created > running > stopped` which includes the use of any storage required during the lifecycle of the container, including the management of the Host OS filesystem by the container runtime. This lifecycle management by the container runtime (when conformant with the above specifications) supports ephemeral storage for Pods.

To support the isolation of the resources used by the infrastructure from the resources used by the workloads the architecture specifies the use of the Kubernetes CPU Manager and [CPU Pooler](https://github.com/nokia/CPU-Pooler/).

> Todo: details and RA2 specifications relating to runtimes in order to meet RM features and requirements from RM chapters 4 and 5.

<a name="4.5"></a>
## 4.5 CNI plugins

As the selected CNI multiplexer/metapulgin MUST support other CNI plugins (`req.inf.ntw.06`) and SHOULD provide an API based solution to administer the networks from a central point (`req.inf.ntw.03`) the selected CNI multiplexer/metapulgin may be [DANM](https://github.com/nokia/danm).<br>

The following table contains a comparision of relevant features and requirements in Multus and DANM.

| Requirement | Support in Multus | Support in DANM |
|-------------|-------------------|-----------------|
| The overlay network encapsulation protocol needs to enable ECMP in the underlay (`infra.net.cfg.002`) | Supported via another CNI plugin | Supported via another CNI plugin |
| NAT (`infra.net.cfg.003`) | Supported via another CNI plugin | Supported |
| Security Groups (`infra.net.cfg.004`) | Not supported | Not supported <sub>1)<sub> |
| SFC support (`infra.net.cfg.005`) | Not relevant | Not relevant |
| Traffic patterns symmetry (`infra.net.cfg.006`) | Not relevant | Not relevant |
| Network resiliency (`req.inf.ntw.01`) | Supported | Supported |
| Centrally administrated and configured (`req.inf.ntw.03`) | Not supported | Partially suported |
| Dual stack IPv4 and IPv6 for Kubernetes workloads (`req.inf.ntw.04`) | Supported via another CNI plugin | Suported |
| Integrating SDN controllers (`req.inf.ntw.05`) | Supported via another CNI plugin | Supported via another CNI plugin |
| More than one networking solution (`req.inf.ntw.06`) | Supported | Supported |
| Choose whether or not to deploy more than one networking solution (`req.inf.ntw.07`) | Supported | Supported |
| Kubernetes network model (`req.inf.ntw.08`) | Supported via another CNI plugin | Supported via another CNI plugin |
| Do not interfere with or cause interference to any interface or network it does not own (`req.inf.ntw.09`) | Supported | Supported |
| Cluster wide coordination of IP address assignment (`req.inf.ntw.10`) | Supported via another CNI plugin | Supported |

<p align="center"><b>Table 4-2:</b> Comparision of CNI multiplexers/metaplugins</p>

1): Under implementation in the current release.  

 [Calico](https://github.com/projectcalico/cni-plugin) may be used as the CNI what complies with the basic networking assumptions of Kubernetes based on the requirement `req.inf.ntw.08` due to it's capability to handle `NetworkPolicies`, what is missing from [Flannel](https://github.com/coreos/flannel-cni).
For the network of signalling connections the built in IPVLAN CNI of DANM or the [MACVLAN CNI](https://github.com/containernetworking/plugins/tree/master/plugins/main/macvlan) may be used as these provide NAT-less connectivity. For the user plane network(s) the [User Space CNI](https://github.com/intel/userspace-cni-network-plugin) may be used. The User Space CNI may use VPP or OVS-DPDK as a backend.

> Editors note: The use of SR-IOV in container environments and, therefore, the inclusion of an SR-IOV CNI plugin and the [SR-IOV Device Plugin](https://github.com/intel/sriov-network-device-plugin) are still under debate.

<a name="4.6"></a>
## 4.6 Storage components

As described in [chapter 3](./chapter03.md), storage in Kubernetes consists of three types of storage:
1. Ephemeral storage that is used to execute the containers
    - **Ephemeral storage follows the lifecycle of a container**
    - See the [Container runtimes](#4.4) section above for more information how this meets the requirement for ephemeral storage for Pods
1. Kubernetes Volumes, which are used to present additional storage to containers
    - **A Volume follow the lifecycle of a Pod**
    - This is a native Kubernetes capability and therefore `req.inf.stg.01` is supported by default
    - This capability also delivers support for ephemeral storage although depending on the Volume Plugin used there may be additional steps required in order to remove data from disk (not all plugins manage the full lifecycle of the storage mounted using Volumes)
1. Kubernetes Persistent Volumes, which are a subset of the above whose lifecycle persists beyond the lifetime of a Pod to allow for data persistence
    - **Persistent Volumes have a lifecycle that is independent of Containers and/or Pods**
    - This supports the requirement `req.inf.stg.01` for persistent storage for Pods

Volume plugins are used in Kubernetes to allow for the use of a range of backend storage systems. There are two types of Volume plugin:
1. In-tree
    - These plugins are built, linked, compiled and shipped with the core Kubernetes binaries
    - Therefore if a new backend storage system needs adding this is a change to the core Kubernetes code
1. Out-of-tree
    - These plugins allow new storage plugins to be created without any changes to the core Kubernetes code
    - The Container Storage Interface (CSI) is such an out-of-tree plugin and many in-tree drivers are being migrated to use the CSI plugin instead (e.g. the [Cinder CSI plugin](https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/using-cinder-csi-plugin.md))
    - In order to support CSI, the following feature gates must be enabled:
      - `CSIDriverRegistry`
      - `CSINodeInfo`
    - In addition to these feature gates, a CSI driver must be used (as opposed to an in-tree volume plugin) - a full list of CSI drivers can be found [here](https://kubernetes-csi.github.io/docs/drivers.html)
    - In order to support ephemeral storage use through a CSI-compatible volume plugin, the `CSIInlineVolume` feature gate must be enabled
    - In order to support Persistent Volumes through a CSI-compatible volume plugin, the `CSIPersistentVolume` feature gate must be enabled

> Should the following paragraph be moved to the Security chapter?

> In order to support automation and the separation of concerns between providers of a service and consumers of the service, Kubernetes Storage Classes should be used. Storage Classes allow a consumer of the Kubernetes platform to request Persistent Storage using a Persistent Volume Claim and for a Persistent Volume to be dynamically created based on the "class" that has been requested. This avoids having to grant `create`/`update`/`delete` permissions in RBAC to PersistentVolume resources, which are cluster-scoped rather than namespace-scoped (meaning an identity can manage all PVs or none).

A note on object storage:
- This Reference Architecture does not include any specifications for object storage, as this is neither a native Kubernetes object, nor something that is required by CSI drivers.  Object storage is an application-level requirement that would ordinarily be provided by a highly scalable service offering rather than being something an individual Kubernetes cluster could offer.

> Todo: specifications/commentary to support req.inf.stg.04 (SDS) and req.inf.stg.05 (high performance and horizontally scalable storage). Also req.sec.gen.06 (storage resource isolation), req.sec.gen.10 (CIS - if applicable) and req.sec.zon.03 (data encryption at rest).


<a name="4.7"></a>
## 4.7 Service meshes

Service meshes are not in scope for the architecture.

<a name="4.8"></a>
## 4.8 Kubernetes Application package manager

The reference architecture specifies the use of a Kubernetes Application package manager using the Kubernetes API-s, such as [Helm v3](https://v3.helm.sh/).

<a name="4.9"></a>
## 4.9 Additional required components

> This chapter should list any additional components needed to provide the services defined in Chapter 3.2 (e.g: Prometheus)
