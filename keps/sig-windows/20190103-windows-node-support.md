---
title: Windows node support
authors:
  - "@astrieanna"
  - "@benmoss"
  - "@patricklang"
  - "@michmike"
owning-sig: sig-windows
participating-sigs:
  - sig-architecture
  - sig-node
reviewers:
  - sig-architecture
  - sig-node
  - sig-testing
  - sig-release
approvers:
  - "@bgrant0607"
  - "@michmike"
  - "@patricklang"
  - "@spiffxp"
editor: TBD
creation-date: 2018-11-29
last-updated: 2019-01-28
status: implementable
---

# Windows node support


## Table of Contents
<!-- TOC -->

- [Table of Contents](#table-of-contents)
- [Summary](#summary)
- [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
- [Proposal](#proposal)
    - [What works today](#what-works-today)
    - [What will work eventually](#what-will-work-eventually)
    - [What will never work (without underlying OS changes)](#what-will-never-work-without-underlying-os-changes)
    - [Relevant resources/conversations](#relevant-resourcesconversations)
    - [Risks and Mitigations](#risks-and-mitigations)
        - [Ensuring OS-specific workloads land on appropriate container host](#ensuring-os-specific-workloads-land-on-appropriate-container-host)
- [Graduation Criteria](#graduation-criteria)
- [Implementation History](#implementation-history)
- [Testing Plan](#testing-plan)
    - [Test Dashboard](#test-dashboard)
    - [Test Approach](#test-approach)
        - [Adapting existing tests](#adapting-existing-tests)
        - [Substitute test cases](#substitute-test-cases)
        - [Windows specific tests](#windows-specific-tests)
- [Conformance Testing](#conformance-testing)
- [API Reference](#api-reference)
    - [V1.Container](#v1container)
    - [V1.Pod](#v1pod)
- [Other references](#other-references)

<!-- /TOC -->

## Summary

There is strong interest in the community for adding support for workloads running on Microsoft Windows. This is non-trivial due to the significant differences in the implementation of Windows from the Linux-based OSes that have so far been supported by Kubernetes.


## Motivation

Windows-based workloads still account for a significant portion of the enterprise software space. While containerization technologies emerged first in the UNIX ecosystem, Microsoft has made investments in recent years to enable support for containers in its Windows OS. As users of Windows increasingly turn to containers as the preferred abstraction for running software and modernizing existing applications, the Kubernetes ecosystem stands to benefit by becoming a cross-platform cluster manager.

### Goals

- Enable users to schedule Windows Server containers in Kubernetes through the introduction of support for Windows compute nodes
- Document the differences and limitations compared to Linux
- Create a test suite in testgrid to maintain high quality for this feature and prevent regression of functionality 

### Non-Goals

- Adding Windows support to all projects in the Kubernetes ecosystem (Cluster Lifecycle, etc)
- Enable the Kubernetes master components to run on Windows
- Support for LCOW (Linux Containers on Windows with Hyper-V Isolation)

## Proposal

As of 29-11-2018 much of the work for enabling Windows nodes has already been completed. Both `kubelet` and `kube-proxy` have been adapted to work on Windows Server, and so the first goal of this KEP is largely already complete. 

### What works today
- Windows-based containers can be created by kubelet, [provided the host OS version matches the container base image](https://docs.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/version-compatibility)
    - Pod (single or multiple containers per Pod with process isolation). There are no notable differences in Pod status fields between Linux and Windows containers
    - Services types NodePort, ClusterIP, LoadBalancer, and ExternalName
    - Workload controllers ReplicaSet, ReplicationController, Deployments, StatefulSets, DaemonSet, Job, CronJob
    - ConfigMap, Secrets: as environment variables or volumes
    - Resource limits
    - Pod & container metrics
    - Horizontal Pod Autoscaling
    - Volumes can be shared between containers in a Pod
    - EmptyDir
- Windows Server 2019 is the only Windows operating system we will support at GA timeframe. Note above that the host operating system version and the container base image need to match. This is a Windows limitation we cannot overcome.
- Customers can deploy a heterogeneous cluster, with Windows and Linux compute nodes side-by-side and schedule Docker containers on both operating systems. Of course, Windows Server containers have to be scheduled on Windows and Linux containers on Linux
- Out-of-tree Pod networking with [Azure-CNI](https://github.com/Azure/azure-container-networking/blob/master/docs/cni.md), [OVN-Kubernetes](https://github.com/openvswitch/ovn-kubernetes), [two CNI meta-plugins](https://github.com/containernetworking/plugins), [Flannel (VXLAN and Host-Gateway)](https://github.com/coreos/flannel) 
- Dockershim CRI
- Many<sup id="a1">[1]</sup> of the e2e conformance tests when run with [alternate Windows-based images](https://hub.docker.com/r/e2eteam/) which are being moved to [kubernetes-sigs/windows-testing](https://www.github.com/kubernetes-sigs/windows-testing)
- Persistent storage: FlexVolume with [SMB + iSCSI](https://github.com/Microsoft/K8s-Storage-Plugins/tree/master/flexvolume/windows), and in-tree AzureFile and AzureDisk providers

### What we need to test and verify if it works or not for GA (items that are unsupported will be documented)
- Headless services (https://github.com/kubernetes/kubernetes/issues/73416)
- OOM reporting (https://github.com/kubernetes/kubernetes/issues/73417)
- QoS (guaranteed, burstable, best effort) (https://github.com/kubernetes/kubernetes/issues/73418)
- Is terminationGracePeriodSeconds supported (https://github.com/moby/moby/issues/25982)
- Pod DNS configuration like hostname, subdomain, hostAliases, dnsConfig, dnsPolicy (https://github.com/kubernetes/kubernetes/issues/73414)

### Windows Node Roadmap (post-GA work)
- Group Managed Service Accounts, a way to assign an Active Directory identity to a Windows container, is forthcoming with KEP `Windows Group Managed Service Accounts for Container Identity`
- `kubectl port-forward` hasn't been implemented due to lack of an `nsenter` equivalent to run a process inside a network namespace.
- CRIs other than Dockershim: CRI-containerd support is forthcoming
- Some kubeadm work was done in the past to add Windows nodes to Kubernetes, but that effort has been dormant since. We will need to revisit that work and complete it in the future.
- Calico CNI for Pod networking
- Hyper-V isolation (Currently this is limited to 1 container per Pod and is an alpha feature)
- It is unclear if the RuntimeClass proposal from sig-node will simplify scheduled Windows containers. We will work with sig-node on this.

### What will never work (Note that some features are plain unsupported while some will not work without underlying OS changes)
- Certain Pod functionality
    - Privileged containers
    - Pod security context privilege and access control settings. Any Linux capabilities or any POSIX capabilities are not supported (this includes SELinux, AppArmor, Seccomp, etc)
    - Reservations are not enforced by the OS, but overprovisioning could be blocked with `--enforce-node-allocatable=pods` (pending: tests needed)
    - Certain volume mappings
      - Single file & subpath volume mounting
      - Host mount projection
      - DefaultMode (due to UID/GID dependency)
      - readOnly root filesystem. Mapped volumes still support readOnly
      - Block device mapping
    - Termination Message - these require single file mappings
    - HugePages
    - Memory as the storage medium
- CSI plugins, which require privileged containers
- File system features like uui/guid, per-user Linux filesystem permissions, and read-only root filesystems
- NFS based storage/volume support
- Host networking is not available in Windows
- ClusterFirstWithHostNet is not supported for DNS. Windows treats all names with a `.` as a FQDN and skips PQDN resolution
- [Some parts of the V1 API](https://github.com/kubernetes/kubernetes/issues/70604)
- Not all features of shared namespaces are supported. This is clarified in the API section below
- The existing node problem detector is Linux-only and requires privileged containers. In general, we don’t expect these to be used on Windows because there’s no privileged support
- Overlay networking support in Windows Server 1803 is not fully functional using the `win-overlay` CNI plugin. Specifically service IPs do not work on Windows nodes. This is currently specific to `win-overlay`; other CNI plugins (OVS, AzureCNI) work. Since Windows Server 1803 is not supported for GA, this is mostly not applicable. We left it here since it impacts beta

### Relevant resources/conversations

- [sig-architecture thread](https://groups.google.com/forum/#!topic/kubernetes-sig-architecture/G2zKJ7QK22E)
- [cncf-k8s-conformance thread](https://lists.cncf.io/g/cncf-k8s-conformance/topic/windows_conformance_tests/27913232)
- [kubernetes/enhancements proposal](https://github.com/kubernetes/features/issues/116)


### Risks and Mitigations

**Second class support**: Kubernetes contributors are likely to be thinking of Linux-based solutions to problems, as Linux remains the primary OS supported. Keeping Windows support working will be an ongoing burden potentially limiting the pace of development. 

**User experience**: Users today will need to use some combination of taints and node selectors in order to keep Linux and Windows workloads separated. In the best case this imposes a burden only on Windows users, but this is still less than ideal. The recommended approach is outlined below, with one of its main goals being that we should not break compatibility for existing Linux workloads

#### Ensuring OS-specific workloads land on appropriate container host
As you can see below, we plan to document how Windows containers can be scheduled on the appropriate host using Taints and Tolerations. All nodes today have the following default labels (These labels will be graduating to stable soon)
- beta.kubernetes.io/os = [windows|linux]
- beta.kubernetes.io/arch = [amd64|arm64|...]

If a deployment does not specify a nodeSelector like `"beta.kubernetes.io/os": windows`, it is possible the Pods can be scheduled on any host, Windows or Linux. This can be problematic since a Windows container can only run on Windows and a Linux container can only run on Linux. The best practice we will recommend is to use a nodeSelector. 

However, we understand that in many cases customers have a pre-existing large number of deployments for Linux containers, as well as an ecosystem of off-the-shelf configurations, such as community Helm charts, and programmatic pod generation cases, such as with Operators. Customers will be hesitant to make the configuration change to add nodeSelectors. Our proposal as an alternative is to use Taints. Because the kubelet can set Taints during registration, it could easily be modified to automatically add a taint when running on Windows only (`“--register-with-taints=’os=Win1809:NoSchedule’” `). By adding a taint to all Windows nodes, nothing will be scheduled on them (that includes existing Linux Pods). In order for a Windows Pod to be scheduled on a Windows node, it would need both the nodeSelector to choose Windows, and a toleration.
```
nodeSelector:
    "beta.kubernetes.io/os": windows
tolerations:
    - key: "os"
      operator: "Equal"
      Value: “Win1809”
      effect: "NoSchedule"
```

## Graduation Criteria
- All features and functionality under `What works today` is fully tested and vetted to be working by SIG-Windows
- SIG-Windows has high confidence to the stability and reliability of Windows Server containers on Kubernetes
- 100% green/passing conformance tests that are applicable to Windows (see the Testing Plan section for details on these tests). These tests are adequate, non flaky, and continuously run. The test results are publicly accessible, enabled as part of the release-blocking suite
- Compatibility will not be broken, either for existing users/clusters/features or for the new features going forward, and we will adhere to the deprecation policy (https://kubernetes.io/docs/reference/using-api/deprecation-policy/).
- Comprehensive documentation that includes but is not limited to the following sections. Documentation will reside at https://kubernetes.io/docs and will adequately cover end user and admin documentation that describes what the user does and how to use it
1. Outline of Windows Server containers on Kubernetes
2. Getting Started Guide, including Prerequisites
3. How to deploy Windows nodes in Kubernetes
4. Overview of Networking on Windows
5. Links to documentation on how to deploy and use CNI plugins for Windows (example for OVN - https://github.com/openvswitch/ovn-kubernetes/tree/master/contrib)
6. Links to documentation on how to deploy Windows nodes for public cloud providers or other Kubernetes distributions (example for Rancher - https://rancher.com/docs//rancher/v2.x/en/cluster-provisioning/rke-clusters/windows-clusters/)
7. How to schedule Windows Server containers, including examples
8. Advanced: How to use metrics and the Horizontal Pod Autoscaler
9. Advanced: How to use Group Managed Service Accounts
10. Advanced: How to use Taints and Tolerations for a heterogeneous compute cluster (Windows + Linux)
11. Advanced: How to use Hyper-V isolation (not a stable feature yet)
12. Advanced: How to build Kubernetes for Windows from source
13. Supported functionality (with examples where appropriate)
14. Known Limitations
15. Unsupported functionality
16. Resources for contributing and getting help - Includes troubleshooting help and links to additional troubleshooting guides like https://docs.microsoft.com/en-us/virtualization/windowscontainers/kubernetes/common-problems

## Implementation History
- Alpha was released with Kubernetes v.1.5
- Beta was released with Kubernetes v.1.9

## Testing Plan


### Test Dashboard

All test cases will be built in kubernetes/test/e2e, scheduled through [prow](https://github.com/kubernetes/test-infra/blob/master/config/jobs/kubernetes-sigs/sig-windows/sig-windows-config.yaml), and published on the [TestGrid SIG-Windows dashboard](https://testgrid.k8s.io/sig-windows) daily. This will be the master list of what needs to pass to be declared stable and will include all tests tagged [SIG-Windows] along with the subset of conformance tests that can pass on Windows. The current dashboard will be renamed "SIG-Windows Release" for clarity [see #10989](https://github.com/kubernetes/test-infra/issues/10989)


Additional dashboard pages will be added over time as we run the same test cases with additional CRI, CNI and cloud providers. They are running the same test cases, and are not required for v1.14 graduation to stable.

- Windows Server 2019 on GCP - this is [in progress](https://k8s-testgrid.appspot.com/google-windows#windows-prototype)
- Windows Server 2019 with OVN+OVS & Dockershim
- Windows Server 2019 with OVN+OVS & CRI-ContainerD
- Windows Server 2019 with Azure-CNI & CRI-ContainerD
- Windows Server 2019 with Flannel & CRI-ContainerD

### Test Approach

The testing for Windows nodes will include multiple approaches:

1. [Adapting](#Adapting-existing-tests) some of the existing conformance tests to be able to pass on multiple node OS's. Tests that won't work will be [excluded](https://github.com/kubernetes/test-infra/blob/master/config/jobs/kubernetes-sigs/sig-windows/sig-windows-config.yaml#L69).
2. Adding [substitute](#Substitute-test-cases) test cases where the first approach isn't feasible or would change the tests in a way is not approved by the owner. These will be tagged with `[SIG-Windows]`
3. Last, gaps will be filled with [Windows specific tests](#Windows-specific-tests). These will also be tagged with `[SIG-Windows]`

All of the test cases will be maintained within the kubernetes/kubernetes repo. SIG-Windows specific tests for 2/3 will be in [test/e2e/windows](https://github.com/kubernetes/kubernetes/tree/master/test/e2e/windows)

Additional Windows test setup scripts, container image source code, and documentation will be kept in the [kubernetes-sigs/windows-testing](https://github.com/kubernetes-sigs/windows-testing) repo. One example is that the prow jobs need a list of repos to use for the test containers, and that will be maintained here - see [windows-testing#1](https://github.com/kubernetes-sigs/windows-testing/issues/1).
Building these containers for Windows requires a Windows build machine, which isn't part of the Kubernetes PR or official builds. If the SIG is given access to a suitable GCR.io account, images can be pushed there. Otherwise, we'll use continue pushing to Docker Hub.


#### Adapting existing tests

Over the course of v1.12/13, many conformance tests were adapted to be able to pass on either Linux or Windows nodes as long as matching OS containers are run. This was done by creating Windows equivalent containers from [kubernetes/test/images](https://github.com/kubernetes/kubernetes/tree/master/test/images). An additional parameter is needed for e2e.test/kubetest to change the container repos to the one containing Windows versions since they're not part of the Kubernetes build process yet.

These tests are already running and listed on the dashboard above, with a few exceptions:

- [ ] "... should function for node-pod communication: udp" - issue [#72917](https://github.com/kubernetes/kubernetes/issues/72917) has a PR open
- [ ] "should be able to pull image from docker hub" - [PR #72777](https://github.com/kubernetes/kubernetes/pull/72777) open
- [ ] "should provide DNS for the cluster" - [PR #72729](https://github.com/kubernetes/kubernetes/pull/72729) open for issue [#70189](https://github.com/kubernetes/kubernetes/issues/70189)


And also some cleanup to simplify the test exclusions:
 - [ ] Skip Windows unrelated tests - [Issue #69871](https://github.com/kubernetes/kubernetes/issues/69871), [PR#69872](https://github.com/kubernetes/kubernetes/pull/69872)

#### Substitute test cases

These are test cases that follow a similar flow to a conformance test that is dependent on Linux-specific functionality, but differs enough that the same test case cannot be used for both Windows & Linux. Examples include differences in file access permissions (UID/GID vs username, permission octets vs Windows ACLs), and network configuration (`/etc/resolv.conf` is used on Linux, but Windows DNS settings are stored in the Windows registry).

These test cases are in review:


- [ ] [sig-network] [sig-windows] Networking Granular Checks: Pods should function for intra-pod communication: http - [PR#71468](https://github.com/kubernetes/kubernetes/pull/71468)
- [ ] [sig-network] [sig-windows] Networking Granular Checks: Pods should function for intra-pod communication: udp - [PR#71468](https://github.com/kubernetes/kubernetes/pull/71468)
- [ ] [sig-network] [sig-windows] Networking Granular Checks: Pods should function for node-pod communication: udp - [PR#71468](https://github.com/kubernetes/kubernetes/pull/71468)
- [ ] [sig-network] [sig-windows] Networking Granular Checks: Pods should function for node-pod communication: http - [PR#71468](https://github.com/kubernetes/kubernetes/pull/71468)


And these still need to be covered: 

- [ ] DNS configuration is passed through CNI, not `/etc/resolv.conf` [67435](https://github.com/kubernetes/kubernetes/pull/67435)
  - Test cases needed for `dnsPolicy`: Default, ClusterFirst, None
  - Test cases needed for `dnsConfig`
  - Test cases needed for `hostname`
  - Test cases needed for `subdomain`

- [ ] Windows doesn't have CGroups, but nodeReserve and kubeletReserve are [implemented](https://github.com/kubernetes/kubernetes/pull/69960)



#### Windows specific tests

We will also add Windows scenario-specific tests to cover more typical use cases and features specific to Windows. These tests will be in [kubernetes/test/e2e/windows](https://github.com/kubernetes/kubernetes/tree/master/test/e2e/windows). This will also include density and performance tests that are adjusted for Windows apps which have different image sizes and memory requirements.

These areas still need test cases written:

- [ ] System, pod & network stats are implemented in kubelet, not cadvisor [70212](https://github.com/kubernetes/kubernetes/pull/70121), [66427](https://github.com/kubernetes/kubernetes/pull/66427), [62266](https://github.com/kubernetes/kubernetes/pull/62266), [51152](https://github.com/kubernetes/kubernetes/pull/51152), [50396](https://github.com/kubernetes/kubernetes/pull/50396)
- [ ] Windows uses username (string) or SID (binary) to define users, not UID/GID [64009](https://github.com/kubernetes/kubernetes/pull/64009)
- [ ] Create a `NodePort` service, and verify it's accessible on both Linux & Windows node IPs on the correct port [tracked as #73327](https://github.com/kubernetes/kubernetes/issues/73327)
- [ ] Verify `ExternalPort` works from Windows pods [tracked as #73328](https://github.com/kubernetes/kubernetes/issues/73328)
- [ ] Verify `imagePullPolicy` behaviors



## Conformance Testing

There were lots of discussions with SIG-Architecture and the Conformance working group on what Windows means for conformance. For the purposes of this KEP - graduating Windows node support to stable does not require conformance testing for v1.14, and will be completed later. This also means that clusters with Windows nodes will not be eligible for the conformance logo. During v1.14, SIG-Windows will be finishing the right set of tests so that we can propose changes to existing tests to make them OS agnostic, and what additional Windows-specific tests are needed. With continued work through the conformance working group, our goal would be to move these into a Windows conformance profile for v1.15. This would mean clusters could be tested and certified with only Linux nodes for 1.15+, no different from how they were run in <= 1.14. Windows nodes could be added and tested against the new conformance profile, but not all clusters will require Windows.


## API Reference

This section provides an API by API list of Windows & Linux differences. Issue [#70604](https://github.com/kubernetes/kubernetes/issues/70604) will be used to track updating the generated API docs with notes on Windows support where needed.

There are no differences in how most of the Kubernetes APIs work. The subtleties around what's different come down to differences in the OS and container runtime. Where a property on a workload API such as Pod or Container was designed with an assumption that it's implemented on Linux, then that may not hold true on Windows.

At a high level, these OS concepts are different:

- Identity - Linux uses userID (UID) and groupID (GID) which are represented as integer types. User and group names are not canonical - they are just an alias in /etc/groups or /etc/passwd back to UID+GID. Windows uses a larger binary security identifier (SID) which is stored in the Windows Security Access Manager (SAM) database. This database is not shared between the host and containers, or between containers.
- File permissions - Windows uses an access control list based on SIDs, rather than a bitmask of permissions and UID+GID
- File paths - convention on Windows is to use `\` instead of `/`. The Go IO libraries typically accept both and just make it work, but when you're setting a path or commandline that's interpreted inside a container, `\` may be needed.
- Signals - Windows interactive apps handle termination differently, and can implement one or more of these:
  - A UI thread will handle well-defined messages including [WM_CLOSE](https://docs.microsoft.com/en-us/windows/desktop/winmsg/wm-close)
  - Console apps will handle `ctrl-c` or `ctrl-break` using a [Control Handler](https://docs.microsoft.com/en-us/windows/console/registering-a-control-handler-function)
  - Services will register a [Service Control Handler](https://docs.microsoft.com/en-us/windows/desktop/Services/service-control-handler-function) function that can accept `SERVICE_CONTROL_STOP` control codes

These conventions are the same:
- Exit Codes mostly follow the same convention where 0 is success, nonzero is failure. The [specific error codes](https://docs.microsoft.com/en-us/windows/desktop/Debug/system-error-codes--0-499-) may differ. Exit codes passed from the Kubernetes components (kubelet, kube-proxy) will be unchanged.


The Windows container runtime also has a few important differences:

- Resource management and process isolation - Linux cgroups are used as a pod boundary for resource controls. Containers are created within that boundary for network, process and filesystem isolation. The cgroups APIs can be used to gather cpu/io/memory stats. Windows uses a Job object per container with a system namespace filter to contain all processes in a container and provide logical isolation from the host.
  - There is no way to run a Windows container without the namespace filtering in place. This means that system privileges cannot be asserted in the context of the host, and privileged containers are not available on Windows. Containers cannot assume an identity from the host because the SAM is separate.
- Filesystems - Windows has a layered filesystem driver to mount container layers and create a copy filesystem based on NTFS. All file paths in the container are resolved only within the context of that container.
  - Volume mounts can only target a directory in the container, and not an individual file.
  - Volume mounts cannot project files or directories back to the host filesystem.  
  - Read-only filesystems are not supported because write access is always required for the Windows registry and SAM database. Read-only volumes are supported
  - Volume user-masks and permissions are not available. Because the SAM is not shared between the host & container, there's no mapping between them. All permissions are resolved within the context of the container.
- Networking - The Windows host networking networking service and virtual switch implement namespacing and can create virtual NICs as needed for a pod or container. However, many configurations such as DNS, routes, and metrics are stored in the Windows registry database rather than /etc/... files as they are on Linux. The Windows registry for the container is separate from that of the host, so concepts like mapping /etc/resolv.conf from the host into a container don't have the same effect they would on Linux. These must be configured using Windows APIs run in the context of that container. Therefore CNI implementations need to call  the HNS instead of relying on file mappings to pass network details into the pod or container.



### V1.Container
- `V1.Container.ResourceRequirements.limits.cpu`
- `V1.Container.ResourceRequirements.limits.memory`


Windows doesn't use hard limits for CPU allocations. Instead, a share system is used. The existing fields based on millicores are scaled into relative shares that are followed by the Windows scheduler. [see: kuberuntime/helpers_windows.go](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/kuberuntime/helpers_windows.go), [see: resource controls in Microsoft docs](https://docs.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/resource-controls)
When using Hyper-V isolation (alpha), the hypervisor also needs a number of CPUs assigned. The millicores used in the limit is divided by 1000 to get the number of cores required. The CPU count is a hard limit.

Huge pages are not implemented in the Windows container runtime, and are not available. They require [asserting a user privilege](https://docs.microsoft.com/en-us/windows/desktop/Memory/large-page-support) that's not configurable for containers.

- `V1.Container.ResourceRequirements.requests.cpu`
- `V1.Container.ResourceRequirements.requests.memory`

Requests are subtracted from node available resources, so they can be used to avoid overprovisioning a node. However, they cannot be used to guarantee resources in an overprovisioned node. They should be applied to all containers as a best practice if the operator wants to avoid overprovisioning entirely.

- `V1.Container.SecurityContext.allowPrivilegeEscalation` - not possible on Windows, none of the capabilies are hooked up
- `V1.Container.SecurityContext.Capabilities` - POSIX capabilities are not implemented on Windows
- `V1.Container.SecurityContext.privileged` - Windows doesn't support privileged containers
- `V1.Container.SecurityContext.readOnlyRootFilesystem` - not possible on Windows, write access is required for registry & system processes to run inside the container
- `V1.Container.SecurityContext.runAsGroup` - not possible on Windows, no GID support
- `V1.Container.SecurityContext.runAsUser` - not possible on Windows, no UID support as int. This needs to change to IntStr, see [64009](https://github.com/kubernetes/kubernetes/pull/64009), to support Windows users as strings, or another field is needed. Work remaining tracked in [#73387](https://github.com/kubernetes/kubernetes/issues/73387)

- `V1.Container.SecurityContext.seLinuxOptions` - not possible on Windows, no SELinux

- `V1.Container.terminationMessagePath` - this has some limitations in that Windows doesn't support mapping single files. The default value is `/dev/termination-log`, which does work because it does not exist on Windows by default.


- `V1.Container.volumeMounts`

> TODO: check if mounting different volumes to containers in the same pod works. May need to go on test TODO list


### V1.Pod

> TODO: double check other fields in [source](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/api/core/v1/types.go#L2743)


- `V1.Pod.hostIPC`, `v1.pod.hostpid` - host namespace sharing is not possible on Windows

- `V1.Pod.hostNetwork` - There is no Windows OS support to share the host network

- `V1.Pod.dnsPolicy` - ClusterFirstWithHostNet - is not supported because Host Networking is not supported on Windows.

- `V1.podSecurityContext.runAsUser` provides a UID, not available on Windows
- `V1.podSecurityContext.supplementalGroups` provides GID, not available on Windows

- `V1.Pod.shareProcessNamespace` - this is an alpha feature, and depends on Linux cgroups which are not implemented on Windows

- `V1.Pod.volumeDevices` - this is an alpha feature, and is not implemented on Windows. Windows cannot attach raw block devices to pods.

- `V1.Pod.Volumes` - EmptyDir, Secret, ConfigMap, HostPath - all work and have tests in TestGrid
  - `V1.EmptyDirVolumeSource` - the Node default medium is disk on Windows. `memory` is not supported, as Windows does not have a built-in RAM disk.

- `V1.VolumeMount.mountPropagation` - only MountPropagationHostToContainer is available. Windows cannot create mounts within a pod or project them back to the node.


References: 

- [FlexVolume does not work on Windows node](https://github.com/kubernetes/kubernetes/issues/56875)
- [feature proposal add SMB(cifs) volume plugin](https://github.com/kubernetes/kubernetes/issues/56005)
- [add NFS volume support for Windows](https://github.com/kubernetes/kubernetes/issues/56188)




## Other references

[Past release proposal for v1.12/13](https://docs.google.com/document/d/1YkLZIYYLMQhxdI2esN5PuTkhQHhO0joNvnbHpW68yg8/edit#)
