---
title: Memory Manager
authors:
  - "@bg-chun"
  - TBD
  - TBD
owning-sig: sig-node
participating-sigs:
  - sig-node
reviewers:
  - TBD
  - TBD
approvers:
  - TBD
  - TBD
editor: TBD
creation-date: 2020-02-03
last-updated: -
status: draft
see-also:
replaces:
superseded-by:
---

# Memory Manager

_Authors:_

- @bg-chun - Byonggon Chun &lt;bg.chun@samsung.com&gt;
- TBD&gt;
- TBD &gt;

## Table of Contents

- [Overview](#overview)
- [Motivation](#motivation)
  - [Related Features](#Related-features)
  - [Related issues](#Related-issues)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
  - [User Stories](#user-stories)
    - [Story 1](#story-1)
    - [Story 2](#story-2)
- [Proposal](#proposal)
  - [Proposed Changes](#proposed-changes)
    - [New Component: Memory Manager](#new-component-memory-manager)
      - [The Concept of Safety Zone](#the-concept-of-safety-zone)
      - [The Concept of Reserved Zone](#the-concept-of-reserved-zone)
      - [NUMA node affinity for memory and hugepages](#numa-node-affinity-for-memory-and-hugepages)
      - [Hint Interface](#hint-interface)
      - [OOM Score Enforcement](#oom-score-enforcement)
      - [CPUSET Enforcement](#cpuset-enforcement)
      - [New Interfaces](#new-interfaces)
    - [Changes of existing components](#changes-of-existing-components)
      - [Topology Manager changes](#topology-manager-changes)
      - [Internal Container Lifecycle changes](#internal-container-lifecycle-changes)
    - [Feature Gate and Kubelet Flags](#feature-gate-and-kubelet-flags)
- [Graduation Criteria](#graduation-criteria)
  - [Phase 1: Alpha (target v1.1x)](#phase-1-Alpha-target-v11x)
  - [Phase 2: Beta](#phase-1-Beta)
  - [GA (stable)](#ga-stable)
- [Appendix](#appendix)
  - [How DPDK works](#how-dpdk-works)
  - [Requirements to guarantee performance of DPDK](#requirements-to-guarantee-performance-of-dpdk)
  - [How Linux OOM system works](#how-oom-system-works)
  - [How Kubernetes utilizes Linux OOM system](#how-kubernetes-utilizes-linux-oom-system)

# Overview

NUMA Awareness is well known for a solution to boost performance for diverse use cases including DPDK and Database on multi socket machine. Since Kubernetes 1.16, it started to support NUMA aware resource allocation through Topology Manager for NUMA sensitive containers.
Topology Manager coordinates resource alignment by using hint interface that allows advertising the resource capability of NUMA node.
In kubenet, CPU Manager, Device Manager & Plugins advertise their resources with topology information as topology hint then Topology manager calculates best set of topology hints for containers.
But there's no feature for memory and hugepages. It leads kubelet to result NUMA Un-Aware memory and hugepages management for containers.
And it can cause inter-NUMA node communication for Memory access which leads to increase an I/O latency and decrease performance.

The Memory Manager is proposed to provide a way of guaranting NUMA awareness for memory and hugepages usage to kubelet.

# Motivation

## Related Features

- [Node Topology Manager][node-topology-manager] is a feature that collects topology hint from various hint providers to calculate NUMA affinity for a container. Topology Manager judge container whether admit or not under criteria of configured topology policy and calculated NUMA affinity.

- [CPU Manager][cpu-manager] is a feature provides a solution for CPU Pinning based on cgroups cpuset subsystem, it also offer topology hint to Topology Manager.

- [Device Manager][device-manager] is a one of features provides topology hint for Topology Manager. The main objectivity of Device Manager is allow vendors to adverise their resources and capability to kubelet.

- [Hugepages][hugepages] is a feature allows container to consume pre-allocated hugepages with pod isolation of huge pages.

- [Node Allocatable Feature][node-allocatable-feature] is a feature helps to reserve compute resources to prevent resource starvation. The kube-reserved and system-reserved is used to reserve resources for kubelet and system(OS, etc). At the moment(1.17), following resources are supported to reserve: cpu, memory, ephemeral storage.

## Related issues

- [Hardware topology awareness at node level (including NUMA)][numa-issue]
- [Support Container Isolation of Hugepages][hugepage-issue]
- [Support isolating memory to single NUMA node][memory-issue]

## Goals

- Guaranteeing isolation of Memory and Hugepages to single NUMA node for containers in the Guaranteed Pod.

- Providing topology hint for Memory and Hugepages to Topology manager, to make Topology Manager to be able to coordinate Memory and Hugepages with CPUs, I/O devices together.

## Non-Goals
- Updating scheduler, pod spec is out of scope at this point.

- This proposal only focus on Linux based system.

## User Stories

### Story 1 : Networking Acceleration using DPDK

- The system such as real-time trading system and 5G system, which requires networking acceleration, uses DPDK to guarantee low latency of packet processing. DPDK(Data Plane Development Kit) is set of libraries to accelerate packet processing on userspace. DPDK requires dedicated resources(such as exclusive cpu usage, hugepages and DPDK compatile Network Interface Card) and alignment of resources to sigle NUMA node to accelerate its performance. There should be a way to guarantee reserve Memory and Hugepages with other computing resources on desinated Socket for containerized Network Function which uses DPDK.

### Story 2 : Database

- Databases(Oracle, PostgreSQL, MySQL, MongoDB, SAP) consumes massive Memory and Hugepages, in order to reduce memory access latency and improve performance dramatically resources(CPU, Memory, Hugepages, I/O Devices) should be aligned.

# Proposal

## Proposed Changes

### New Component: Memory Manager
 Memory Manager is a new component of Kubelet, it guarantees NUMA aware memory usage for the Pod which has Guarantedd QoS, here memory includes hugepages.

 To guarantee NUMA aware memory usage, When Guaranteed QoS Pod admission is requested, Memory Manager caculates NUMA node affinity to figure out appropriate NUMA node where enough memory and hugepages exist for containers in the Pod. Then, when Pod deployed, it enfoces containers in the Pod to consume memory and hugepages from designated NUMA node which Topology Manager selected.

Note That :
- Memory Manager isolate Memory and Hugepages to single NUMA node to prevent remote(Inter-NUMA) memory access. For this, Memory Manager enforce cgroups cpuset subsystem using CRI API. For more details, see [CPUSET Enforcement](#cpuset-enforcement).

- Memory Manager guarantees that the container(in Guanteed QoS Pod) consumes requested amount of memory(which is isolated to a NUMA node) through linux OOM system. For more details, see [OOM Score Enforcement](#oom-score-enforcement)

More details of this component are listed below.

#### The Concept of Safety Zone
 Memory Manager has the concept of safety zone to manage memory and hugepages at the NUMA node level. Basically, Kubernetes support reserving memory at the node level through Node Allocatable feature. It provides options such as `system-reserved` and `kube-reserved` to reserve CPUs, Memory, Storage for system services and kubernetes system components. Fortunately, it is possible to reserve CPUs with consideration of NUMA because CPUs are reserved using their logical core number. On the other hand, it is impossible to reserve memory with consideration of NUMA, the only thing we can do is reserving memory at the node level.

 The concept of safety zone is proposed to reserve certain amount of memory and hugepages equally on each NUMA node. The value of safety zone can be confiugred by configuration parameter of Memory Manager. For more details see the [Feature Gate and Kubelet Flags](#feature-gate-and-kubelet-flags) section.
- For memory, total amount of sefety zone memory(value of safety zone memory * number of NUMA nodes) cannot be greater than total amount of reserved memory(kube-reserved, system-reserved and eviction-threshold) by Node Allocatable feature.

- For hugepages, at the moment(v1.17), Node Allocateble features does not support reserving hugepages, so that Memory Manager will not have safety zone for hugepages. When Node Allocatable feature start to support reserving hugepages([#83541](https://github.com/kubernetes/kubernetes/pull/83541)), safety zone for hugapages will works in the same manner of memory.

For example, if Node Allocatable feature reserved total 10Gbi memory on node which is two sockets server machine, safety zone memory cannot be greater than 5Gbi.

                  +-----------------+-------------------------+
NUMA node0(40Gi)  | Safety Zone(5Gi)|    Allocatable(35Gi)    |
                  +-----------------+-------------------------+
                  +-----------------+-------------------------+
NUMA node1(40Gi)  | Safety Zone(5Gi)|    Allocatable(35Gi)    |
                  +-----------------+-------------------------+
TBD: this text images will be replaced to real image.


#### The Concept of Reserved Zone
 Memory Manager has the concept of reserved zone to calculate NUMA node affinity precisely.
The Reserved zone is literally reserved for the containers which has Guaranteed QoS. Container doesn't always consume memory as much as they requested. For example container may request 5Gbi memory, but it can consumes only 3Gbi at runtime. Memory Manager treats requested memory of Guaranteed QoS containers as reserved memory. Memory Manager guarantee Guaranteed QoS containers usage of memory and hugepages on single NUMA node as they requested by using [CPUSET enforcement](#cpuset-enforcement) and [OOM Score Enforcement](#oom-score-enforcement). 
           
           +---------------+------------+--------------+
           |               |       Allocatable         |
NUMA node  +  Safety Zone  +------------+--------------+
           |               |  Reserved  |     free     |
           +---------------+------------+--------------+
TBD: this text images will be replaced to real image.


Note that 
- Reserved zone cannot be greater than Allocatable zone.
- Typical Linux processes and Containers which has BestEffort or Burstable QoS may consumes memory on a NUMA node where Guaranteed QoS containers also consumes memory. It is also possible that Guaranteed QoS Containers requested lots of memory but consumes just a little bit of them and the others occupy reserved zone when only few free memory is available on the NUMA node. At this situation, OOM may happen by Guaranteed QoS Container if the container try to consume more memory. Then the others will be killed by OOM killer to make free memory. Because, unlike Guaranteed QoS containers, the others has high OOM socre(which has low priority at OOM system).

#### NUMA node affinity for memory and hugepages
 Memory Manager calculates NUMA node affinity that represents which NUMA node has enough capacity of memory and/or hugepages for containers in Guaranteed QoS Pod. To calculate affinity, Memory Manager takes capacity of memory and pre-allocated hugepages per NUMA node on its intialize sequence then calculate `Allocatable Memory of NUMA node` for every NUMA nodes.  Then when pod admission is requested, Memory Manager checks the pod has Guaranteed QoS or not, if the pod has Guaranteed QoS, Memory Manager calculates NUMA node affinity by the following way: it checks resources availability of each NUMA nodes based on `Allocatable Memory of NUMA node` and total amount of `reserved` memory and hugepages of NUMA node.
 
Here are formulas that shows how to calculate NUMA node affinity for memory and hugepages.
- NUMA Node affinity of memory can be calcuated by below formulas.
   - Allocatable Memory of NUMA node = Total Memory of NUMA node - pre-allocated hugepages on the NUMA node - safety zone memory of NUMA node.
   - NUMA node Affinity for memory = (Allocatable Memory of NUMA node - Total amount of reserved memory of NUMA node) > requested memory of container ? true : false

- NUMA Node affinity of hugepage can be calculated by below formulas.
   - Allocatable hugepages of NUMA node is equal to Total amount of pre-allocated hugepages of NUMA node.
     - safety zone for hugepages will be supported later.
   - NUMA node Affinity for hugepages = (Allocatable hugepages of NUMA node - Total amount of reserved hugepages of NUMA node) > reqested hugepages of container ? true : false
  
#### Hint Interface
Topology Manager defines HintProvider interface to take node affinity of resources from resource managers(CPU, Device Manager). Memory Manager implements the interface to provide topology hint to Topology Manager.

#### OOM Score Enforcement
 TBD, this section will explain how Memory Manager guarantee a certain amount of memory on specific NUMA node to container, spoiler is low OOM score(low score has high priority) enforcement which kubelet already doing.

#### CPUSET Enforcement
Memory Manager restricts memory access of container(which has Guaranteed QoS) to a specific NUMA node by using cgroup cpuset subsystem. The subsystem provides the mechanism(`cpuset.cpus`, `cpuset.mems`) for assigning a set of cpus and memory nodes(it equals to NUMA nodes) to a set of linux process.  In kubelet, CPU Manager already uses `cpuset.cpus` to allocate exclusive logical cpu core to a container. Similar to CPU Manager, Memory Manager uses `cpuset.mems` to restrict containers memory access to specific memory node. 

TBD1: explaining relevant CRI API  
TBD2: explaining Topology Manager a little bit

Consequently, operator can expect Topology Manager(which configured single-numa policy) coordinates exclusive cpus, memory, hugepages to the same NUMA node.
 
#### New Interfaces

```go
package memory manager

type State interface {
  GetAllocatableMemory() uint64
  GetMemoryAssignments(containerID string) (memorytopology.MemoryTopology, bool)
  GetMachineMemory() memorytopology.MemoryTopology
  GetDefaultMemoryAssignments() ContainerMemoryAssignments
  SetAllocatableMemory(allocatableMemory uint64)
  SetMemoryAssignments(containerID string, mt memorytopology.MemoryTopology)
  SetMachineMemory(mt memorytopology.MemoryTopology)
  SetDefaultMemoryAssignments(as ContainerMemoryAssignments)
  Delete(containerID string)
  ClearState()
}

type Manager interface {
  Start(ActivePodsFunc, status.PodStatusProvider, runtimeService)
  AddContainer(p *v1.Pod, c *v1.Container, containerID string) error
  RemoveContainer(containerID string) error
  State() state.Reader
  GetTopologyHints(pod v1.Pod, container v1.Container) map[string][]topologymanager.TopologyHint
}

type Policy interface {
  Name() string
  Start(s state.State)
  AddContainer(s state.State, pod *v1.Pod, container *v1.Container, containerID string) error
  RemoveContainer(s state.State, containerID string) error
}
```

_Listing: Memory Manager and related interfaces (sketch)._

![memory-manager-interfaces](https://raw.githubusercontent.com/bg-chun/kep-draft/master/class_diagram.png)

_Figure: Memory Manager components._

![memory-manager-class-diagram](https://raw.githubusercontent.com/bg-chun/kep-draft/master/related_interfaces.png)

_Figure: Resource Managers with topology manager_

### Changes of existing components

#### Topology Manager changes

Topology Manager takes topology hints(bits that represents NUMA node) from resource managers such as cpu manager, device manager. Then, it calculates best affinity under given policy(preferred or strict) to determine pod admission. Likewise other resource managers, Memory Manager provide hints to Topology Manager.

Memory Manager implements 'Manager' interface, so that Topology Manager be able to add Memory Manager as hint provider at initialization sequence.

```go
package cm

func NewContainerManager(...) (ContainerManager, error) {
  ...
  cm.topologyManager.AddHintProvider(cm.memoryManager)
  ...
}
```

#### Internal Container Lifecycle changes

InternalContainerLifecycle interface defines following interfaces to manage container resources, PreStartContainer, PreStopContainer, PostStopContainer. In this cases, Memory Manager has to involve managing memory/hugepages when start/stop container.

Memory Manager manage memory with pod, container ID so these function needs that too.

```go
package cm

func (i *internalContainerLifecycleImpl) PreStartContainer(...) {
  ...
  err := i.memoryManager.AddContainer(pod, containerID)
  ...
}

func (i *internalContainerLifecycleImpl) PreStopContainer(...) {
  ...
  err := i.memoryManager.RemoveContainer(pod, containerID)
  ...
}

func (i *internalContainerLifecycleImpl) PostStopContainer(...) {
  ...
  err := i.memoryManager.RemoveContainer(pod, containerID)
  ...
}
```
### Feature Gate and Kubelet Flags

A new feature gate will be added to enable the Memory Manager feature. This feature gate will be enabled in Kubelet, and will be disabled by default in the Alpha release.

- Proposed Feature Gate:  
  `--feature-gate=MemoryManager=true`

This will be also followed by a Kubelet Flag for the Memory Manager configuration, which is described above. It will have following validation condition:(`TBD`).
- Proposed Configuration Flag:  
  `TBD`

# Graduation Criteria

## Phase 1: Alpha (target v1.19)
- Feature gate is disabled by default.
- Alpha implementation of Memory Manager based on SingleNUMA policy of Topology Manager

## Phase 2: Beta

- TBD

## GA (stable)

- TBD

# Appendix

## How DPDK works

TBD

## Requirements to guarantee performance of DPDK

TBD

## How OOM system works

TBD

## How Kubernetes utilizes Linux OOM system

TBD, spoiler: each QoS type has different OOM score.

[node-topology-manager]: https://github.com/kubernetes/enhancements/blob/dcc8c7241513373b606198ab0405634af643c500/keps/sig-node/0035-20190130-topology-manager.md
[cpu-manager]: https://github.com/kubernetes/community/blob/master/contributors/design-proposals/node/cpu-manager.md
[device-manager]: https://github.com/kubernetes/community/blob/master/contributors/design-proposals/resource-management/device-plugin.md
[hugepages]: https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/20190129-hugepages.md
[node-allocatable-feature]: https://github.com/kubernetes/community/blob/master/contributors/design-proposals/node/node-allocatable.md
[numa-issue]: https://github.com/kubernetes/kubernetes/issues/49964
[hugepage-issue]: https://github.com/kubernetes/kubernetes/issues/80716
[memory-issue]: https://github.com/kubernetes/kubernetes/issues/81009
