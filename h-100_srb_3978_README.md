---
template_version: 2023-04-07 # DO NOT CHANGE -- this comes from the copied template.md

## These *MUST* be updated by the mentor as the proposal goes through the SRB process.
state: closed # open or closed
status_label: in-production # See https://github.ibm.com/cloudlab/srb#status-labels
last_update: 2024-11-12 # Set to today's date whenever the proposal is updated
---

<!-- cSpell: ignore MTFDKCC TNHR TEPX TEPS APISERVER RTTCC NACLs mcdvm baremetalproxy rocev VPCUI -->
<!-- cSpell: ignore EVNA CNAs serialnumber MTZPQQ Perinkulam Superspine OFED DCGMI HWCLASS -->

# SRB Proposal Template

# 1. Introduction

## 1.1 Project Name

H100 Based Training Systems & Cluster (Vela2)

## 1.2 Submitter and Contributors


## 1.3 Mentor(s)


## 1.4 Offering Manager(s)


## 1.5 Review Flow (determined by mentor)

Full

## 1.6 Production Target Date

3Q24

Aha Links:

- [VSIVPC-635 - 8 H100 per node based GPU Compute Profiles](https://bigblue.aha.io/features/VSIVPC-635)
- [CNWVPC-504 - Cluster API Support](https://bigblue.aha.io/features/CNWVPC-504)

VPC Requirement JIRAs:

- [Compute Req - VPCREQ-412](https://jiracloud.swg.usma.ibm.com:8443/browse/VPCREQ-412)
- [Platform Req - VPCREQ-415](https://jiracloud.swg.usma.ibm.com:8443/browse/VPCREQ-415)

PMCOM

- [PMCOM-5298 - H100 Server Enablement](https://jiracloud.swg.usma.ibm.com:8443/browse/PMCOM-5298)

# 2. Project Overview

## 2.1 Overview

IBM is in the business of building [AI models](https://www.ibm.com/artificial-intelligence). AI
models are built on training infrastructure, and used on inference infrastructure. In order to build
hundreds of billion parameter models, large scale clusters are needed to do so with state of the art
networking, accelerators, and servers.

IBM Cloud has provided Vela, which is an
[NVIDIA A100](https://www.nvidia.com/en-us/data-center/a100/) generation cluster built in WDC. It
has 400 Gbps of networking when using RoCE and GDR. However it is not large enough to support the
next generation set of models.

This project is to build the next generation training servers and clusters, built atop
[NVIDIA H100](https://www.nvidia.com/en-us/data-center/h100/) and up to 3.2 Tbps of networking
bandwidth per server. This will extend the baseline enablement done in the IBM Cloud VPC stack, but
provide a significant improvement to overall model training capability.

## 2.2 Consumer Interaction Model

This proposal introduces two new compute instance profiles and the concept of a cluster network. One
of the Instance Profiles is for research, the other for commercial clients. The research profile
helps with capacity management and increased velocity for firmware releases for their server set.

Additionally a new concept called `Cluster Networks` (API: `cluster_network`) will be introduced. In
select sites, customers will be able to create a cluster network - which represents a certain set of
performance criteria for a given set of interconnected systems. In this first cluster network
implementation, the interconnect between the servers will be 3.2 Tbps for the traffic through the
cluster network (GPU to GPU).

Not all sites that have the H100 Instance Profile will have a cluster network. Only sites where we
have strategically deployed a dedicated network will be able to provision a cluster network.

## 2.3 Requirements and Use Cases

- Develop a scalable H100 GPU instance deployment design:
  - Deploy as stand alone inference servers
  - Deploy within a physically independent aggregation switch network which will be used for Cluster
    Network enabled workloads. Details on Cluster Network customer facing details later in
    requirements / sections.
    - Scale down to a minimum size of 12 racks (36 servers / 288 GPUs)
    - Enable a scalable design up to 250 racks (750 servers / 6000 GPUs)
  - Support placement groups with node and rack anti-affinity
- Create a `gx3d-160x1792x8h100` instance profile and associated pricing plan
  - Must support up to 15 VNI/vNICs (north/south) with full network capabilities
  - Have almost 1.8 TB memory
  - Have 160 vCPUs which run atop the 8474C processor
- Research must have a unique instance profile that allows them to deploy on servers with slightly
  newer firmware
  - This firmware will be deployed quicker to the research nodes, but eventually rolled back to the
    gen pop nodes
  - The new instance profile for research will be called `gx3d-160x1792x8h100-research`
- Must support deploying instances into an optional cluster network which provides a dedicated
  network for accelerator traffic
  - Users explicitly opt-in to using the cluster network by defining their cluster network, subnets,
    etc.
  - They may choose not to use a cluster, in which case, the instance does not connect any of the
    cluster NICs.
- Develop new billing parts for the H100
  - Will use the Gen 3 billing strategy
  - Will support 1 and 3 year reservations
    - Reservations will be supported for both clustered and non-clustered H100 instances.
- Support both commercial and research clients
  - Research will consume this infrastructure "as a typical IBM Cloud customer"

Restrictions:

- The new instance profiles are expected to tak about 20 minutes to start.
  - The I/O connections on the system, and PCIe remapping take a long time for VSI start.

### 2.3.1 Cluster Use Cases

For users opting to use cluster networking, the following must be supported.

**NOTE**: During the server commitment, only the server will be able to be provisioned for inference
workloads. The cluster use cases will be in a subsequent commit. The cluster network will not be
available during the server-only window.

- As a user, I need to create a cluster network which supports a specific performance specification
  - The cluster network must be provisioned from a cluster network profile, which will indicate a
    specific performance envelope
  - The cluster network profile type for this specification will be `h100`
  - The `h100` cluster network profile type will indicate:
    - A native 400 GigE underlying fabric (2x200 GigE)
    - Support for GDR and RoCE
  - The profile will describe a type, and documentation can describe for a given profile type what
    performance envelope they should expect.
- As a user, I will not be able to create a cluster network unless it is in a region with an
  available `cluster_network_profile`.
- As a user, I must be able to delete a cluster network once all instances attached to it are
  deleted.
  - If there are active resources in a cluster network, the cluster network will fail to delete
  - If there are no active instances, but there are subordinate resources (ex. `subnets` attached to
    the `cluster_network`), then the subordinate resources can cascade delete.
- As a user, I need to be able to define a subnet which is attached to the cluster network.
  - A cluster network must be able to support a set of subnets.
- As a user, I must be able to determine which types of cluster network profiles a given Instance
  Profile can connect to

  - The `gx3d-160x1792x8h100` instance profile will need to indicate it can connect to a `h100` type
    cluster.
  - In the future, we expect that a storage enabled server profile (likely bare metal servers) will
    also be able to connect directly into the cluster. It will provide GPUDirect storage over the
    RDMA network to the training servers.
  - That storage enabled server may be able connect to several different types of cluster networks.
    Such as a `h100`, `mi300x` or `gaudi3` cluster type.
  - Clusters are only available in certain zones across the global footprint. The user must be able
    to discover which zones have the corresponding cluster network capabilities.

- As a user, I must have a consistent deployment pattern of the `cluster_network_attachments` across
  a set of instances.

  - Ex. if the user deploys with 8 `cluster_network_attachments` for a set of instances, they must
    align across a consistently (same PCIe addresses within the VM) across the all of the instances.

- As a user, I must be able to know the quantity of `cluster_network_attachments` that can be set on
  a given instance

  - The instance profile for `gx3d-160x1792x8h100` will support 0, 8, 16 or 32
    `cluster_network_attachments`. This must be expressed on the instance profile.
  - The user must set exactly that quantity of `cluster_network_attachments` on a given instance. If
    they set a value that is outside the approved range, the API will throw an error.
  - While 8 `cluster_network_attachments` is the expected normal quantity, some power users that
    wish to increase the entropy in their ECMP paths may use more. However the quantities are
    constrained by the instance profile.

- As a user, I must be able to reserve an IP for my `cluster enabled vNICs`

  - There are various libraries which use IP address schemes to try to determine locality with other
    servers.
  - Providing the user with the ability to set the IP address can affect their ability to influence
    certain algorithms that may be behind proprietary accelerator software boundaries.

- As a user, the network interfaces in my instance must present themselves as 400 GigE with GPU
  Direct and RoCE v2 capable.

  - This requirement is the specific network interface within the instance (virtual machine).
  - The instance profile must specify the type of cluster networks it can connect to and the
    quantity and speed of the `cluster_network_attachments`.
  - The virtual machines network interface must also be SR-IOV to achieve this speed. The virtual
    functions will connect directly to the VSI.
  - When the number of cluster_network_attachments exceeds the number of physical cluster network
    cards, the bandwidth will be `pooled` on a given physical network interface card.
    - Ex. if the user specifies 16 `cluster_network_attachments`, there will be 8 sets of two, and
      each set will share 400 GigE of bandwidth.

- As a user, routing between the `subnets` attached to the `cluster_network` is supported.

  - Typical cluster topologies allow for traffic across subnets within the cluster.
  - However, the routing domain is within the cluster itself and not tout to the VPC.

- As a user, I must be able to query the REST APIs to understand the IP addresses of the interfaces
  attached to the cluster network for a given instance.

  - Users will utilize [Compute Resource Identity](../1494/README.md#212-compute-resource-identity)
    with control nodes. They are anticipated to create a logical control node which learns about and
    monitors the cluster.
  - It is also useful for the users building custom monitoring dashboards to have additional
    metadata available to them at the API to describe their instances and their intent.

- As a user, the cluster networking for a given instance must also be expressed via the
  [metadata API](../1494).

  - Users have the option to learn more about their instance through metadata rather than through
    the RCOS API. The metadata API will be updated to reflect the cluster network attachments.

- As a user, I must be able to configure IP Spoofing on my `cluster network interfaces`

  - IBM Research runs a custom container network solution for some of it's cluster. This solution
    can override the IPs on the cluster network. When they do this, they run the same challenges as
    one has on the cloud network with IP spoofing. But it is critical for portions of their
    workload.

- There are no additional charges for cluster networks.

- As a user, I will be able to create up to 10 cluster_networks.

  - If the user needs more than 10, they will request a quota increase.

- As a user, I will be able to use an overlapping address space between my VPC and my cluster
  network.
  - The cluster network will have a set of subnet_prefixes. These can overlap with the VPC network,
    but the defaults will be an independent address space from the default VPC address space.
  - By default, the address spaces will not overlap. But the user must be able to have it overlap if
    needed.

Restrictions:

- As a user, I will only be able to attach my `cluster network interfaces` when the instance is
  being created or when the instance is stopped.

  - Specifically, the `cluster_network_attachments` on an instance can only be set when the instance
    is stopped or created.

- Users will not be able to configure `cluster network interfaces` for a reduced set of accelerators
  on the H100 profile

  - Specifically, this instance profile has 8 GPUs. The only approved quantity of
    `cluster network interfaces` for this profile are:
    - 0: No cluster networking
    - 8: A single cluster network interface per accelerator
    - 16: Two cluster network interfaces per accelerator (increased entropy)
    - 32: Four cluster network interfaces per accelerator (significantly increased entropy)
  - Users will not be able to start an instance with a different quantity than what is noted above
    and documented publicly.

- A given instance can only connect to a single `cluster network`.

  - If the user specifies a set of `cluster_network_attachments` which spread across multiple
    `cluster_networks`, then the API call must fail.

- As a user, I must be able to set Security Groups and configure Network Access Control Lists
  (NACLs) for my `interfaces` and `subnets` attached to the `cluster_network`

  - The API complexity significantly increases including security groups and NACLs in the design.
    However a default security group allowing all customer-driven communication across their
    isolated logical cluster will be created in the control plane. If/when IBM determines to open up
    security groups and NACLs within the cluster, the underlying security group providing
    communication will be shown.
  - Effectively, a security group will be created that is unique to each cluster. It is simply not
    customer managed or customer observable.

- As a user, I must be aware that my traffic running on `subnets` attached to a `cluster_network`
  will not route into standard cloud subnets.

  - The traffic must be isolated within the subnets on the cluster network, otherwise, the traffic
    will hit the super spine.
  - If the traffic hits the super spine, there may be stability issues with the cloud. Additionally
    the performance profile of the physical cluster network becomes compromised.
  - To ensure traffic is isolated, the host has 8 physical NICs for cluster network traffic and 1
    physical NIC for VPC traffic. In the physical network implementation, logical rails have been
    created in the aggregation switches to ensure the cluster network traffic routes through a
    deterministic destination port. Additionally, the spine switches are configured to drop RDMA
    traffic if its route is out to super spine or GBL. The combination of the 8 physical NICs,
    along with the logical rails implemented at the aggregation switch, and the spine switch
    configuration to drop incorrectly routed RDMA traffic, prevents cluster network traffic from
    hitting the super spine. For further details, see section
    [3.1.1](#3116-sdn-architecture) and [section 3.1.1.3.3](#311331-vrf-and-logical-rails).

- As a user, I must _not_ be able to use certain SDN features with my `subnets` attached to my
  `cluster_network` or the `interfaces` within the `cluster_network`. This must be documented
  publicly as well. The disablement of these features ensures the cluster network traffic remains
  isolated to the logical cluster network and can not egress to the broader cloud network. These SDN
  features would egress traffic outside of the cluster network, thus impacting the performance and
  stability of the network.

  - See the [Out-of-Scope section](#out-of-scope---sdn) for the features and explanation.
  - Two categories for limitation:

    - Those that would egress outside of the cluster, thus impacting stability and performance
      - These limitations will be documented as `limitations when using a cluster network`
      - limitations:
        - Public Gateway support
        - VPN support
        - LBaaS support
        - Floating IP support
        - Dynamic Route Servers
        - Endpoint Gateways
        - Private Path
    - Technical Server Issues
      - These limitations must be documented as
        `limitations when using this instance profile in a cluster network`
      - limitations:
        - Flow Logs

  - There will be a restriction of a single physical cluster of a given type (ex. H100) per zone for
    the initial release.
    - A subsequent SRB will introduce the ability to have multiple physical clusters of the same
      type within a zone.
    - Multiple physical cluster support would be needed if you had two H100 clusters, perhaps in
      separate DC rooms (and therefore on a different spine).
    - For this release, all deployments are either in zones without a physical cluster (single node
      only) or all of the hosts in that zone are connected to the physical cluster.

### 2.3.2 Research Acceptance Criteria

A large set of the server [deployments](#261-deployments) are earmarked for the research team. They
have outlined the following acceptance criteria:

Note:

- Research can not perform meaningful business work on the cluster until they have indicated that
  they accept that the acceptance tests have completed.

Acceptance Criteria:

- [DCGMI -r 4](https://docs.nvidia.com/datacenter/dcgm/latest/release-notes/changelog.html) passes
  on the individual nodes. This is an individual node diagnostics test.
- [Single Node HPL](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/hpc-benchmarks) tests
  pass.
  - Acceptance must be within 90% of the BlueVelas baseline.
  - BlueVela's HPL result for single node is:
    - Max FLOPS (GFlop/s): 371,500
    - Min FLOPS (GFlop/s): 363,800
- [MPI Cluster Kit](https://docs.nvidia.com/networking/display/hpcxv216/clusterkit) needs to be
  validated with 128 nodes.
  - Performance must be within 80% of the Blue Velas baseline
  - Performance data captured below.
- [NCCL Functional Test](https://github.com/NVIDIA/nccl-tests)
  - Performance must deliver 320 Gbps per cluster network interface (assuming a cardinality of 8
    NICs per VSI)
  - Acceptance hand over will be run with 128 nodes. Research reserves the right to wave this as a
    gate if they wish to receive the systems earlier and this 128 node limit is a gate.
- [Multi Node HPL Functional Test](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/hpc-benchmarks)
  - Must deliver scaling of at least 75% for 128 servers (or more)

All multi-node tests should be able to run with up to 150 nodes. However, 128 is being used as the
threshold as there will likely be multiple concurrent nodes being tested at once.

![BlueVelas Cluster Kit Baseline](./assets/bluevelas_clusterkit_baseline.png)

### 2.3.3 Beta and GA phases

Since the scope of the SRB covers both the H100 profile as well as the cluster networking service,
the beta and GA phases for these will be staggered.

H100 Profile [SRB commit review](https://github.ibm.com/cloudlab/srb/pull/4771):

- Beta:
  - Start Date: 01 August 2024
  - End Date: 1 October 2024
- GA:
  - Start Date: 1 October 2024

Cluster Network Service:

- Beta:
  - Start Date: 15 October 2024
  - End Date: 01 December 2024
- GA:
  - Start Date: 01 December 2024

Note: See the Lessons Learned section.

## 2.4 Out-of-Scope

### Out of Scope - Compute

- Live migration of instances
- On prem H100 design
  - There is a second H100 training solution being built on prem. It's design is outside the scope
    of this document.
- These servers will not support Dedicated Hosts
  - each instance profile consumes the entire physical host, therefore use of dedicated hosts does
    not functionally add value
- Cluster Network Subnets will not have default root prefixes
- FedRamp support

  - The CX-7 NICs that need to be used in this configuration will not have FIPs compliance.
  - The only path to support would be to move to BlueField in NIC mode
  - BlueFields in NIC mode with dual port do not physically fit within these servers.

### Out of Scope - SDN

Since the cluster network is meant for ultra-fast communication between devices, many network
features will not be supported.

- Features explicitly not supported to ensure traffic stays pinned within the physical cluster
  network aggregation switch complex:
  - Public Gateway support
  - VPN support
  - LBaaS support
  - Floating IP support
  - Dynamic Route Servers
  - Endpoint Gateways
  - Private Path

These features are not supported in a cluster due to limitations with SR-IOV Virtual Functions

- Hot attach/detach of accelerated NICs.
- Flow Logs
  - This will impact FS Cloud compliance. Use of cluster networking will not be FS Cloud compliant.

Additionally, it should be noted that provisioning of a `cluster_network` does not guarantee cluster
capacity. Capacity will be determined at instance provisioning time, and if there is no capacity, a
corresponding instance `status_reason` will be provided indicating insufficient capacity.

For scope purposes, the following will be out of scope initially for `subnets` and `interfaces`
within a `cluster network`. Customer input has indicated they are not needed in the cluster network.
However, if a use case presents itself, we could add as part of a follow up proposal.

- Custom Routing Tables
  - Don't allow custom routing tables on the cluster subnets.
- Secondary IPs

  - Research has indicated they do not need this for the accelerator NICs

- As an operator, I will not be able to specify to which `physical cluster` a given customer cluster
  network maps to
  - In the future, we could have multiple physical cluster network implementations in the same zone
    of a given `type`.
  - In the event of that, the operations team will need an enhanced `cluster scheduler` developed
    and the ability to override which physical cluster network implementation a users
    `cluster network` (virtual) object points to.
  - For this proposal and the [deployments](#261-deployments) specified, each zone will have at most
    one physical cluster of a given type.
  - Advanced Cluster scheduling and overrides will come as part of a separate proposal, when we have
    two clusters of the same type within a zone.

While pooled bandwidth is a requirement for the `cluster_network_attachments`, pooled bandwidth for
standard `network_attachments` is out of scope for this proposal. See the
[bandwidth pooling](hld/bandwidth-pooling.md) document for additional details/history.

Lastly, all new functions in SDN should by default be assumed to not be supported in a cluster
network. As an example, Network Packet Mirroring
([CNWVPC-495](https://bigblue.aha.io/features/CNWVPC-495)) would be only for the standard network
and not for the cluster network. Network requirements can be evaluated for inclusion, but in general
broad connectivity requirements and capabilities which may severely limit performance are not
candidates for the cluster network. Security requirements (ex. encrypted subnets) need to be
evaluated. They may be available in future cluster network profile types, but not prior versions.

## 2.5 Dependencies, Dependents, and Incompatibilities

### 2.5.1 Other VPC Projects Under Development

- [RoCE / GPU Direct](https://github.ibm.com/cloudlab/srb/pull/3860)
- [Compute Metrics Agent](https://github.ibm.com/cloudlab/srb/pull/4423)
  - This project improves the visibility of the GPUs, and is being extended as the basis to improve
    accelerator (and compute health) holistically. These benefits extend to Vela2, Vela 1, and the
    majority of the fleet. This project was kicked off with the intent to fill the observability
    issues with Vela 1 and improve the baseline for Vela2.

### 2.5.2 Other IBM Projects Under Development

IBM RedHat OpenShift Kubernetes (ROKS) support will be added to take advantage of this
infrastructure. It is a follow-on delivery dependent on the work here.

### 2.5.3 Third-party

There are several hardware dependencies that are needed for this project.

<!-- cSpell: disable -- disables checking -->

| Manufacturer | Type           | Model                     | SMC Part Num      | Status in Fleet |
| ------------ | -------------- | ------------------------- | ----------------- | --------------- |
| Supermicro   | System / Board | SYS-821GE-TNHR            | SYS-821GE-TNHR    | New             |
| Micron       | Disk           | MTFDKCC3T2TFS-1BC45ABYY   | N/A               | Existing        |
| Intel        | Processor      | 8474C                     | N/A               | Existing        |
| NVIDIA       | GPU            | H100 SXM                  | N/A               | New             |
| NVIDIA       | NIC            | CX-7 - 900-9X7AH-0079-DTZ | AOC-CX7AH0079-DTZ | New             |
| Arista       | TOR Switch     | 7060X5S                   | N/A               | New             |
| Arista       | mTOR Switch    | 7010TX                    | N/A               | Existing        |
| Arista       | Agg Chassis    | 7812R                     | N/A               | New             |
| Arista       | Agg Line Card  | 7800R3                    | N/A               | New             |

Notes:

- The Micron NVMe disks and the Intel 8474C processor do not have SMC specific parts.
- The NVIDIA H100 GPU does not have a SMC part, as it's required to be purchased as a L10 server
  and is included in the `SYS-821GE-TNHR` system.

<!-- cSpell: enable -- enables checking -->

Roughly this breaks down into two categories - server and network. The networking components that
are new are primarily used to drive the 400 Gigabit networking in this environment. The server
components that are new are the GPUs and the server itself.

We are leveraging vendors and baseline components in those vendor portfolios to expedite the time to
market.

No new third party software dependencies are expected in this SRB.

### 2.5.4 External and/or Circular

N/A

### 2.5.5 Brittle or Unusual

The system is very specific. No deviation from the parts will be allowed.

These systems will not be Secure Booted, as the workload that is run here requires PCIe tuning
within the VSI. The VSIs running here are single tenant, and therefore are not used in a concurrent
multi tenant fashion.

The team is evaluating how to include Secure Boot with GPUs in the future. However, that has
significant compatibility, complexity, stability and performance challenges in this environment.
There are also license restrictions as that may force a change to vGPU technologies. This is
considered out of scope for now.

The VSIs deployed on this infrastructure will also have the same limitations inherited from the
[SR-IOV SRB](../2191/README.md). Specifically, flow logs and live migration will not be supported.

The VSIs will not be confidential compute enabled - no SGX or TDX. A future SRB will evaluate
utilizing TDX with H100 GPUs, but that is follow up work. Those confidential computing capabilities
will be specified in the instance profile as `disabled` only.

### 2.5.6 Incompatibilities

These systems can not sit on a general population spine. Much like Vela, they need their dedicated
GPU agg switch to support the RDMA traffic across the nodes.

### 2.5.7 Databases

N/A

## 2.6 Integration, Deployment, and Operations Considerations

All of these servers will be managed with the existing tool set that we have operationally. This
includes:

- System Firmware Bringup Release bundle
- Governed Qualified Configurations
- HostOS / Kube Release Bundle deployment
- Integration with existing compute-proxy / agent to provision the devices and GPUs
- Existing monitoring / alerting will be utilized

### 2.6.1 Deployments

The following deployments will occur as part of the Vela2 project:

- Development
  - [Development / Integration / Staging](./deployments/development.md)
- Production
  - Cluster Deployments
    - [WDC07](./deployments/wdc07sr04.md)
    - [FRA04](./deployments/fra04sr04.md)
  - [Server only (inference)](./deployments/prod-inference.md)

## 2.7 Big Rules

### 2.7.1 Consistent Customer Experience

This infrastructure will only be offered on x86 VSI. It is a very specific workload type for a
client need. This is not a cross platform feature.

This offering will be limited to the following zones:

- `us-east-wdc07-a`
- `eu-de-fra04-a`

The `universal_name` must be used in public documentation that covers profile availability.
Documentation will include instructions for the user to look-up their account's zone mapping through
the public Geography API to confirm the logical name that will be used during provisioning. If a
customer requests to create a cluster network in a zone where it is unavailable, they will receive a
`400` HTTP `response` with an error message "The zone is not listed as supported on the specified
cluster network profile" as per this
[commit](https://github.ibm.com/genctl/regional-network-workspace/commit/72169f0812efa4da83443c6f917227035945c23f).

Access to the research specific profiles will be gated by an IBM Research only allow-list, as is
used on the IBM Vela solution. The customer facing H100 instance profiles are open to the market,
but are only available in the select regions / zones. The cluster network profile for the beta is
behind an allow-list.

#### 2.7.1.2 Allow-lists

- H100 instance profile
  - restrict access to the H100 profile to select customers
  - controlled and managed in global catalog
  - allow-list process: public, no longer behind an allow-list
- H100 instance profile (research only)
  - restrict access to the H100 research profile to IBM research
  - controlled and managed in global catalog
  - allow-list process: The GPU offering manager works with research to allow-list users to that
    account
- H100 cluster network profile
  - restrict access to the H100 cluster network profile to select customers
  - controlled and managed in global catalog
  - Allow-list process during beta: The GPU / Cluster Network offering managers work with the
    client to allow-list the accounts
  - GA: public, not behind an allow-list

### 2.7.2 VPC Big Rules

- [ ] All components must be able to be upgraded live (without service degradation/outage) and
      without manual intervention
  - No - firmware of the specific devices requires that the server be taken out of the pool. This is
    consistent with server firmware updates today.
- [x] All components must present usage metrics
- [x] All components must use the
      [VPC/NG monitoring framework](https://github.ibm.com/cloudlab/srb/tree/master/architecture/telemetry/monitoring)
- [x] All components must use existing VPC/NG control planes -- no new control planes
- [x] All components must be controlled via well-defined APIs
- [x] All components must provide functional parity across all modalities (UI, CLI, API, Terraform)
- [ ] All components must provide functional parity across all targeted platform architectures (x86
      and Z)
  - The only targeted architecture for this is x86 VSI.
- [x] All customer-facing GA features must interoperate with all other customer-facing GA features
  - Cluster features are explicitly cordoned off from certain SDN capabilities. The new resource
    introduced allows for a clean split between what is supported and isn't.
  - Flow logs will not be supported on the cluster subnets or vnics.
- [x] All customer-facing UIs and CLIs must use the customer-facing VPC API to operate on VPC/NG
      resources
- [x] All customer-facing APIs must adhere to the
      [API handbook guidelines](https://pages.github.ibm.com/CloudEngineering/api_handbook/)
- [x] All regional APIs must be the same in all regions (this implies equivalent functionality in
      all regions)
  - The cluster API will be globally available.
  - The cluster network profile will only be available in the two regions initially.
- [x] All components must perform authorization using the
      [VPC/NG IAM services](architecture/platformIntegration/genesis-iam)
- [x] All components must be internally stateless and recreate their runtime state from external
      sources
- [x] Any caching or shadowing of state must never result in an API consumer observing inconsistent
      or incorrect behavior
- [x] All hardware components must use the
      [VPC/NG diagnosability framework](https://github.ibm.com/cloudlab/srb/tree/master/architecture/telemetry/diagnostics)
      and implement the on-demand diagnostic API
- [x] All services must present an API for get/set/watch of state
- [x] All connections between components must be mutually authenticated, and encrypted using AES128
      or higher
- [x] All components must integrate into the existing operations tooling for deployment and
      maintenance

### 2.7.3 Cloud Security Baseline Requirements

- Authentication & access control: Enforce authentication and access control on all public
  interfaces and between service components.
  - Yes
- Secrets management: Store and manage secrets securely.
  - Yes
- Least privilege operation: Always operate at the lowest level of privilege required to execute
  tasks effectively. Don't run processes as root.
  - Yes
- Compute isolation: Isolate critical processes and untrusted code at the compute level.
  - Yes
- Network isolation: Group related systems into network segments that are isolated from unrelated
  systems, adding additional isolation for high risk systems.
  - Yes
- Encryption at rest & secure deletion: Encrypt all sensitive data at rest, all customer data is
  sensitive.
  - Yes
- Encryption in transit: Encrypt all sensitive data "on-the-wire", all customer data is sensitive.
  - Yes
- Encrypt to standards: Use AES-256 for symmetric encryption, use recommended TLS cipher suites,
  follow NIST 800-131A for anything else.
  - Yes
- Logging & non-repudiation: Log CRUD operations and forward all log entries off box to prevent
  tampering.
  - Yes
- Logging of credentials: Don't include any type of credentials in logs.
  - Yes
- Injection prevention: Prevent injection vulnerabilities by always validating user input, using
  language-specific safe libraries and APIs, and perform periodic application vulnerabilities scans.
  - Yes
- Secure development practices: Build security into all phases of the software development
  lifecycle.
  - Yes
- Update & secure configuration: Keep all aspects of the service updated and securely configured.
  - Yes

## 2.8 Assumptions, Risks, and Constraints

Supply chain constraints were an initial risk. That was mitigated in the plan, and is no longer a
concern.

The development racks will not utilize a new core network switch. The existing 100 GigE network will
be used for the development agg switches. 400 GigE will be deployed within the development racks,
but multi rack scaling of the 400 GigE network will not occur until production.

A minimal mitigation is that the staging racks will be cross connected to allow for 4-node full
bi-directional bandwidth. However the more interesting mesh scenarios will not be able to be
validated until production.

# 3. Architecture, Interfaces, and Impact

## 3.1 Architecture and Interfaces

This architecture introduces a new Instance Profile which has the following structure.

- 160 vCPUs
- 1796 GB DDR5 Memory
- 8 PLX Switches, each with
  - H100 GPU
  - 7.68 TB NVMe drive
  - 1 CX-7 NIC for traditional SDN networking
  - 8 CX-7 NICs for dedicated cluster network traffic

This design will be supported in many different sites:

| Location         | Design | Function                         | Scale                                          |
| ---------------- | ------ | -------------------------------- | ---------------------------------------------- |
| WDC07 SR03       | Vela   | Training (Research)              | 280 A100 Servers (2240 GPUs) + Private Cluster |
| WDC07 SR04       | Vela2  | Training (Research + Commercial) | 186 H100 Servers (1488 GPUs) + Cluster         |
| FRA04 SR04       | Vela2  | Training (Commercial)            | 27 H100 Servers (216 GPUs) + Cluster           |
| DAL              | Vela2  | Inference (Commercial)           | 7 H100 Servers (56 GPUs)                       |
| TOR              | Vela2  | Inference (Commercial)           | 7 H100 Servers (56 GPUs)                       |
| SAO              | Vela2  | Inference (Commercial)           | 7 H100 Servers (56 GPUs)                       |
| MAD              | Vela2  | Inference (Commercial)           | 7 H100 Servers (56 GPUs)                       |
| SYD              | Vela2  | Inference (Commercial)           | 7 H100 Servers (56 GPUs)                       |
| LON              | Vela2  | Inference (Commercial)           | 7 H100 Servers (56 GPUs)                       |
| TOK              | Vela2  | Inference (Commercial)           | 7 H100 Servers (56 GPUs)                       |
| _Design Maximum_ | Vela2  | Training                         | 750 H100 Servers (6,000 GPUs)                  |

The "+ Cluster" indicates that the deployment will include a dedicated private cluster network to
support the RDMA based traffic between the servers. This is an optional feature that customers will
need to opt into.

The "+ Private Cluster" indicates that the clustering support was hidden, and restricted through
allow-lists to only research. It is a label to indicate the previous pattern, which is now being
expanded into public facing cluster network support.

Note that the design maximum is the theoretical maximum size supported by this architecture. It is
not an actual build out. If future implementations are to be deployed, they can evaluate growing to
that size.

All standard customer facing functions will be available to these instances. Customers wishing to
use the the high speed connectivity will need to define a cluster network at the API level. Doing so
provisions a `cluster network`, which can then be used to attach `interfaces` via a
`cluster network attachment` to their instance. These interfaces allow for use the RDMA over
Converged Ethernet (RDMA) and GPU Direct (GDR) capabilities. This is only available in WDC07 and
FRA04 and in certain dev/stage m-zones.

The ability to create cluster networks will only be available in the two zones which initially
support it.

### 3.1.1 Architecture and Technical Design

#### 3.1.1.1 Workload Deployments

We are building this solution to provide a location in the IBM Cloud to train and inference high end
AI models. Some AI training may require a few GPUs, but the largest hundred billion parameter models
require thousands of homogenous GPUs.

To put in perspective the demands of these types of workloads, the IBM research team projected
training durations against a few different model sizes.

| Parameter Count | GPUs              | Number of Tokens | Training Duration |
| --------------- | ----------------- | ---------------- | ----------------- |
| 10B             | 256 A100 (Vela)   | 1.2 Trillion     | 43 days           |
| 100B            | 1024 H100 (Vela2) | 2 Trillion       | 44 days           |
| 200B            | 2048 H100 (Vela2) | 2 Trillion       | 44 days           |
| 500B            | 2500 H100 (Vela2) | 5 Trillion       | 232 days          |

These training runs take many weeks to complete (though are tolerant to nodes failing within the
run). Duration can be improved by improving the accelerator performance and by improving the number
of accelerators within a homogenous cluster network.

Training requires high end compute nodes and an ultra faster, low latency clustered network.
Inference requires a high end compute node server.

The Vela2 design is such that we will be able to deploy a physical cluster network as small as 11
racks (33 servers, 264 H100 GPUs) or as large as 276 racks (828 servers, 6624 GPUs).

#### 3.1.1.2 High Level Concepts for Vela2

The requirements of this SRB can not be fulfilled with the existing VPC design. A few new concepts
and capabilities need to be introduced into our overall design to meet the needs of these types of
workloads.

##### 3.1.1.2.1 Cluster Networks

Vela 1 was the first solution IBM provided which had a cluster network. In this implementation,
there was no clear separation between a customer facing `cluster network` and the
`physical cluster network`. They were in essence the same entity.

The Vela1 network supports RDMA traffic between servers in the cluster. This improves throughput and
dramatically reduces the latency of the network. Additionally it also improves the consistency of
the networks performance.

The restriction with Vela 1 is that it was an operational construct. Only research had access to
profiles with RDMA, and we knew that only servers that physically sat in a given room could
provision the special Instance Profile which supported RDMA. It was not a concept that could be used
outside a limited environment.

Vela2 will take the concept of a cluster, and empower our customers to utilize it. With this design,
the customer will be able to:

- Create a cluster network at the API level
  - This cluster network is now a customer facing abstraction, which will map back to a physical
    cluster network implementation on the back end (similar to VSIs landing on hosts).
- Create cluster subnets to the cluster network
- Create/Update interfaces within the cluster
- Attach interfaces to an instance through the `cluster_network_attachments`

The cluster network concept will be very similar to the concepts introduced by the VNI project. In
fact, behind the scenes, it will reuse many of the same resources/objects.

However, the cluster network will appear to the user as a new top level resource. This is because
this object does influence instance scheduling. Where a workload can be placed will be dependent on
how the user defined their cluster network. While the cluster network defines to the customer a
certain performance level, that performance is delivered in large part through instance placement.

Cluster networks will be able to support variation by having a `Cluster Network Profile`. This SRB
introduces a cluster network profile of `H100`. This is a specifically tuned cluster for the H100
GPU instance profile.

A general population user will see the cluster APIs, but querying for the cluster network profiles
will return an empty list in all regions except the two which support cluster networks.

Physically, host hypervisor servers will be tagged as residing within a physical instance of a
cluster. The customers cluster network will map to a specific instance of a physical cluster within
the zone. That will ensure that workloads which need the network capabilities land within the same
physical cluster network.

The cardinality between the customers cluster networks and physical cluster implementations is many
to one. A customers cluster network maps to exactly one physical cluster implementation. But a
physical cluster implementation supports many customer's cluster networks.

In theory a physical cluster implementation can support more customer facing cluster networks than
servers it has. This is because a cluster network has it's own life cycle, it can be provisioned and
then never have any infrastructure (instances) deployed within it. Provisioning a cluster network
simply creates a link to a physical cluster implementation to help future scheduling.

Cluster networks scheduling will be very primitive for this proposal. Meaning that the scheduling
will be direct to a single physical cluster implementation of a matching type. The logic will in
fact use just the _first_ physical cluster implementation of that type it finds. When we have two
physical cluster implementations of the same type proposed for a zone, a new SRB will be introduced
to provide:

- Cluster Scheduling
- Operator controls to improve management across multiple physical cluster implementations

The key for this proposal is to ensure the data model supports that future, but not to miss the go
to market time frame.

If a workload does not specify any cluster network attachments, but the hypervisor hosts for the
instance profile reside within a physical cluster implementation - it is still a candidate to land
that instance on that host hypervisor. However the workload will be isolated to just standard cloud
networking.

##### 3.1.1.2.2 Physical Mapping of Network in Node

For the H100 solution, the cluster networks defined by the customer will deploy against a set of
physically separate NICs in the server.

![Vela2 Compute Node Traffic Layout](./assets/Vela2ComputeLayout.png)

Specifically, the standard `network_interfaces` will be deployed to the NIC at the top of the
diagram. That NIC is physically direct connected to the CPU. The NICs on the sides of the server
will run the cluster network traffic.

Future server designs that integrate cluster network support may unify the cluster network
interfaces on top of the same physical NIC as the standard cloud network interfaces. There are more
future scenarios discussed in the [bandwidth document](./hld/bandwidth-pooling.md).

##### 3.1.1.2.3 Cluster Network to Physical Cluster Implementation Mapping

In this proposal, the customer will be able to create a `Cluster Network`.

When the user creates a cluster network, it will need to be scheduled to a physical cluster
implementation within the zone. This cluster network will have a profile (ex. `H100`) which
indicates that it can only run atop a physical cluster implementation with the same profile name.
The user will not see the mapping to the physical cluster implementation, but it will exist behind
the scenes.

For this first implementation, there is only a single physical cluster implementation being
delivered (for which multiple logical customer facing clusters can connect to) - in two zones. But
this design sets the foundation data model for having multiple clusters of the same type in a given
region. When we have a request to support two or more clusters of the same type in a region, a new
SRB will be created to introduce cluster scheduling.

Conceptually though, when two physical cluster implementations of the same type are introduced into
the same zone, the data model will look like the following.

![Virtual to Physical Mapping](./assets/VirtualToPhysicalMapping.png)

In the figure, it should be pointed out that IBM Cloud `Account: C` has two distinct
`cluster network` objects. Each cluster network object must map to exactly one
`physical cluster implementation`. But an account can have multiple cluster networks. Those cluster
networks may map to the same physical cluster implementation, or different ones.

The current design also allows for multiple clusters of the same type. Ex. if we have an H100 and
B100 (future GPU) cluster in the same zone, the virtual cluster definition should map to the right
physical cluster in the back end.

##### 3.1.1.2.4 Limited Zonal Availability

Less a concept, but more of a constraint. Not all zones will have access to the H100 hardware or
their corresponding physical cluster implementations. In fact, due to their high cost they will only
be in a single zone in some regions.

There is [Aha VSIVPC-690](https://bigblue.aha.io/features/VSIVPC-690) being evaluated for future
offerings.

Offering decided that the H100 instance profile will be available to external customers. However,
the system is so large that quota bumps are required to be able to utilize it (instance storage
being the limiter). So customers that provision it will need to work with IBM support to get a quota
increase.

##### 3.1.1.2.5 Zonelet Integration

The H100 servers within a physical cluster implementation can be run from within the main zone or a
zonelet. It is an operational decision whether or not the servers in a physical cluster
implementation are run from a separate zonelet or not.

The sites which deploy cluster networks will be deployed as zonelets. This means that WDC and FRA
clusters will be deployed as a zonelet. However, the inference regions will be deployed within the
main zone.

#### 3.1.1.3 Physical Design

#### 3.1.1.3.1 Server Node Design

The Vela2 physical compute server will be a significant leap ahead of the Vela compute node - both
at an accelerator and a system level.

The accelerator will be the `H100 SXM 80 GB` and the server will be the `SYS-821GE-TNHR` which is a
partner design implementation of the [DGX H100](https://www.nvidia.com/en-us/data-center/dgx-h100/).

| Performance                | A100 SXM4 80 GB | H100 SXM5 80 GB | Improvement |
| -------------------------- | --------------- | --------------- | ----------- |
| FP64 (per GPU)             | 9.7 TFLOPS      | 34 TFLOPS       | 250% faster |
| FP32 (per GPU)             | 19.5 TFLOPS     | 67 TFLOPS       | 244% faster |
| TF32 (per GPU)             | 312 TFLOPS \*   | 989 TFLOPS \*   | 217% faster |
| FP16 Tensor Core (per GPU) | 624 TFLOPS \*   | 1979 TFLOPS \*  | 217% faster |
| BF16 Tensor Core (per GPU) | 624 TFLOPS \*   | 1979 TFLOPS \*  | 217% faster |
| INT8 Tensor Core (per GPU) | 1248 TFLOPS \*  | 3958 TFLOPS \*  | 217% faster |
| GPU Memory                 | 80 GB           | 80 GB           | ---         |
| GPU Onboard Memory Speed   | 2,039 GB/s      | 3,350 GB/s      | 64% faster  |

The system level performance improves dramatically as well.

|                          | Vela                                 | Vela2                          | Improvement     |
| ------------------------ | ------------------------------------ | ------------------------------ | --------------- |
| CPU                      | 2 x 24 core Cascade/Ice lake         | 2 x 48 core Sapphire Rapids    | 100% more cores |
| Customer vCPUs           | 80                                   | 160                            | 100% more vCPUs |
| Customer Memory          | 1280 GB                              | 1792 GB                        | 40% more memory |
| Memory Speed             | 280 GB/s (two sockets)               | ~750 GB/s                      | 168% faster     |
| GPUs per Server          | 8                                    | 8                              | ---             |
| CPU to GPU Connectivity  | SXM3 over PCIe Gen 3                 | SXM5 over PCIe Gen 5           | 300% faster     |
| GPU to GPU Connectivity  | NVLink3 (600 GB/s)                   | NVLink4 (900 GB/s)             | 50% faster      |
| Network Throughput       | 200 Gbps (400 Gbps w/ GDR)           | 3200 Gbps GPU / 400 Gbps Cloud | 1500% faster    |
| Network Capabilities     | SR-IOV, RDMA, GPUDirect              | SR-IOV, RDMA, GPUDirect        | ---             |
| PCI Switch Clustering    | 2 GPUs, 2 Disks, NIC per PCIe Switch | 1 GPU, 1 Disk, 1 NIC           |                 |
| PCIe Switches per Server | 4                                    | 8                              | ---             |
| DGX Compatible           | Yes (Partner Design)                 | Yes (Partner Design)           | ---             |

Additionally this server has 9 NICs - 1 dedicated for standard cloud networking. The are 8x CX-7's
cards for the dedicated cluster networking. The H100 server is part of a class of servers which have
dedicated NICs for cluster networking. Other servers may run their cluster network over the same
physical NIC as the standard cloud networking. The traffic over the 8x CX-7's is meant to support
GPU to GPU traffic only.

![Vela2 Compute Node Traffic Layout](./assets/Vela2ComputeLayout.png)

As noted here, the top CX-7 is the NIC dedicated to standard cloud networking. It attaches to the
CPU. The CX-7's on the left and right are the dedicated cluster network NICs, and are attached to
the PLX switches in the HGX board. Only GPU to GPU traffic flow through those cluster network NICs.

The VCN will only run atop the standard cloud networking NIC. All other cloud network functions
(storage, File as a Service, data path) will only run on that NIC. The only traffic to flow over the
cluster_network is for the GPU to GPU traffic. Which is why the VCN is isolated off of the standard
cloud networking NIC.

Power Break will be configured on the H100 training servers as well. If a line cord or power supply
fails, the systems must scale down to 6 kW of power.

##### 3.1.1.3.2 Rack Design

The Vela2 rack will have:

- (1x) Arista DCS-7010TX-48-R: Provides the management networking
- (2x) Arista DCS-7060DX5-64S-R: Provides the 400 GigE networking

Where a physical cluster implementation is deployed, Vela2 will support three servers per rack. It
will use the power cap technology to hit this density. For inference only deployments, where no
physical cluster implementation is delivered, only two servers per rack will be supported.

This key difference is needed because we will not over commit the power grid dramatically for
inference locations. Where physical cluster implementations are deployed, the data center providers
are collaborating with IBM to ensure we can achieve the density of power we need.

#### 3.1.1.3.3 Physical Network Architecture

The network architecture for Vela2 physical cluster implementations follows IBM Cloud's Agg/ToR
standard topology. The Vela2 generation is an iterative design atop what was deployed with Vela. The
goal of the Vela2 physical cluster network is to be close to a 1:1 over commit ratio between the
ToRs and the Aggregation switches.

The physical cluster implementation aggregation switches will connect to the broader WDC07 VPC
network through the super spines. The GPU traffic will only ever flow within the Training Cluster
Network. The network traffic is treated as RDMA only within the dedicated spine network.
Additionally all RDMA traffic will be set to drop between the spine and the super spine.

This breaks down to each rack having:

- 1.2 Tbps of standard cloud network traffic (3x servers, 2x200G per server)
- 9.6 Tbps of cluster network traffic (3x servers per rack, 8x 400G per server)
  ([See Single vs. Dual Port](#524-single-port-nics-versus-dual-port-nics))

![Vela2 Network Design](./assets/Vela2NetworkTopology.png)

The network will have a 1.125 : 1 over subscription. If looking at raw cluster network traffic,
there is no over subscription, but the standard cloud network traffic adds a little load atop the
backing switches.

This network design is also scalable. Each ToR needs 12x 400G uplinks. Therefore the Agg layer can
scale from 4, 6, 8 or 12 aggregation switches depending on the deployment. The number of line cards
within the chassis is also scalable. This affords a very modular and scalable design. As an example:

| Design Point | Maximum Size          | Chassis Switch | Number of Spine Switches |
| ------------ | --------------------- | -------------- | ------------------------ |
| Small        | 22 racks (528 GPUs)   | Arista 7808R3  | 4                        |
| Typical      | 138 racks (3312 GPUs) | Arista 7812R3  | 6                        |
| Extreme      | 276 racks (6624 GPUs) | Arista 7816R3  | 12                       |

However, see the various deployments. The largest initial deployment will be in WDC07, which will be
about 70 racks of GPU compute (210 servers / 1680 GPUs).

The ToR switches in clustered deployments will have Arista Dynamic Load Balancing (DLB) enabled.

The deployments are listed [here](#261-deployments), which explains each deployments physical
infrastructure needs.

Note that the Inference Only deployments have the same TOR switches in rack, however there will not
be a Training Cluster Network deployed.

##### 3.1.1.3.3.1 VRF and Logical Rails

As part of the performance evaluation of the cluster, significant issues were uncovered in the AI
network. The issue is documented in the picture below. Essentially, due to queue pair imbalances and
ECMP hashes, certain workloads were running very poorly.

![understanding-collisions](./assets/understanding-collisions.png)

The Network Architecture team, Research and Arista developed a VRF based solution which created
logical rails in the aggregation switches. The number of VRF rails may be dependent on the number of
uplinks from TOR switches to the aggregation switches. While 4 'rails' are defined in WDC, in FRA 6
'rails' are defined. This ensured that the aggregation switches sent the traffic to a deterministic
destination port. ECMP hashing was effectively removed from the agg layer and beyond (for AI
traffic). This is a workaround until CLB is available on the aggregation layer. Once it is available,
we will test and compare the results.

![redundant-logical-rails](./assets/redundant-logical-rails.png)

ECMP hashing still occurs on the source ToR. The solution utilizes Arista's Dynamic Load Balancing
(DLB) to rehash on the source ToR if a given link is congested. There are still optimizations to be
made here, as the DLB polling rate is not always fast enough. There are other considerations still
being evaluated, but the primary enhancements are expected to be on the CX-7 side.

Performance assessment of CX-7 ZTR-RTTCC (zero Touch RoCE - Round Trip Time Congestion Control) algorithm
is still ongoing as a joint effort by the research and the performance team. If the work completes post-GA,
the corresponding tunings will be applied post-GA.

#### 3.1.1.3.4 Network Latency

The switches have the following latency:

| Switch     | Purpose             | Latency (64 byte packet) |
| ---------- | ------------------- | ------------------------ |
| 7060DX5-64 | GPU Top of Rack     | 825 ns typical           |
| 7816R3     | Typical/Extreme Agg | 4 μs typical             |
| 7804R3     | Small Agg           | 4 μs typical             |

Additionally there is minimal wire line / transceiver latency added.

Cross rack communication will be three switch hops - ToR -> Agg -> ToR.

#### 3.1.1.4 Overall Cluster Design

There are two clusters to consider:

- Physical Cluster Implementation: This is a specific implementation of a cluster in the data
  center. A physical cluster has both a type and a name.
- Cluster Network: This is virtual cluster that the user creates at the API. It is created based off
  a Cluster Network Profile, and internally (hidden from the user) will map to a compatible Physical
  Cluster Implementation.

The work flow is the following:

- Physical Cluster Implementation:

  - A cluster definition is added to the `mzone.yaml` file.
  - Upon the next expansion, kube-define will be updated to create the appropriate objects to
    represent the cluster objects on the backend.
  - Release engineering determines when / where to define clusters.
  - Individual server nodes will be added to the cluster through additional modifications to the
    server spec in the mzone.yaml.
  - See [m-zone yaml updates](#31151-m-zone-yaml-updates) for more information on this flow.

- Customer Flow:
  - Cluster Network
    - The customer will enumerate the `cluster network profiles` available for a given zone.
    - Once they choose a `cluster network profile`, they will create a `cluster network`.
      - Behind the scenes, the cluster network resource created by the user should then be mapped to
        a specific `physical cluster implementation`.
      - This is used to inform the VM scheduler candidate hypervisors when an Instance is
        provisioned which connects to the `cluster network`.
    - The user will then create `subnets` as child objects on the `cluster network`.
    - The user may pre-create `interface` objects on the `cluster network`, or they may do this as
      part of instance provision.
  - Instance Provision
    - Customer will identify a cluster network capable instance profile (specifically in this case
      the `gx3d-160x1792x8h100` profile).
    - Customer will set the parameters on the instance creation that are standard (ex. image
      selection, volume definition, standard cloud networking, advanced configuration like metadata,
      etc...)
    - Customer will select a set of `subnets` or `interfaces` to attach to the `cluster network`.
      These are the `cluster network attachments`
      - The number of cluster network attachments are defined by the Instance Profile.
      - The order of the cluster network attachments defines the layout on the backend system
    - The instances will be created
      - If there is no capacity in the physical cluster implementation to support the instance, the
        instance will move to a `failed` state and the `status_reason` for the instance will
        indicate that there is a lack of capacity.
      - If there is capacity, the instance resource will be created and then available to the user.

See the [API Definition](#313-customer-facing-apis) for more details on the customer facing design.

##### 3.1.1.4.1 Customer Pricing

Clusters will have no charge. The charges are rolled into the instance costs.

##### 3.1.1.4.2 Global Catalog and Cluster Network Profile Names

Cluster network profiles will be defined in the Global Catalog, and the first cluster network will include
the following information:

```yaml
- name: H100
  type: h100
  family: vela
  display_name: H100 GPU Based AI Training Cluster
  zones: [us-east-3, eu-de-3]
```

The cluster network profile name will be H100 and the family will be vela at GA, however the intention
is to establish a naming taxonomy for cluster network profile names and families in preparation for
future cluster network profiles as new hardware is introduced that requires a different physical cluster
network implementation. The H100 profile will be deprecated, and replaced by a new cluster network profile
that follows these conventions. The deprecation will not require any existing cluster networks to be
recreated.

The cluster profiles will need to be added to the
[vpc-bss-onboarding](https://github.ibm.com/cloudlab/vpc-bss-onboarding). However there will be no
billing for this resource.

[BSS Onboarded Production Cluster Profiles](https://github.ibm.com/cloudlab/vpc-bss-onboarding/tree/master/resources/is.cluster-network/product_schema/prod)

##### 3.1.1.4.3 Cluster API Design

See the [Customer Facing API section](#313-customer-facing-apis) for details. The only expected API
changes for this project are due to the cluster design.

##### 3.1.1.4.4 Creation / Scheduling

There is a restriction in the requirements that there will be no more than one physical cluster
implementation of a given type within a zone at a time. In the future, as part of a subsequent SRB,
our roadmap is requiring that we will need to support multiple physical cluster implementations
within a single zone. But that is out of scope for this initial SRB.

Given that, this SRB will lay the initial foundation items to support mapping a customers
`cluster network` to a backend `physical cluster implementation`. However certain areas of the logic
will be enhanced as part of that subsequent SRB.

When the customer creates a cluster network via the API, the RNOS component will create a
`ClusterNetworkAllocation`. That projects down to zonal which creates a `ClusterAllocation`
object. A new zonal compute controller watches for this object and effectively binds that allocation
to a specific physical cluster implementation of that type. Given the restriction noted above, the
scheduling of the cluster network to a physical cluster is trivial.

**Note: The cluster network does not guarantee VSI capacity within the cluster.**

If the user adds cluster network attachments to their VSI, those cluster network attachments are
known by compute and fed into the VSI scheduler. The VSI scheduler will then filter out hosts that
are not within the physical cluster implementation that the cluster network is attached to.

All cluster network attachments must map to a single cluster type as well, which is enforced by RCOS
when the VSI is created/updated.

If there is not capacity in the cluster, it will be represented as an out of capacity message when
the VSI is scheduled. This may be re-evaluated in a future SRB where multiple physical cluster
support is introduced.

The zones within WDC and FRA that have H100 hosts are the only ones deploying a cluster. However,
all of the H100 hosts within those zones will be connected to the same backend cluster. No hosts in
those zones should be outside the cluster. As we increase the quantity of clusters, this experience
will improve (which will be outlined in subsequent SRBs). Additionally, at some future point, we may
enhance this to allow H100 servers to exist in those same zones, but sit outside the cluster. Both
of these are outside the scope of this SRB however.

More details of the High Level Design can be found
[here](https://confluence.swg.usma.ibm.com:8445/display/CTL/Cluster+Scheduling).

#### 3.1.1.5 Deployment Architecture

It is a pre-requisite to read the
[High Level Concepts for Vela2](#3112-high-level-concepts-for-vela2) prior to reading this section.
This is because identification of the physical clusters, where the VCN is configured, etc... is
paramount to this section.

##### 3.1.1.5.1 M-Zone Yaml Updates

In order to configure the hosts correctly, several updates are needed to the mzone yaml to pass into
HostOS/Kube Define.

The mzone yaml is the compiled element that informs each of these tools the configuration for the
node. The update to the section will be the following.

```yaml
- hostname: dal1-qz2-sr1-rk277-s12
  arch: amd64
  hostIP: 192.168.21.25
  fabricIP: 10.22.19.251
  FPextIP: 172.31.186.93
  MAC: "7C:C2:55:79:59:80"
  FPMAC: "94:6D:AE:7F:05:6A" # Only one FP Mac on these servers, as there is only one cloud nic
  cluster_map: # New Field: Indicates which NIC and which cluster it fits in.
    - mac: 94:6D:AE:D3:84:BE
      ip: 172.31.186.25
      cluster: WDC07-SR05-H100
    - ...
  ipmi: 192.168.205.60
  uuid: 00000000-0000-0000-0000-3cecef36e836
  profileClass: gx3d-h100 # UPDATED FIELD: Represents the instance profile class this supports
  hwClass: gx3d-h100-smc # NEW FIELD: indicates the specific PCIe topology for downstream services
  uefi: true
  compute: true
  master: false
  control: false
  gpu: true
  service: false
  localdisk: true
  edge: false
  solid_state_device: # UPDATED FIELD: 8 of these for the disks.  No structural changes though.
    - psid: 80M4Z9GVQ53J6MTZPQQ5W9A7N62TZ6SL
      serialnumber: S6EVNA0TB02685
    - ...
```

The new / updated attributes will also need to be reflected in the sysman etcd.

Additionally, the `profileClass` for the `gx3d-160x1792x8h100-research` profile will be
`gx3d-research`. The `hwClass` will stay consistent as the `gx3d-h100-smc` as that is consumed
primarily by SDN and others for the PCIe static topology which doesn't change across the systems.
The new profile class ensures that research deploys to research allotted servers per the
[requirements](#23-requirements-and-use-cases).

In the Vela2 design, the `CLUSTER_MAP` section will not have any macs which overlap with the
`FPMAC`. However, in future generations, the CLUSTER_MAP could contain NICs which are part of the
cluster. It should not be assumed that they're fully independent. Rather the Vela2 specification has
them independent.

Additionally, the clusters need to be defined in the mzone.yaml itself. This will be a new section
in the mzone.yaml definition. It will look like the following:

```yaml
clusters:
  - name: WDC07-SR05-H100
    type: H100
  - name: WDC07-SR04-A100 # Example only, to show different type, and multiplicity
    type: A100
  - name: WDC07-SR06-H100 # Example only, to show multiplicity of the same type
    type: H100
```

Note: A100 won't actually be defined as a cluster. It is simply shown here to demonstrate where
multiplicity could exist. However all future generations should be specified as proper clusters.

##### 3.1.1.5.2 Deployment File Updates

Ultimately, the mzone.yaml is simply a rendered artifact from several different sources. While
checked into platform-inventory, it really is primarily a compiled artifact.

Two new pieces of information need to be fed into it.

[Deployment Types](https://github.ibm.com/cloudlab/platform-inventory/tree/master/region/deployments/types)
and there specific
[instances](https://github.ibm.com/cloudlab/platform-inventory/tree/master/region/deployments/zones)
identify all of the various deployments within the cloud. The deployments for which clusters reside
in will be updated to indicate the clusters they host.

For example, mzone777 will be updated in the deployments with the following:

```yaml
- name: mzone777
  type: vpc_mzone
  stage: production
  owners: *id001
  notes: production m-zone for wdc3-qz1
  hostname_prefixes:
    - wdc3-qz1
  metadata: # NEW FIELD
    clusters:
      - name: WDC07-SR05-H100
        type: H100
      - name: WDC07-SR04-A100 # Example only, to show different type, and multiplicity
        type: A100
      - name: WDC07-SR06-H100 # Example only, to show multiplicity of the same type
        type: H100
```

Note: A100 won't actually be defined as a cluster. It is simply shown here to demonstrate where
multiplicity could exist. However all future generations should be specified as proper clusters.

The new metadata section will be cross referenced with the mzone.yaml checked in as part of the CI.

Additionally, the
[server allocation](https://github.ibm.com/cloudlab/platform-inventory/tree/master/region/allocations)
will be updated with the following to indicate the cluster it is a part of.

```yaml
- hostname: wdc3-qz1-sr4-rk001-s04
  inventory_state: production
  workflow: none
  allocation: mzone777
  class: gx3d-h100-smc
  config_version: 1.0.0.0
  role: vpc_compute # will be vpc_compute_research if for research
  cluster: WDC07-SR05-H100 # NEW FIELD
```

This additional metadata will also be used to update the CI. The CI will cross reference the
mzone.yaml and ensure that the node is properly ear-marked for the right cluster.

##### 3.1.1.5.3 Architecture File Updates

The architecture files that are generated by the platform inventory CI will also be updated.

```yaml
- hostname: wdc3-qz1-sr4-rk001-s04
  nicCount: 9 # Updated
  profileClass: gx3d # Updated
  hwClass: gx3d-h100-smc # NEW FIELD
  cluster: WDC07-SR05-H100 # NEW FIELD
  uefi: true
```

This is a compiled artifact from the
[Architecture File Generator](https://github.ibm.com/cloudlab/platform-inventory/blob/master/tools/generate_arch_files.py).
The compiled artifacts are sent to Box, and used by the fleet team to determine the parameters they
need to generate expansion files. These new fields provide the necessary data to appropriately
compile the expansion files. The format for those matches the
[mzone yaml](#31151-m-zone-yaml-updates).

The fleet tools will need to be updated to reflect this new design.

###### 3.1.1.5.4 System Specification Design

There will be several new specifications built:

- [Server Specification](https://github.ibm.com/cloudlab/platform-inventory/blob/master/specifications/solutions/vsi-hosts/gx3d-h100-smc.yaml)
- [Rack Specification](https://github.ibm.com/cloudlab/platform-inventory/blob/master/specifications/racks/compute_h100_s12_s42_7010tx-48_pdu_down.yaml)
- Components
  - A new
    [Mellanox](https://github.ibm.com/cloudlab/platform-inventory/blob/master/specifications/components/mellanox.yaml)
    NIC will be added to the components
  - A new
    [NVIDIA](https://github.ibm.com/cloudlab/platform-inventory/blob/master/specifications/components/nvidia.yaml)
    GPU will be added to the components
  - A new
    [Supermicro Server Chassis](https://github.ibm.com/cloudlab/platform-inventory/blob/master/specifications/components/supermicro.yaml)
    will be added to the components
  - The existing
    [Micron 7450 NVMe Drives](https://github.ibm.com/cloudlab/platform-inventory/blob/master/specifications/components/micron.yaml)
    will be referenced by the server specification

##### 3.1.1.5.5 HostOS Updates

There are a few updates expected for HostOS. However it is expected that this solution run atop
HostOS 6.x (not a future HostOS 7.x).

1. Moving to the CX-7 NICs may require a new release of the HostOS boot bundle
1. Add to the /etc/network.conf file:
   1. Defines the NICs on the system for 9 NICs (does not need to identify specific NICs for
      purposes, as NIC scheduling is a compute role).
   1. Adds to ensure that the `hwClass` is embedded (if not present, do not add anything)
1. HostOS will require additional firewall changes (iptables and ipsets) to support the additional
   east/west NICs.

##### 3.1.1.5.6 Kube Define (GravingYard)

Kube define (Graving Yard) will annotate a few objects:

- The `genctl.nics` annotation on the hypervisor node will be updated to indicate it participates in
  a physical cluster implementation (ex. the 8 GPU nics will be part of the cluster, but the cloud
  NIC will not in the H100 case).
- The node object will be annotated to include the type of cluster (ex. H100) and the name of the
  cluster it participates in.

These updates will retrieve their data through the
[mzone server definition updates](#31151-m-zone-yaml-updates).

#### 3.1.1.6 SDN Architecture

The initial SDN architecture extends the Vela design - which is the
[SR-IOV enablement](../2191/README.md). Virtual Functions will be connected into the VSI which
provide CX-7 ports directly into the VSI.

The Vela2 architecture for SDN though breaks into a few separate features.

The high level view of how the network will be set up is that the north/south will run the VCN,
storage and all cloud networking functions. The east/west will not run the VCN or any other priority
traffic. Instead they will run just the GPU training traffic.

At a high level, the layout looks like the following.

![Network Design of the TEPs](./assets/NetworkDesignTEPs.png)

SDN will be allotted 2 GB of memory per physical NIC for huge pages. SDN will also be alloted 1 full
physical core per NIC for DPDK processing (18 vCPUs total).

##### 3.1.1.6.1 Network Configuration Updates

The network.conf will be updated to include new key pieces of information

Key pieces of information that are needed by SDN:

- Existing, but being extended
  - `TEPX_IP`: The Tunnel Endpoint Address for the X NIC (ex. TEP0_IP). There will be 9 TEP_IP's for
    this system type.
  - `TEPX_PY_IP`: The TEP IP for a given port
  - `TEPX_PY_PCI_ADDR`: The PCI address for a given port for the NIC.
  - `PROFILE_CLASS`: Will be updated to include the physical system profile.
- New Fields
  - `TEPX_ROCE_VERSION`: Indicates if roce is enabled. Values are `rocev2` or `none`
  - `TEPX_GDR_COMM`: Indicates if GPU Direct is enabled on this TEP. Values are `gdr` or `none`
  - `HWCLASS`: This will map to the profile class.

Here is a sample output for the configuration update

```shell
NODE_TYPE=Compute
NODE_TYPES=master:false,compute:true,control:false,service:false,observability:false,acadia:false,mgmt:false,mcdvm:false,edge:false,nfs:false,baremetalproxy:false,baremetal:false
PROFILE_CLASS=gx3d-h100
HWCLASS=gx3d-h100-smc
VERSION=1.0.0
EDGE=False
EDGE_IP=10.12.60.5
FABRIC_IP=10.22.65.226
TEP0_IP=172.29.112.92
TEP0_P0_IP=172.31.112.93
TEP0_P0_PCI_ADDR=0000:53:00.0
TEP0_P1_IP=172.31.112.221
TEP0_P1_PCI_ADDR=0000:53:00.1
TEP0_ROCE_VERSION=none
TEP0_GDR_COMM=none

TEP1_IP=172.29.112.24
TEP1_P2_IP=172.31.112.25
TEP1_P2_PCI_ADDR=0000:da:00.0
TEP1_P3_IP=172.31.112.153
TEP1_P3_PCI_ADDR=0000:da:00.1
TEP1_ROCE_VERSION=rocev2
TEP1_GDR_COMM=gdr

TEP2_IP=172.29.112.22
TEP2_P4_IP=172.31.112.23
TEP2_P4_PCI_ADDR=0000:bb:00.0
TEP2_P5_IP=172.31.112.151
TEP2_P5_PCI_ADDR=0000:bb:00.1
TEP2_ROCE_VERSION=rocev2
TEP2_GDR_COMM=gdr

TEP3_IP=172.29.112.20
TEP3_P6_IP=172.31.112.21
TEP3_P6_PCI_ADDR=0000:aa:00.0
TEP3_P7_IP=172.31.112.149
TEP3_P7_PCI_ADDR=0000:aa:00.1
TEP3_ROCE_VERSION=rocev2
TEP3_GDR_COMM=gdr

TEP4_IP=172.29.112.18
TEP4_P8_IP=172.31.112.19
TEP4_P8_PCI_ADDR=0000:9b:00.0
TEP4_P9_IP=172.31.112.147
TEP4_P9_PCI_ADDR=0000:9b:00.1
TEP4_ROCE_VERSION=rocev2
TEP4_GDR_COMM=gdr

TEP5_IP=172.29.112.16
TEP5_P10_IP=172.31.112.17
TEP5_P10_PCI_ADDR=0000:5c:00.0
TEP5_P11_IP=172.31.112.145
TEP5_P11_PCI_ADDR=0000:5c:00.1
TEP5_ROCE_VERSION=rocev2
TEP5_GDR_COMM=gdr

TEP6_IP=172.29.112.14
TEP6_P12_IP=172.31.112.15
TEP6_P12_PCI_ADDR=0000:3b:00.0
TEP6_P13_IP=172.31.112.143
TEP6_P13_PCI_ADDR=0000:3b:00.1
TEP6_ROCE_VERSION=rocev2
TEP6_GDR_COMM=gdr

TEP7_IP=172.29.112.12
TEP7_P14_IP=172.31.112.13
TEP7_P14_PCI_ADDR=0000:29:00.0
TEP7_P15_IP=172.31.112.141
TEP7_P15_PCI_ADDR=0000:29:00.1
TEP7_ROCE_VERSION=rocev2
TEP7_GDR_COMM=gdr

TEP8_IP=172.29.112.10
TEP8_P16_IP=172.31.112.11
TEP8_P16_PCI_ADDR=0000:19:00.0
TEP8_P17_IP=172.31.112.139
TEP8_P17_PCI_ADDR=0000:19:00.1
TEP8_ROCE_VERSION=rocev2
TEP8_GDR_COMM=gdr

KUBE_APISERVER_IP=10.51.41.10
VCN_PGW_CIDR=52.118.228.128
VCN_SGW_CIDR=10.22.128.0
PEER_IPS=10.0.14.101,10.0.14.102
LOCAL_BGP_ASN=4203065544
PEER_BGP_ASN=36351
BGP_COMMUNITY_ORIGIN_LOCAL_CUSTOMER=36351:4450
BGP_COMMUNITY_ORIGIN_BACKEND_LOCAL_CUSTOMER=13884:4450
BGP_COMMUNITY_ORIGIN_GENESIS=65201:4
BGP_COMMUNITY_ORIGIN_VPC=65201:6
BGP_INTERNET_RD_RT=36351:1015
BGP_GENESIS_SERVICE_RD_RT=36351:268540309
BGP_INFRASTRUCTURE_SERVICE_RD_RT=36351:1001
BGP_CLOUD_SERVICE_ENDPOINT_RD_RT=36351:268521259
INFRASTRUCTURE_SERVICE_NETWORK_CIDRS=161.26.0.0/16
CLOUD_SERVICE_ENDPOINT_NETWORK_CIDRS=166.8.0.0/14
VCN_PUBLIC_GW_IP=52.118.228.128
VCN_SERVICE_GW_IPS=10.22.128.0
VCN_SERVICE_GW_CIDRS=161.26.0.0/16,166.8.0.0/14
GEN_TYPE=3
smartnic=null
OBS_IP_CIDR=10.22.66.0/24
```

##### 3.1.1.6.2 Configuration of the VCN on Vela2

The VCN of Vela2 will be set up only on the standard cloud NIC (CPU-connected) on the compute host.
See the [Vela2 Compute Node](#3112-high-level-concepts-for-vela2) for a primer on the NIC types. The
eight dedicated cluster network NICs will only have the TEPs configured.

Only the standard cloud NIC will be configured to have the VCN/loopback run on top of it. The rest
of the cluster network NICs will have TEPs running on them only.

##### 3.1.1.6.3 Configuration of vNICs for VMs

There will be two types of physical NICs within the system. Eight Cluster Network NICs and a single
Standard Cloud NIC. All of the physical NICs will be a CX-7. The definition of whether or not a vNIC
(or VNI attachment) is attached as a cluster network interface or standard cloud interface is
defined through the interface specification.

Compute will send to fabcon the reference for which TEP to schedule the NIC on. It will determine
which TEP to provision against based off the instance spec, the associated cluster network
reference, and the information from the corresponding Kubernetes compute node definition of the NICs
and their roles.

RoCE enablement was done via [SRB 3856](https://github.ibm.com/cloudlab/srb/issues/3856). However it
needs to be updated to support only being configured on a subset of NICs.

Due to the single tenant nature of the VSIs, the Cluster network interfaces will have unbounded
bandwidth on the H100 implementation. It has been determined that setting any rate limit at such
high speeds results in a significant reduction in performance - well below the rate limited amount.
This is not a concern for a single tenant solution, especially when the NIC is essentially sold
directly to the customer.

The Cloud networks will be virtio NICs atop the standard cloud CX-7 NIC. Additionally all storage,
COS, File, etc... traffic will flow over the standard cloud nic.

As a reminder, flow logs, security groups, NACLs, Floating IPs, LBaaS connectivity, public gateways,
etc... will function wholly atop the standard cloud NIC.

#### 3.1.1.7 Compute Design

##### 3.1.1.7.1 Hypervisor

###### 3.1.1.7.1.1 Hypervisor Overhead

We will set aside additional overhead for these node types. More memory/cores is required for the 8
NICs. Additionally, optimizations that are coming in the future will likely require additional
memory / core overhead.

As such, threadbare / kube-define will be updated to set the hypervisor overhead the following way.

| Component | System    | Customer  | Overhead                            |
| --------- | --------- | --------- | ----------------------------------- |
| Memory    | 2048 GB   | 1796      | 256 GB                              |
| CPU       | 192 vCPUs | 160 vCPUs | 32 vCPUs (10 for SDN, 22 for other) |

In a typical system, the SDN overhead is one physical core per card plus a single Hyper Thread. That
will be the same in the H100 server, however there are 9 CX-7's.

Current layout is:

- 9 physical cores for SDN + 1 HT
- 22 threads remaining (7 physical cores + 15 HT's)

The memory overhead will be 256 GB. SDN is expected to use 68 GB and is documented
[here](https://ibm.ent.box.com/file/1525419580536?s=yk3of4d2kgm6atufceisjq6qv0uq9hix).

That said, huge pages will also be used for this system. 1820 GB of memory will be used for huge
pages to support both the VM's memory plus the 2 GB per NIC that SDN requires.

```text
Huge Pages Allocation

  VM    : 1792 GB
+ SDN HP: 18 GB
+ Buffer: 10 GB
-------------------
          1820 GB
```

Therefore the overall memory allocation works out to:

```text
  VM:            1792 GB
+ SDN HP:        18 GB
+ HP Buffer:     10 GB

+ Root FS:       30 GB
+ SDN Other:     50 GB
+ KDump:         6 GB (NUMA 1)
+ Control Plane: 5 GB
------------------------------
                 1911 GB
```

This leaves `137 GB` of buffer. Some will be used by kernel space (some of which is already
accounted for in the SDN Other). The rest is primarily for control plane plus VM memory tax.

The VM memory tax on a 2048 GB system is 123 GB.

It should also be noted that the SDN Other will come down significantly at run time. The I/O buffers
for SDN's Cluster Network account for 38 GB. Most of those will be assigned directly to the VM via
SR-IOV, so that number will be at least half at the host level. It is only needed when the VM is
provisioning the VF, but before it is assigned to the VM.

###### 3.1.1.7.1.2 Hypervisor Tagging

The system specifications for this the H100 hypervisor server will be named `gx3d`. Additionally, a
parallel `gx3d-research` configuration will be built. This will be applied to the research nodes, as
they generally are more aggressive with new firmware then the `gx3d` class. The physical build out
will be identical between the two. Functionally though, they serve as early evaluation of what rolls
to the standard `gx3d` servers.

The hypervisor will only support the m-zone role of `vpc_compute`.

The following is the instance profile to hardware profile map.

| Instance Profile             | Hardware Profile Class | Hardware Spec          |
| ---------------------------- | ---------------------- | ---------------------- |
| gx3d-160x1792x8h100          | gx3d                   | gx3d-h100-smc          |
| gx3d-160x1792x8h100-research | gx3d-research          | gx3d-h100-smc-research |

See the [Instance Profile Design](#31174-h100-instance-profile-design) for more information on the
instance profiles.

The profile class will be set in the [mzone.yaml](#31151-m-zone-yaml-updates).

Note that this is different from the a100 generation. With the a100 generation, the hardware profile
class for research was `gx2-a100` and for external clients it was `gx2-a100-ext`. In this
generation, research will be the unique configuration. This reflects the evolution of the design
where we support external as the lead with, and research as the bleeding edge design.

Additionally, there are two hardware specs. The physical layout of these will be identical, but
given research is planning to be more aggressive with the firmware roll out, they need an
independent spec to manage their firmware levels. This is a byproduct of the
[infrastructure automation design](https://github.ibm.com/cloudlab/platform-inventory/tree/master/specifications).

###### 3.1.1.7.1.3 Q35 Support

One of the primary feedback items as part of the Vela project was that the
[NCCL Library](https://developer.nvidia.com/nccl) tuning was very difficult. One reason for this is
that the native PCIe tree was lost in translation from the host to the VM. This is the nature of the
`i440fx` machine type that the cloud uses by default.

As part of the [SGX](../3122/) project, Q35 support was added as a machine type. This allowed for
VMs to be created with a proper PCIe topology. However, that support does not allow for any pass
through devices. As part of this project, we need to support PCIe pass through.

The compute code will be augmented to support native pass through of the PCIe tree. An additional
PCIe switch should be created for every additional physical piece of hardware attached to the VM.

![Q35 Tree](./assets/Q35-Tree.png)

If the VM is not enabled to connect to the cluster, there would still be four additional 'static'
PCIe switches. However the Ethernet Controllers would not be attached.

As a reminder, the Cluster enabled vNICs can not be attached live. All cluster NICs must be attached
at VM boot time.

###### 3.1.1.7.1.4 Bandwidth Pooling for Cluster Networks

When the cluster network cardinality is set to 8, each cluster NIC will have up to 400 Gbps of
throughput (a total of 3.2 Tbps).

When the cluster network cardinality is set to 16 or 32, the cluster NIC bandwidth of the peer NICs
are pooled. Example:

| Card | Total Bandwidth | Cluster NICs in Pool |
| ---- | --------------- | -------------------- |
| 1    | 400 Gbps        | 1, 9, 17, 25         |
| 2    | 400 Gbps        | 2, 10, 18, 26        |
| 3    | 400 Gbps        | 3, 11, 19, 27        |
| 4    | 400 Gbps        | 4, 12, 20, 28        |
| 5    | 400 Gbps        | 5, 13, 21, 29        |
| 6    | 400 Gbps        | 6, 14, 22, 30        |
| 7    | 400 Gbps        | 7, 15, 23, 31        |
| 8    | 400 Gbps        | 8, 16, 24, 32        |

The included visual shows the logical cluster network attachment mapping to physical cards.

![Bandwidth pool visual](./assets/bandwidth-pool-visual.png)

###### 3.1.1.7.1.5 VSI Resize

VSI resize will be a supported feature of the H100 profiles. An instance can be resized to the H100
type assuming the user has authority to the appropriate profile.

If they are resizing away from a H100 profile that participates in a cluster, the user will need to
remove the cluster network attachments prior to the resize operation. If they move to a profile that
does not support clustering, the resize operation should fail. Once they remove the cluster network
attachments, the resize operation will be supported.

##### 3.1.1.7.2 Resource Objects

###### 3.1.1.7.2.1 Global Catalog Updates - Instance Profile

Note: Not all attributes (ex. bandwidth) is confirmed.

As a reminder, not all of the data in the Instance Profile's Global Catalog entry is expressed
through the RCOS Instance Profile API. It can provide additional information to RCOS, which can be
passed down to genctl-compute. As this section is reviewed, it will be highlighted which data will
be made external versus what is generally used internally.

None of the data will be worrisome for a customer to see, however their standard interaction is
through the RCOS instance profile API. Meaning if they stumble on some of the data through the
[global catalog web page](https://globalcatalog.cloud.ibm.com/), there is no issue.

The global catalog instance profile object needs to be updated to indicate the following:

- That the instance profile can connect to a cluster network.
- The types of cluster networks that are supported by the instance profile
- The number of cluster network `positions` for each instance profile
- The bandwidth for the positions of the positions.

The `gx3d-160x1792x8h100` profile will have the following:

```json
"metadata": {
  "other": {
    "profile": {
      "clustering": {
        "positions": [1, 2, 3, 4, 5, 6, 7, 8], // how many independent bandwidth groups
        "cardinalities": [0, 1, 2, 4], // how many cluster vNICs per position
        "bandwidth": {
          "type": "fixed",
          "value": 400000 // max bandwidth through a given position, in this case it's line rate
        },
        "port_speed": 0, // 0 indicates uncapped
        "nic_types": ["sr-iov"], // Indicates that these NICs are sr-iov instead of standard virtio
        "cluster_types": ["h100"] // Indicates the cluster profiles that can be used by this instance profile.
      }
    }
  }
}
```

See the [bandwidth pooling](./hld/bandwidth-pooling.md) document for more information.

###### 3.1.1.7.2.2 Regional Instance Profile Structure

The instance profile object will be updated to reflect the additional information within the global
catalog entry. See the [API Design](#31143-cluster-api-design) section for more details. This
section explains how to connect the information from the previous section, into readable information
from the global catalog profile entry.

Two key fields are required to be available on the instance profiles.

- `cluster_network_attachment_count`: This represents the number of cluster network attachments for
  a given profile.

  - Most profiles will have a value of `[0]`, which indicates no cluster network attachments.
    - If 0 is missing from the cardinalities, then it _requires_ that a cluster network be specified
      for the profile.
    - When 0 is not in the list of cardinalities, then a cluster_type must be present in the
      profile. The user will be expected to connect an instance created from this profile with a
      cluster network.
      - This will be exceedingly rare. No known scenarios where this is going to occur, all known
        scenarios will allow for detaching from a cluster. However, we are keeping this possibility
        open for future.
  - The `gx3d-160x1792x8h100` profile will have a value of `[0, 8, 16, 32]`.
  - This value can be determined with the following formula:
    `[len(positions) * cardinality for cardinality in cardinalities]`
  - The specific structure in the API will be:

```json
"cluster_network_attachment_count": {
  "type": "enum",
  "values": [0, 8, 16, 32]
}
```

- `supported_cluster_network_profiles`: This represents the corresponding `cluster_network_profile`
  types that are supported by this instance profiles.
  - This is simply a pass through of the `cluster_types` value from the global catalog entry (though
    it really references the cluster profiles).

###### 3.1.1.7.2.3 Additional Objects on the Instance

The regional (and correspondingly zonal) instance specification will need to be updated to include
the cluster network attachments.

These attachments need to be projected down to zonal for the scheduler information as well.
`ClusterNICs` will be added to the compute spec. It will be a list describing the set of cluster
network attachments to include on an interface. The cluster nics will map back to the corresponding
cluster subnet / cluster network owned by the user.

##### 3.1.1.7.3 VM Scheduler Evolution

There are two elements to scheduling. The first is scheduling the cluster network (virtual) to the
physical cluster implementation. The design for this is captured
[here](https://confluence.swg.usma.ibm.com:8445/display/CTL/Cluster+Scheduling).

Secondly, when a VM is provisioned it will be tagged as participating in a cluster or not based off
the cluster network attachments provided. If there are cluster network attachments, those will point
to a specific cluster network. All cluster network attachments on an instance must point to the same
cluster network provided by the user.

Once the customer's cluster network is identified, the corresponding physical cluster implementation
backing that cluster network is known. The scheduler will then ensure that only hosts
[tagged](#31151-m-zone-yaml-updates) as participating in that physical cluster implementation are
valid for the VSI.

Note: servers that are in a cluster are still candidates to be scheduled to even if the user does
not specify a cluster network attachment.

For this initial GA, there is a restriction of one physical cluster implementation per zone for a
give cluster network profile type. That means a single zone can currently only house a single H100
cluster network. That is expected to expand in a future SRB.

##### 3.1.1.7.4 H100 Instance Profile Design

The instance profile for this node will be `gx3d-160x1792x8h100`. The research specific nodes will
deploy a `gx3d-160x1792x8h100-research` profile. This is functionally equivalent (same BSS plan
even), but will deploy only onto the `gx3d-research` nodes. See the
[Hypervisor Tagging](#311712-hypervisor-tagging) section for information on why research has a
different set of servers.

Both profiles will have:

- 160 vCPUs across 2 sockets
- 1792 GB DDR5 memory
- 8x 80 GB H100 GPUs
- Up to 15 vNICs (with 8 being up to 400 Gbps)
- 8x 7.68 TB NVMe drives (pass through)

Billing for this profile will require a new BSS plan. It will follow the Gen 3 precedent. That
means:

- is.instance
  - 1 new plan
  - 1 new Instance-Hours part
  - 1 new DH instance hours part (won't be used, simply created for consistency)
  - Appropriate parts for images
- is.reservation
  - New parts to support the new profile

Both the consumer/research instance profiles will share the same plan/parts.

##### 3.1.1.7.5 Compute Restrictions / Limitations

All cluster network attachments must be specified on an instance object as part of a single API
invocation.

Cluster Network Attachments can only be set when an instance is stopped or when it is being created.
Cluster Network Attachments do not support hot attach/detach.

Multiple cluster networks on a single instance is not supported. Meaning all Cluster Network
Attachments on a given instance need to connect back to a single Cluster Network.

For this initial view, all servers that are deployed in a zone are either in a cluster or not. If we
determine that we're growing heterogenous hardware in (some servers in a cluster in a zone, others that
are not), this will be revisited as a new requirement.

Reservations will be supported for both clustered and non-clustered H100 instances since all H100
servers in a given zone are either connected to the cluster network or they are not. Thus the reservation
service can simply reserve space for an H100 VSI profile in a given zone, regardless of the H100 cluster
network configuration.

##### 3.1.1.7.6 Dedicated Host Support

Dedicated Hosts will not be supported with this offering. As such, no updates were made to support
cluster networks. Dedicated Hosts will not be supported with this offering. As such, no updates will
be made to support cluster networks.

When an offering that supports dedicated hosts with cluster networks comes into scope, that SRB will
introduce the capability to handle scheduling.

It is expected that the Dedicated Host object will receive a cluster network reference (not the
cluster network subnets). This should provide the ability for the Dedicated Host scheduler to land
on the right set of nodes.

##### 3.1.1.7.7 Auto Scale Support / Instance Templates

Instance templates will be extended to support the `cluster_network_attachments` property as well as
the ability to specify the H100 profile as in the `profile` property. Users will be able to create
instance templates with their desired instance configuration for attaching to a cluster network. The
instance templates can be used independently to provision an instance by referencing a
`source_template`.

Instance groups will inherit support for H100 and cluster networks from instance templates enabling
users to scale their instances automatically with the existing auto scale policies as well as
scheduling scaling events. The `is.cluster-network` service CRN will need to be updated to allow
service to service authorization policy creation with the `is.instance-group` service.

Auto Scale support will be effectively unchanged and should work out of the box.

### 3.1.2 Customer Experience

#### 3.1.2.1 User Interface (UI)

Two new instance profiles are made available, one public H100 profile and a research-only H100
profile. A new panel will also be created to capture the cluster network capabilities.

![Cluster Network Landing Page](./assets/ux/ClusterNetworkLandingPage.png)
![Cluster Network Creation Panel](./assets/ux/ClusterNetworkCreate.png)
![Cluster Network Created Page](./assets/ux/ClusterNetworkCreated.png)
![Instance Create with Cluster](./assets/ux/InstanceCreate.png)

#### 3.1.2.2 Command Line Interface (CLI)

Cluster network service will be available with the IBM Cloud CLI.

Support for cluster networks in the CLI extends to creating, reading, and managing cluster networks,
cluster network subnets, cluster network interfaces, and cluster network attachments.

Examples:

List cluster network profiles:

```text
ibmcloud is cluster-network-profiles
Listing cluster network profiles under account VPCUI-DEV as user Sreekar.B.V@ibm.com...
Name   Family   Zone                               Resource type
h100   vela     us-south-1,us-south-2,us-south-3   cluster_network_profile
```

Create a cluster network:

```text
is cluster-network-create --name cli-cn-1 --vpc cli-vpc --zone us-south-2 --profile h100
Creating cluster network with name cli-cn-1 under account VPCUI-DEV as user Sreekar.B.V@ibm.com...

ID                  0726-e90dc45b-2f42-47ff-ab55-8ab38de69830
Name                cli-cn-1
CRN                 crn:v1:staging:public:is:us-south-2:a/efe5afc483594adaa8325e2b4d1290df::cluster-network:0726-e90dc45b-2f42-47ff-ab55-8ab38de69830
Profile             h100
VPC                 ID                                          Name
                    r134-09ac9db7-3731-4de5-9de0-ed94dfa77d02   cli-vpc

Zone                us-south-2
Subnet Prefixes     Allocation Policy   CIDR
                    auto                10.0.0.0/9

LifeCycle State     pending
LifeCycle Reasons   Code   Message   More Info
                    -      -

Resource type       cluster_network
Resource group      ID                                 Name
                    11caaa983d9c4beb82690daab08717e9   Default

Created at          0001-01-01T05:53:28+05:53
```

CLI support and internal documentation:

- [cluster network](https://github.ibm.com/Bluemix/riaas-cli/blob/dev/docs/cluster_network.md)
- [cluster network attachment](https://github.ibm.com/Bluemix/riaas-cli/blob/dev/docs/cluster_network_attachment.md)
- [cluster network interface](https://github.ibm.com/Bluemix/riaas-cli/blob/dev/docs/cluster_network_interface.md)
- [cluster network subnet](https://github.ibm.com/Bluemix/riaas-cli/blob/dev/docs/cluster_network_subnet.md)
- [cluster network reserved ip](https://github.ibm.com/Bluemix/riaas-cli/blob/dev/docs/cluster_network_subnet_rip.md)

#### 3.1.2.3 Terraform and Packer

Cluster networks will be fully supported with Terraform. This includes terraform templates to deploy
and manage cluster networks alongside their existing VPC resources. Terraform documentation has been
updated to include guidance on creating terraform templates to manage cluster networks and its
resources.

Further details on the scope and implementation can be found
[here](https://github.com/ibm-vpc/terraform-provider-ibm/pull/361).

##### `ibm_is_cluster_network_profile`

Provides a read-only data source to retrieve information about a ClusterNetworkProfile.

```hcl
data "ibm_is_cluster_network_profile" "example" {
  name = "name"
}
```

Provides a read-only data source to retrieve information about a ClusterNetworkProfileCollection.

```terraform
data "ibm_is_cluster_network_profiles" "example" {
}
```

##### `ibm_is_cluster_network`

Provides a read-only data source to retrieve information about a ClusterNetwork.

```hcl
data "ibm_is_cluster_network" "is_cluster_network" {
  is_cluster_network_id = "is_cluster_network_id"
}
```

Create, update, and delete ClusterNetworks with this resource.

```hcl
resource "ibm_is_cluster_network" "is_cluster_network_instance" {
  name = "my-cluster-network"
  profile {
    name = "h100"
  }
  resource_group {
    id = "fee82deba12e4c0fb69c3b09d1f12345"
  }
  subnet_prefixes {
    allocation_policy = "auto"
    cidr = "10.0.0.0/24"
  }
  vpc {
    name = "my-vpc"
  }
  zone {
    name = "us-south-1"
  }
}
```

##### `ibm_is_cluster_network_subnet`

Provides a read-only data source to retrieve information about a ClusterNetworkSubnet.

```hcl
data "ibm_is_cluster_network_subnet" "is_cluster_network_subnet" {
  cluster_network_id = "cluster_network_id"
  is_cluster_network_subnet_id = "is_cluster_network_subnet_id"
}
```

##### `ibm_is_cluster_network_subnet_reserved_ips`

Provides a read-only data source to retrieve information about a ClusterNetworkSubnetReservedIP.

```hcl
data "ibm_is_cluster_network_subnet_reserved_ip" "is_cluster_network_subnet_reserved_ip" {
  cluster_network_id = "cluster_network_id"
  cluster_network_subnet_id = "cluster_network_subnet_id"
  is_cluster_network_subnet_reserved_ip_id = "is_cluster_network_subnet_reserved_ip_id"
}
```

Provides a read-only data source to retrieve information about a
ClusterNetworkSubnetReservedIPCollection.

```hcl
data "ibm_is_cluster_network_subnet_reserved_ips" "is_cluster_network_subnet_reserved_ips" {
  cluster_network_id = "cluster_network_id"
  cluster_network_subnet_id = "cluster_network_subnet_id"
  sort = "name"
}
```

##### `ibm_is_cluster_network_interface`

Create, update, and delete ClusterNetworkInterfaces with this resource.

```hcl
resource "ibm_is_cluster_network_interface" "is_cluster_network_interface_instance" {
  cluster_network_id = "cluster_network_id"
  name = "my-cluster-network-interface"
  primary_ip {
    address = "10.1.0.6"
    name = "my-cluster-network-subnet-reserved-ip"
  }
  subnet {
    name = "my-cluster-network-subnet"
  }
}
```

##### `ibm_is_instance_cluster_network_attachments`

Provides a read-only data source to retrieve information about an InstanceClusterNetworkAttachment.

```hcl
data "ibm_is_instance_cluster_network_attachment" "is_instance_cluster_network_attachment" {
  instance_id = "instance_id"
  is_instance_cluster_network_attachment_id = "is_instance_cluster_network_attachment_id"
}
```

Provides a read-only data source to retrieve information about an
InstanceClusterNetworkAttachmentCollection.

```hcl
data "ibm_is_instance_cluster_network_attachments" "is_instance_cluster_network_attachments" {
  instance_id = "instance_id"
}
```

#### 3.1.2.4 Customer-facing Documentation

The following documents will need to be updated as part of this SRB:

- [Instance Profiles](https://cloud.ibm.com/docs/vpc?topic=vpc-profiles)
  - The new instance profiles will need to be described in the GPU section
- [Cluster Network Documentation](https://cloud.ibm.com/docs/vpc?topic=vpc-about-networking-for-vpc&interface=ui)
  - The network documentation will need to be updated to include a new section that describes the
    cluster network design in IBM cloud.
- [Accelerated Compute Section](https://cloud.ibm.com/docs/vpc?topic=vpc-about-advanced-virtual-servers&interface=ui)
  - A new section for accelerated computing will be required.
  - Will require describe the minimum driver levels for all GPU node types (v100, l4, l40s, a100
    dgx, a100 pcie, and the h100 dgx server)
  - Will require documentation on how to set up RoCE traffic on an H100 DGX node
  - Will need to include a default NCCL configuration document

Additionally, a whitepaper or blog post will be needed to describe the technical design of the
following for our customers:

- Cluster network design philosophy
- Latencies for IBM Cloud Cluster network for H100 nodes, as well as bandwidth and tail latencies
- NCCL optimization for training workloads in the IBM cloud (tied to a specific NCCL version)

[See drafts of customer-facing documentation](https://test.cloud.ibm.com/docs/vpc?topic=vpc-about-cluster-network&interface=ui).

### 3.1.3 Customer-facing APIs

[API Spec Pull Request #5850](https://github.ibm.com/riaas/api-spec/pull/5850) contains the bulk of
the changes introduced, initially. See the full [API spec](https://pages.github.ibm.com/vpc/vpc-spec-artifacts/branch/master/swagger-ui.html?feature=is-cluster-network)
for a browsable view.

There are 27 new `public` operations:

- `GET /cluster_network/profiles`
- `GET /cluster_network/profiles/{name}`
- `GET /cluster_networks`
- `POST /cluster_networks`
- `GET /cluster_networks/{cluster_network_id}/interfaces`
- `POST /cluster_networks/{cluster_network_id}/interfaces`
- `DELETE /cluster_networks/{cluster_network_id}/interfaces/{id}`
- `GET /cluster_networks/{cluster_network_id}/interfaces/{id}`
- `PATCH /cluster_networks/{cluster_network_id}/interfaces/{id}`
- `GET /cluster_networks/{cluster_network_id}/subnets`
- `POST /cluster_networks/{cluster_network_id}/subnets`
- `GET /cluster_networks/{cluster_network_id}/subnets/{cluster_network_subnet_id}/reserved_ips`
- `POST /cluster_networks/{cluster_network_id}/subnets/{cluster_network_subnet_id}/reserved_ips`
- `DELETE /cluster_networks/{cluster_network_id}/subnets/{cluster_network_subnet_id}/reserved_ips/{id}`
- `GET /cluster_networks/{cluster_network_id}/subnets/{cluster_network_subnet_id}/reserved_ips/{id}`
- `PATCH /cluster_networks/{cluster_network_id}/subnets/{cluster_network_subnet_id}/reserved_ips/{id}`
- `DELETE /cluster_networks/{cluster_network_id}/subnets/{id}`
- `GET /cluster_networks/{cluster_network_id}/subnets/{id}`
- `PATCH /cluster_networks/{cluster_network_id}/subnets/{id}`
- `DELETE /cluster_networks/{id}`
- `GET /cluster_networks/{id}`
- `PATCH /cluster_networks/{id}`
- `GET /instances/{instance_id}/cluster_network_attachments`
- `POST /instances/{instance_id}/cluster_network_attachments`
- `DELETE /instances/{instance_id}/cluster_network_attachments/{id}`
- `GET /instances/{instance_id}/cluster_network_attachments/{id}`
- `PATCH /instances/{instance_id}/cluster_network_attachments/{id}`

#### Updated Public Operations

There are 10 updated `public` operations:

- `GET /instance/profiles`
  - add SchemaProperty [200] `/profiles/*/cluster_network_attachment_count*`
  - add SchemaProperty [200] `/profiles/*/supported_cluster_network_profiles*`
- `GET /instance/profiles/{name}`
  - add SchemaProperty [200] `/cluster_network_attachment_count*`
  - add SchemaProperty [200] `/supported_cluster_network_profiles*`
- `GET /instance/templates`
  - add SchemaProperty [200] `/templates/*/cluster_network_attachments`
- `POST /instance/templates`
  - add SchemaProperty [request] `/cluster_network_attachments`
  - add SchemaProperty [201] `/cluster_network_attachments`
- `GET /instance/templates/{id}`
  - add SchemaProperty [200] `/cluster_network_attachments`
- `PATCH /instance/templates/{id}`
  - add SchemaProperty [200] `/cluster_network_attachments`
- `GET /instances`
  - add OperationParameter [request] `cluster_network.id` in query
  - add OperationParameter [request] `cluster_network.crn` in query
  - add OperationParameter [request] `cluster_network.name` in query
  - add SchemaProperty [200] `/instances/*/cluster_network`
  - add SchemaProperty [200] `/instances/*/cluster_network_attachments`
- `POST /instances`
  - add SchemaProperty [request] `/cluster_network_attachments`
  - add SchemaProperty [201] `/cluster_network`
  - add SchemaProperty [201] `/cluster_network_attachments`
- `GET /instances/{id}`
  - add SchemaProperty [200] `/cluster_network`
  - add SchemaProperty [200] `/cluster_network_attachments`
- `PATCH /instances/{id}`
  - add SchemaProperty [200] `/cluster_network`
  - add SchemaProperty [200] `/cluster_network_attachments`

#### New Metadata Operations

There are 2 new `metadata` operations:

- `GET /metadata/v1/instance/cluster_network_attachments`
- `GET /metadata/v1/instance/cluster_network_attachments/{id}`

#### Updated Metadata Operations

There is 1 updated `metadata` operation:

- `GET /metadata/v1/instance`
  - add SchemaProperty [200] `/cluster_network`
  - add SchemaProperty [200] `/cluster_network_attachments`

#### New customer-facing concepts

- **Instance Profile - Cluster Network Attachments** - With the H100 instance profile, customers can
  provision instances that expose thew new GPU configuration. Instances created with this profile
  have a new `cluster_network_attachments` property with which to connect to cluster networks. This
  is an optional property, and, if not set will be used for inferencing. If set, the cluster
  networking will be configured and the instance will be properly configured for a training
  workload.
- **Cluster Network** - A cluster network is a high performance network within a VPC's boundary, but
  with it's own isolated IPv4 address space. Each cluster network has associated a cluster network
  profile that describes the type of components that it may interconnect. Within the cluster network
  are a set of basic networking abstractions to provide the customer with the flexibility and
  control needed to configure high performance workloads. If a customer requests to create a cluster
  network in a region or zone where it is not supported, then a `400` response will be returned.
- **Cluster Network Subnet** - A cluster network subnet is a subnet within a cluster network. It
  does not have as many features as VPC subnets, but it does allow the customer to specify the
  subnet CIDR and makes possible cluster network subnet reserved IPs.
- **Cluster Network Subnet Reserved IP** - Just as with VPC subnet reserved IPs, one may reserve IP
  addresses in the cluster network subnets as well. These reserved IPs support customer-specified
  addresses and an auto_delete mechanism just as with VPC subnet reserved IPs.
- **Cluster Network Interfaces** - The cluster network interface is a resource similar to the VPC's
  virtual network interface but scoped to a cluster network. This mechanism will provide means to
  extend the networking abstractions in the future. However, today, only `primary_ip` and
  `auto_delete` will be supported.
- **Instance Cluster Network Attachments** - Instances that connect to a cluster network, will
  declare cluster network subnet connectivity and addressing on an array of instance cluster network
  attachments. These attachments are ordered and propagated to the instance configuration on the
  hypervisor to provide consistent bringup. Each physical NIC connected to the instance may host
  multiple network attachments as allowed by the instance profile. By effect, this divides the
  bandwidth across all attachments on that NIC. By passing in a multiple of the physical number of
  NICs on the instance of cluster network attachments, the system will configure an evenly
  distributed setup of attachments across the NICs.
- **Instance Lifecycle State Changes** - Instances will be in a `waiting` state if the number of
  cluster network attachments in a non-deleting state is invalid based on the instance profile. If
  there are any cluster network attachments is in a `deleting` state, then the instance will be in
  an `updating` state. If the sum of cluster network attachments in a non-deleting state is valid,
  then the instance will be in a `stable` state.

| Valid sum of `stable` CNAs | invalid sum of `stable` CNAs | CNAs `deleting` | Instance state |
| :------------------------: | :--------------------------: | :-------------: | :------------: |
|            `1`             |             `0`              |       `0`       |    `stable`    |
|            `1`             |             `0`              |       `1`       |   `updating`   |
|            `0`             |             `1`              |       `0`       |   `waiting`    |
|            `0`             |             `0`              |       `1`       |   `updating`   |
|            `0`             |             `1`              |       `1`       |   `updating`   |

API spec changes part of this proposal:

- [6053](https://github.ibm.com/riaas/api-spec/pull/6053)
- [6044](https://github.ibm.com/riaas/api-spec/pull/6044)
- [6025](https://github.ibm.com/riaas/api-spec/pull/6025)
- [6017](https://github.ibm.com/riaas/api-spec/pull/6017)
- [5973](https://github.ibm.com/riaas/api-spec/pull/5973)
- [5962](https://github.ibm.com/riaas/api-spec/pull/5962)
- [5850](https://github.ibm.com/riaas/api-spec/pull/5850)

#### 3.1.3.1 Activity Tracker (AT)

#### AT Changes: public

(Note: `*` indicates an even that is always generated; other events are conditionally generated.)

| Operation                                                                                           | AT Events                                                                                                                                                                                                                                                                                                             |
| --------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| GET /cluster_network/profiles                                                                       | `is.cluster-network.profile.read*`                                                                                                                                                                                                                                                                                    |
| GET /cluster_network/profiles/{name}                                                                | `is.cluster-network.profile.read*`                                                                                                                                                                                                                                                                                    |
| GET /cluster_networks                                                                               | `is.cluster-network.cluster-network.read*`                                                                                                                                                                                                                                                                            |
| POST /cluster_networks                                                                              | `is.cluster-network.cluster-network.create*`                                                                                                                                                                                                                                                                          |
| GET /cluster_networks/{cluster_network_id}/interfaces                                               | `is.cluster-network.interface.read*`                                                                                                                                                                                                                                                                                  |
| POST /cluster_networks/{cluster_network_id}/interfaces                                              | `is.cluster-network.interface.attach*`, `is.cluster-network.interface.create*`, `is.cluster-network.subnet-reserved-ip.attach*`, `is.cluster-network.subnet-reserved-ip.create`, `is.cluster-network.subnet.update`                                                                                                   |
| DELETE /cluster_networks/{cluster_network_id}/interfaces/{id}                                       | `is.cluster-network.interface.delete*`, `is.cluster-network.interface.detach*`, `is.cluster-network.subnet.update`, `is.cluster-network.subnet-reserved-ip.delete`, `is.cluster-network.subnet-reserved-ip.detach*`                                                                                                   |
| GET /cluster_networks/{cluster_network_id}/interfaces/{id}                                          | `is.cluster-network.interface.read*`                                                                                                                                                                                                                                                                                  |
| PATCH /cluster_networks/{cluster_network_id}/interfaces/{id}                                        | `is.cluster-network.interface.update*`                                                                                                                                                                                                                                                                                |
| GET /cluster_networks/{cluster_network_id}/subnets                                                  | `is.cluster-network.subnet.read*`                                                                                                                                                                                                                                                                                     |
| POST /cluster_networks/{cluster_network_id}/subnets                                                 | `is.cluster-network.cluster-network.update*`, `is.cluster-network.subnet.create*`                                                                                                                                                                                                                                     |
| GET /cluster_networks/{cluster_network_id}/subnets/{cluster_network_subnet_id}/reserved_ips         | `is.cluster-network.subnet-reserved-ip.read*`                                                                                                                                                                                                                                                                         |
| POST /cluster_networks/{cluster_network_id}/subnets/{cluster_network_subnet_id}/reserved_ips        | `is.cluster-network.subnet.update*`, `is.cluster-network.subnet-reserved-ip.create*`                                                                                                                                                                                                                                  |
| DELETE /cluster_networks/{cluster_network_id}/subnets/{cluster_network_subnet_id}/reserved_ips/{id} | `is.cluster-network.subnet.update*`, `is.cluster-network.subnet-reserved-ip.delete*`                                                                                                                                                                                                                                  |
| GET /cluster_networks/{cluster_network_id}/subnets/{cluster_network_subnet_id}/reserved_ips/{id}    | `is.cluster-network.subnet-reserved-ip.read*`                                                                                                                                                                                                                                                                         |
| PATCH /cluster_networks/{cluster_network_id}/subnets/{cluster_network_subnet_id}/reserved_ips/{id}  | `is.cluster-network.subnet-reserved-ip.update*`                                                                                                                                                                                                                                                                       |
| DELETE /cluster_networks/{cluster_network_id}/subnets/{id}                                          | `is.cluster-network.cluster-network.update*`, `is.cluster-network.subnet.delete*`                                                                                                                                                                                                                                     |
| GET /cluster_networks/{cluster_network_id}/subnets/{id}                                             | `is.cluster-network.subnet.read*`                                                                                                                                                                                                                                                                                     |
| PATCH /cluster_networks/{cluster_network_id}/subnets/{id}                                           | `is.cluster-network.subnet.update*`                                                                                                                                                                                                                                                                                   |
| DELETE /cluster_networks/{id}                                                                       | `is.cluster-network.cluster-network.delete*`, `is.cluster-network.interface.delete`, `is.cluster-network.subnet-reserved-ip.delete`, `is.cluster-network.subnet.delete`                                                                                                                                               |
| GET /cluster_networks/{id}                                                                          | `is.cluster-network.cluster-network.read*`                                                                                                                                                                                                                                                                            |
| PATCH /cluster_networks/{id}                                                                        | `is.cluster-network.cluster-network.update*`                                                                                                                                                                                                                                                                          |
| GET /instances/{instance_id}/cluster_network_attachments                                            | `is.instance.cluster-network-attachment.read*`                                                                                                                                                                                                                                                                        |
| POST /instances/{instance_id}/cluster_network_attachments                                           | `is.cluster-network.interface.attach*`, `is.cluster-network.interface.create`, `is.cluster-network.subnet.update`, `is.cluster-network.subnet-reserved-ip.attach`, `is.cluster-network.subnet-reserved-ip.create`, `is.instance.cluster-network-attachment.attach*`, `is.instance.cluster-network-attachment.create*` |
| DELETE /instances/{instance_id}/cluster_network_attachments/{id}                                    | `is.cluster-network.interface.delete`, `is.cluster-network.interface.detach`, `is.cluster-network.subnet-reserved-ip.delete`, `is.cluster-network.subnet-reserved-ip.detach`, `is.cluster-network.subnet.update`, `is.instance.cluster-network-attachment.delete*`, `is.instance.cluster-network-attachment.detach*`  |
| GET /instances/{instance_id}/cluster_network_attachments/{id}                                       | `is.instance.cluster-network-attachment.read*`                                                                                                                                                                                                                                                                        |
| PATCH /instances/{instance_id}/cluster_network_attachments/{id}                                     | `is.instance.cluster-network-attachment.update*`                                                                                                                                                                                                                                                                      |

#### AT Changes: internal

| Operation                                                        | AT Events                                                                                                                                                                                                                                                                                                             |
| ---------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| GET /instances/{instance_id}/cluster_network_attachments         | `is.instance.cluster-network-attachment.read*`                                                                                                                                                                                                                                                                        |
| POST /instances/{instance_id}/cluster_network_attachments        | `is.cluster-network.interface.attach*`, `is.cluster-network.interface.create`, `is.cluster-network.subnet.update`, `is.cluster-network.subnet-reserved-ip.attach`, `is.cluster-network.subnet-reserved-ip.create`, `is.instance.cluster-network-attachment.attach*`, `is.instance.cluster-network-attachment.create*` |
| DELETE /instances/{instance_id}/cluster_network_attachments/{id} | `is.cluster-network.interface.delete`, `is.cluster-network.interface.detach`, `is.cluster-network.subnet-reserved-ip.delete`, `is.cluster-network.subnet-reserved-ip.detach`, `is.cluster-network.subnet.update`, `is.instance.cluster-network-attachment.delete*`, `is.instance.cluster-network-attachment.detach*`  |
| GET /instances/{instance_id}/cluster_network_attachments/{id}    | `is.instance.cluster-network-attachment.read*`                                                                                                                                                                                                                                                                        |
| PATCH /instances/{instance_id}/cluster_network_attachments/{id}  | `is.instance.cluster-network-attachment.update*`                                                                                                                                                                                                                                                                      |

#### AT Changes: metadata

| Operation                                                  | AT Events                                               |
| ---------------------------------------------------------- | ------------------------------------------------------- |
| GET /metadata/v1/instance/cluster_network_attachments      | `is.metadata.instance-cluster-network-attachment.read*` |
| GET /metadata/v1/instance/cluster_network_attachments/{id} | `is.metadata.instance-cluster-network-attachment.read*` |

#### 3.1.3.2 Global Search and Tagging (GhoST)

Cluster networks will integrate with GhoST and ReX. There is an outstanding known issue that must be
addressed for GA. See known issues in [section 6](#6-known-issues).

#### 3.1.3.3 IBM Cloud Compliance and Security Center (IBM SCC)

Users of the H100 GPU node that choose not to use the cluster network will be FS Cloud compliant.

Per the SEC046 guidance, IBM SCC rules will not be updated for cluster networking. The SCC rules are
currently paused. This is signed off in the
[SEC046](https://github.ibm.com/ibmcloud/ciso-signoff/issues/2099#issuecomment-74258143).

When the requirement is reactivated, the IBM SCC will need to be updated however to detect if the
customer is utilizing the cluster network. Since the cluster network does not support flow logs, it
will need to indicate that use of that feature is not FS Cloud compliant.

### 3.1.4 Internal APIs and Schemas

A new operator API for cluster networks to physical cluster implementation may be introduced in a
follow up SRB (when we have more than one physical cluster implementation within a given zone).

There will be updates to the threadbare, which impacts scheduling overhead. See section
`Hypervisor Overhead` to see the new overhead values. This is typically expressed as part of the
host bring up with labels/annotations on the nodes.

#### New Internal Operations

There are 5 new `internal` operations:

- `GET /instances/{instance_id}/cluster_network_attachments`
- `POST /instances/{instance_id}/cluster_network_attachments`
- `DELETE /instances/{instance_id}/cluster_network_attachments/{id}`
- `GET /instances/{instance_id}/cluster_network_attachments/{id}`
- `PATCH /instances/{instance_id}/cluster_network_attachments/{id}`

#### Updated Internal Operations

There are 4 updated `internal` operations:

- `GET /instances`
  - add OperationParameter [request] `cluster_network.id` in query
  - add OperationParameter [request] `cluster_network.crn` in query
  - add OperationParameter [request] `cluster_network.name` in query
  - add SchemaProperty [200] `/instances/*/cluster_network`
  - add SchemaProperty [200] `/instances/*/cluster_network_attachments`
- `POST /instances`
  - add SchemaProperty [request] `/cluster_network_attachments`
  - add SchemaProperty [201] `/cluster_network`
  - add SchemaProperty [201] `/cluster_network_attachments`
- `GET /instances/{id}`
  - add SchemaProperty [200] `/cluster_network`
  - add SchemaProperty [200] `/cluster_network_attachments`
- `PATCH /instances/{id}`
  - add SchemaProperty [200] `/cluster_network`
  - add SchemaProperty [200] `/cluster_network_attachments`

#### Updated Partner Operations

As part of the `is-volume-preallocation-profile` feature, the instances paths are slated to be included
in the partner exposure. The new instance cluster_network_attachment operations will be included
for the sake of consistency. These operations will not be delivered as part of the cluster network
`ga` promotion, but instead remain under the `is-volume-preallocation-profile` feature.

There are 4 updated `partner` operations:

- `GET /instances`
  - add OperationParameter [request] `cluster_network.id` in query
  - add OperationParameter [request] `cluster_network.crn` in query
  - add OperationParameter [request] `cluster_network.name` in query
  - add SchemaProperty [200] `/instances/*/cluster_network`
  - add SchemaProperty [200] `/instances/*/cluster_network_attachments`
- `POST /instances`
  - add SchemaProperty [request] `/cluster_network_attachments`
  - add SchemaProperty [201] `/cluster_network`
  - add SchemaProperty [201] `/cluster_network_attachments`
- `GET /instances/{id}`
  - add SchemaProperty [200] `/cluster_network`
  - add SchemaProperty [200] `/cluster_network_attachments`
- `PATCH /instances/{id}`
  - add SchemaProperty [200] `/cluster_network`
  - add SchemaProperty [200] `/cluster_network_attachments`

### 3.1.5 Deployment Architecture

This SRB introduces a new hypervisor server to environments. Those hypervisor servers will be
tagged, see the [Hypervisor Tagging](#311712-hypervisor-tagging) for details. Given each hypervisor
server will be tagged, the only Instance Profile that will be deployed to it will be the ones
introduced in the [Instance Profile](#31174-h100-instance-profile-design).

As such, hypervisor servers will be non-disruptively added to the zone.

The H100 instance profiles are available publicly to customers, however they do require a quota bump
to use. This was a late change from the offering management team, where previously it would be
behind an allow-list. This was approved by IBM Cloud's Senior Executive team.

The h100 cluster network profile utilizes an allow-list for the beta.  For GA it will be available
globally.

The following feature flags will be used to gate the enablement of the feature

| Feature Flag Name                                    | Type     | Scope    | Consumers            | Purpose                                                                                   |
| ---------------------------------------------------- | -------- | -------- | -------------------- | ----------------------------------------------------------------------------------------- |
| `is-cluster-network`                                 | Maturity | Regional | RNOS, RCOS, RMDS, FF | Maturity flag to enable the new APIs and new properties                                   |
| `is-cluster-network-allow-list`                      | Boolean  | Regional | RCOS, RNOS, FF       | allow-list flag to gate whether callers may specify cluster network properties            |
| `is-cluster-network-allocation-status-check-enabled` | Boolean  | Regional | RNOS                 | Development flag to skip waiting for cluster network allocation before reporting `stable` |
| `regional-health-enable-cluster-network`             | Boolean  | Regional | RHW                  | Enable running healthchecks for is.cluster-network                                        |
| `is-h100-q35-use-static-device-groups`               | Boolean  | Zonal    | Genctl-Compute       | Genctl-Compute internal property to ensures that VSIs will use a static q35 topology      |
| `is-cluster-network-genctl-network`                  | Boolean  | Zonal    | Genctl-Network       | Enable genctl-network processing of objects for is.cluster-network                        |

The UI will leverage the `is-cluster-network` and `is-cluster-network-allow-list` as exposed through
the internal Features API to onboard during the development time-frame and to provide beta and
select availability experiences for customers.

For reference, see
[this commit](https://github.ibm.com/riaas/api-spec/commit/3892cbc13eaa47b126dab72965e501a15e311e4c)
to define feature flags to the internal features API.

## 3.2 Architectural Considerations and Impact

### 3.2.1 Security

This project does not improve or hurt overall IBM Cloud security.

It extends existing design patterns and provides a new part with faster/more of the same base parts.

There are no new network entry points. The SDN protects both the cluster and the cloud NIC. In the
case of a cluster, a new SDN router is created behind the scenes. All traffic that flows through the
cluster network is isolated to the cluster network itself.

The serer has no change in scope from the A100 project. THe single tenant nature of the box means
that the NVLink can be safely passed through into the VM. When the customer is utilizing the GPUs,
the NVLink bus that connects the GPUs is passed directly through as well. This precedent was set in
the A100 HGX integration project (Vela 1).

#### 3.2.1.1 FS Cloud Considerations

This server will be able to be used in a FS Cloud configuration. However, if the customer enables
the clustering capability, it will no longer be FS Cloud compliant. This is because flow logs can
not be supported on the cluster network.

See the
[IBM Cloud Compliance and Security Center](#3133-ibm-cloud-compliance-and-security-center-ibm-scc)
section for more details as to why the cluster capability is not FS Cloud compliant.

#### 3.2.1.2 Secrets

No new secrets are required.

#### 3.2.1.3 Control Plane Hardening

No impact to control plane hardening.

#### 3.2.1.4 Significant Change Considerations

This solution will not be deployed within the FedRAMP boundary. If offering requires that, it will
be a separate SRB.

This solution leverages the same SDN logic on the cluster subnet for protections, as the core IBM
Cloud network. The cluster resource simply restricts the traffic within the aggregation network.

Cluster network traffic is fully segmented and is in a separate address space from a given
customer's VPC network. Interactions with resources outside of the cluster network flows through a
separate physical NIC from the other cluster network interfaces. This ensures the cluster network is
behind a boundary and isolated from external traffic.

![cluster-network](./assets/cluster-network.png)

| Yes | No  | N/A |                                                                                                                                              |
| :-- | :-- | :-- | :------------------------------------------------------------------------------------------------------------------------------------------- |
|     | x   |     | Introduce a new major/minor Linux distribution or base image into your service’s compute resources (hypervisor, VSI, container)              |
| x   |     |     | Change impacted service's trust zones or change how impacted service's trust zones are mapped to network segments                            |
|     | x   |     | Change Ports/Protocols/Services exposed to public routes                                                                                     |
|     | x   |     | Change the connections between IBM operated components/control planes that cross the boundaries of your service’s trust zones                |
|     | x   |     | Change the network encryption used to protect data in flight                                                                                 |
|     | x   |     | Change the database/datastore used to persist service data/metadata                                                                          |
|     | x   |     | Change the encryption used to protect data at rest                                                                                           |
|     | x   |     | Change the classification of data processed by any portion of your compute resources to a higher level                                       |
|     | x   |     | Introduce new cryptographic libraries or modules, including switching to different versions                                                  |
|     | x   |     | Change the list of dependencies that your service leverages                                                                                  |
| x   |     |     | Change any hardware your service leverages                                                                                                   |
|     | x   |     | Introduce changes to the software packages used by your service components                                                                   |
|     | x   |     | Introduce new privileged process (a new system process or container running with sudo/root)                                                  |
|     | x   |     | Change how the components of your service authenticate and authorize to each other                                                           |
|     | x   |     | Change to operators remote access (including the network encryption used)                                                                    |
|     | x   |     | Change how impacted services collect audit information, OR what audit information is collected                                               |
|     | x   |     | Change identities and/or access                                                                                                              |
|     | x   |     | Change the following IBM Cloud public documentation: regional availability, high availability considerations, BCDR customer responsibilities |

**Change any hardware your service leverages** The cluster network introduces new hardware detailed
in [section 3.1.1.3.3](#31133-physical-network-architecture). Additionally, the introduction of the
H100 profile and server design detailed in [section 3.1.1.3.1](#31131-server-node-design).

**Change impacted service's trust zones or change how impacted service's trust zones are mapped to**
**network segments** The cluster network impacts the "Customer on Premise Assets or Customer VPC"
trust zone. It extends this trust zone to include the customer's cluster network. Logically, it is a
part of the customer's VPC, however it is in a separate address space from the customer's VPC.

#### 3.2.1.5 Identity and Access Management

#### IAM Changes

#### IAM Changes: public

(Note: `*` indicates an IAM action that is always required; other actions are conditionally
required.)

| Operation                                                                                           | IAM Actions                                                                                                                                                                            |
| --------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| GET /cluster_network/profiles                                                                       | `is.cluster-network.profile.list*`                                                                                                                                                     |
| GET /cluster_network/profiles/{name}                                                                | `is.cluster-network.profile.read*`                                                                                                                                                     |
| GET /cluster_networks                                                                               | `is.cluster-network.cluster-network.list*`, `is.cluster-network.cluster-network.read*`                                                                                                 |
| POST /cluster_networks                                                                              | `is.cluster-network.cluster-network.create*`, `is.vpc.vpc.operate*`                                                                                                                    |
| GET /cluster_networks/{cluster_network_id}/interfaces                                               | `is.cluster-network.interface.list*`                                                                                                                                                   |
| POST /cluster_networks/{cluster_network_id}/interfaces                                              | `is.cluster-network.interface.create*`, `is.cluster-network.subnet.operate`, `is.cluster-network.subnet.update`                                                                        |
| DELETE /cluster_networks/{cluster_network_id}/interfaces/{id}                                       | `is.cluster-network.interface.delete*`                                                                                                                                                 |
| GET /cluster_networks/{cluster_network_id}/interfaces/{id}                                          | `is.cluster-network.interface.read*`                                                                                                                                                   |
| PATCH /cluster_networks/{cluster_network_id}/interfaces/{id}                                        | `is.cluster-network.interface.update*`                                                                                                                                                 |
| GET /cluster_networks/{cluster_network_id}/subnets                                                  | `is.cluster-network.subnet.list*`                                                                                                                                                      |
| POST /cluster_networks/{cluster_network_id}/subnets                                                 | `is.cluster-network.subnet.create*`                                                                                                                                                    |
| GET /cluster_networks/{cluster_network_id}/subnets/{cluster_network_subnet_id}/reserved_ips         | `is.cluster-network.subnet.read*`                                                                                                                                                      |
| POST /cluster_networks/{cluster_network_id}/subnets/{cluster_network_subnet_id}/reserved_ips        | `is.cluster-network.subnet.update*`                                                                                                                                                    |
| DELETE /cluster_networks/{cluster_network_id}/subnets/{cluster_network_subnet_id}/reserved_ips/{id} | `is.cluster-network.subnet.update*`                                                                                                                                                    |
| GET /cluster_networks/{cluster_network_id}/subnets/{cluster_network_subnet_id}/reserved_ips/{id}    | `is.cluster-network.subnet.read*`                                                                                                                                                      |
| PATCH /cluster_networks/{cluster_network_id}/subnets/{cluster_network_subnet_id}/reserved_ips/{id}  | `is.cluster-network.subnet.update*`                                                                                                                                                    |
| DELETE /cluster_networks/{cluster_network_id}/subnets/{id}                                          | `is.cluster-network.subnet.delete*`                                                                                                                                                    |
| GET /cluster_networks/{cluster_network_id}/subnets/{id}                                             | `is.cluster-network.subnet.read*`                                                                                                                                                      |
| PATCH /cluster_networks/{cluster_network_id}/subnets/{id}                                           | `is.cluster-network.subnet.update*`                                                                                                                                                    |
| DELETE /cluster_networks/{id}                                                                       | `is.cluster-network.cluster-network.delete*`                                                                                                                                           |
| GET /cluster_networks/{id}                                                                          | `is.cluster-network.cluster-network.read*`                                                                                                                                             |
| PATCH /cluster_networks/{id}                                                                        | `is.cluster-network.cluster-network.update*`                                                                                                                                           |
| GET /instances/{instance_id}/cluster_network_attachments                                            | `is.instance.instance.read*`                                                                                                                                                           |
| POST /instances/{instance_id}/cluster_network_attachments                                           | `is.cluster-network.interface.create`, `is.cluster-network.interface.operate`, `is.cluster-network.subnet.operate`, `is.cluster-network.subnet.update`, `is.instance.instance.update*` |
| DELETE /instances/{instance_id}/cluster_network_attachments/{id}                                    | `is.instance.instance.update*`                                                                                                                                                         |
| GET /instances/{instance_id}/cluster_network_attachments/{id}                                       | `is.instance.instance.read*`                                                                                                                                                           |
| PATCH /instances/{instance_id}/cluster_network_attachments/{id}                                     | `is.instance.instance.update*`                                                                                                                                                         |

#### IAM Changes: internal

| Operation                                                        | IAM Actions                                                                                                                                                                            |
| ---------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| GET /instances/{instance_id}/cluster_network_attachments         | `is.instance.instance.read*`                                                                                                                                                           |
| POST /instances/{instance_id}/cluster_network_attachments        | `is.cluster-network.interface.create`, `is.cluster-network.interface.operate`, `is.cluster-network.subnet.operate`, `is.cluster-network.subnet.update`, `is.instance.instance.update*` |
| DELETE /instances/{instance_id}/cluster_network_attachments/{id} | `is.instance.instance.update*`                                                                                                                                                         |
| GET /instances/{instance_id}/cluster_network_attachments/{id}    | `is.instance.instance.read*`                                                                                                                                                           |
| PATCH /instances/{instance_id}/cluster_network_attachments/{id}  | `is.instance.instance.update*`                                                                                                                                                         |

Can be viewed in the [API spec PR](https://github.ibm.com/riaas/api-spec/pull/6053).

### 3.2.2 Performance, Scalability, and Resource Consumption

See section 3.1 for in depth details.

The scale for the environments is described in the [deployments](#261-deployments) section.

The hypervisor overhead is 256G overall.

The performance team will develop training workload tests to be periodically run against the 4-node
staging environment. If regressions are detected, then software releases can be stopped.

See the [requirements](#23-requirements-and-use-cases) for additional performance needs.

Provisioning of an H100 instance is expected to take 20-30 minutes. This is due to the high amounts
of PCIe connectivity. The libvirt start times are quite slow compared to standard VSIs.

#### 3.2.2.1 Performance Test

For the H100 project, the team produced a new test suite that is built with IBM's Granite models.
Additionally, Llama testing occurred. Staging has only have 2 nodes for the test, but in the WDC07
production environment, the team tested up to 128 nodes before hand off to the clients.

#### 3.2.2.2 Always-On Staging Test

The staging environment is [earmarked](#261-deployments) four servers. At any given time, three of
those should be active in production.

Two of those nodes will be dedicated for an always on test. It needs to stress the network, and
verify that the node to node communication is robust. An alert must be produced as part of this test
if a blip is discovered. If the network blip is discovered, both the GPU and SDN development teams
must be notified.

This is to ensure that all change requests validated in Staging can confirm that they do not impact
the AI workload. Having an always on test also ensures that changes executed by independent teams
are still verified. Ex. if Acadia delivers a change we can confidently assert that it does not
affect the AI workloads in production (or does).

The performance team must build and maintain this test. The GPU/SDN dev and SRE teams must be
alerted when the performance dips. Initial triage should occur within 8 work hours.

### 3.2.3 Monitoring, Diagnostics, and Troubleshooting

These nodes are sufficiently complex that the failure rate is expected to be high - double digit
within the first 2 years. This is based off experience with the A100 DGX design.

During that time, two primary issues have been identified. Nodal failures and node degradations.

Node failures are typically due to GPU failures or GPU Switch Board Failures. Other failures like
DIMM or CPU failures are significantly less common, but when they occur they do crash the nodes.

![GPU Failure](./assets/Vela2_Part_Failure.png)

When a part fails, the standard lost node remediation framework kicks in and a new node will be
executed. The infrastructure automation will kick in to automatically diagnose if the failure is a
hardware failure or a software failure. If software, the node will be re-added, but if a hardware
failure it will go to DC Ops and the team will evaluate for a RMA.

Degradations are significantly more complex. When a node degrades, IBM needs to detect this and
inform the customer so that they can move to a new node. Degradations typically occur for the
following reasons:

- PCIe Link Degredation
- NIC port failure (one port failed, the other active)
- Power Capping Enabled (cpu/gpu throttling)

A new agent is being introduced on the compute node to detect the states of these components. It
will be plumbed to provide events/alerts which will be used to inform SRE of an issue. The SRE teams
will plumb the communications with the customers for degradations as some customers will want to
stay on the degraded node where as others may want to move.

The diagnostics agent was proposed in
[SRB 4195 - Compute Diagnostics Agent](https://github.ibm.com/cloudlab/srb/pull/4423), and was
intended to be delivered prior to vela 2 GA, however it is now out of scope due to development
capacity constraints and prioritization. Host diagnostics and observability is still an issue that
will need to be addressed in future development cycles.

Additionally the SuperMicro system for Vela2 has been updated to support emitting the GPU and NVLink
health through the OOB connections (BMC). This will allow IBM to observe the devices more
holistically, without the need for challenging hypervisor mechanisms (which often work only if bare
metal, but fail in virtualized environments).

### 3.2.4 Lifecycle, Metering, and Billing

The only top-level resource with a new life cycle is the Cluster Network. The cluster network will
be registered with resource controller synchronously. The cluster network is a zonal resource that
can be created and deleted. When created, subnets and interfaces can be created as child resources
to the cluster network. Cluster network attachments are created between the instance and the cluster
network interfaces.

Details on the cluster network CRN can be found
[here](../../architecture/platformIntegration/VPC_CRN.md#vpc-crn-list).

#### Cluster Network Lifecycle States

| State    | Description                                                    |
| -------- | -------------------------------------------------------------- |
| Pending  | Resource created but not yet allocated to a physical cluster   |
| Stable   | Resource is created and physical cluster is allocated          |
| Deleting | A request to delete the resource has been accepted             |
| Failed   | Resource is in an unexpected state from failed allocation, etc |

##### Cluster Network Future Lifecycle States

| State   | Description                                                                                        |
| ------- | -------------------------------------------------------------------------------------------------- |
| Waiting | Cluster network allocation has been revoked and a new physical cluster allocation is being created |

#### Cluster Network Lifecycle Transitions

| From     | To       | Description                                                                                                     |
| -------- | -------- | --------------------------------------------------------------------------------------------------------------- |
| -        | Pending  | POST /cluster_networks is called successfully                                                                   |
| Pending  | Stable   | Cluster network is allocated to a physical cluster successfully                                                 |
| Stable   | Deleting | DELETE /cluster_networks/{id} is called when there are no instances bound to interfaces in this cluster network |
| Deleting | -        | Resource is fully deleted from the backend systems and de-provisioned from the data plane                       |

##### Cluster Network Future Lifecycle Transitions

| From    | To      | Description                                                                                                                      |
| ------- | ------- | -------------------------------------------------------------------------------------------------------------------------------- |
| Stable  | Waiting | Cluster network with no instances or stopped instances is unbound from a physical cluster as per ClusterNetworkAllocation record |
| Waiting | Stable  | Cluster network is allocated to a physical cluster as per ClusterNetworkAllocation record                                        |

#### Other New Resources

The other new resources/sub-resources being introduced:

- Cluster network profiles
- Cluster network subnet
- Cluster network subnet reserved ip
- Cluster network interfaces
- Instance cluster network attachments

These will retain the life cycle of their root object.

#### Billing/Metering

Billing and metering will only be for the Instance itself. The cluster network is not an additional
charge feature. That cost is rolled into the infrastructure cost for the instance. For details on
instance billing and how it works, please see the [CBLM SRB](../3488/README.md).

##### BSS-Integration Onboarding

A new cluster network lifecycle event is defined in shared-eventing and includes the usage of
`RegistrationData`.

- [shared-eventing commit](https://github.ibm.com/genctl/shared-eventing/commit/62587abf0bc94f769f7ab6143c513e22a0720b22).
- Updated RMC entry to include a billing plan for cluster networks.
  ![BSS Billing Plan](./assets/BSS-billing-plan.png)
- [BSS-onboarding product schema commit for cluster network type](https://github.ibm.com/cloudlab/vpc-bss-onboarding/commit/8ddc8c44e276fc8cb3231f7ee4326ba25de08991)
- [BSS-onboarding product schema commit for h100 instance profile](https://github.ibm.com/cloudlab/vpc-bss-onboarding/commit/210be1146e6f754fa13dcb72133333a07186a178)

###### Resource Management Console (RMC)

The custer network service is registered under the composite Infrastructure Services (`is`) entry as
`is.cluster-network`.

![RMC Cluster Network](./assets/rmc-is-cluster-network.png)

###### Global Catalog onboarding

The cluster network service is onboarded to global catalog to meet the requirements for BSS
onboarding. Details on the global catalog integration can be found in
[section 3.1.1.4.2](#31142-global-catalog-design-for-clusters).

![Global catalog listing](./assets/global-catalog-listing.png)

##### Quotas

Quotas will be placed on cluster networks and its sub-resources. These limits are intended to
prevent malicious and/or runaway client programs from disproportionately consuming the undercloud
resources required for cluster networks.

| New Quotas                                  | Soft Limit | Hard Limit | Rationale                                                                                                                                                  |
| ------------------------------------------- | ---------- | ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Cluster networks per account per Region     | 10         | 200        | Recommended 1 cluster network per "AI Training infrastructure". Early stakeholder interviews suggests no more than 1 or 2 cluster networks per account.    |
| Cluster network subnets per cluster network | 32         | 64         | Reference topologies need up to 32 subnets per cluster network for 4x network interface multiplexing. We will reassess as new architectures are developed. |
| Cluster network reserved IPs per account    | 5k         | 20k        | These are constrained by how many CRDs can exist in etcd. We follow convention for VPC subnet reserved IPs.                                                |

Just as with virtual network interfaces, we will not limit cluster network interfaces as they will
be constrained by the total number of cluster network subnet reserved IPs.

The quota for cluster network reserved IPs is lower than that of standard reserved IPs.  It was set
to 5k as a soft limit because that should support most known users out of box.  But it will prevent
abuse scenarios that could be expected.

#### Fraud and Abuse

Both the Instance and Cluster network services uses the Resource Controller service broker for VPC
that is owned by the VPC platform team. The service broker currently supports the enable/disable and
delete functionality for instances as they are the primary VPC resource that would be in violation
of fraud and abuse policies.

#### IAM, GhoST, and ReX

The cluster network type/service will be onboarded into IAM, GhoST, and ReX.

Details can be found in the corresponding sections of this SRB:

- IAM integration - [section 3.2.1.5](#3215-identity-and-access-management)
- GhoST & ReX integration - [section 3.1.3.2](#3132-global-search-and-tagging-ghost)

Evidence of onboarding can be found here in the
[IAM documentation for cluster networks](https://test.cloud.ibm.com/docs/account?topic=account-iam-service-roles-actions#is.cluster-network-roles)
that is populated as part of publishing IAM actions through RMC.

### 3.2.5 Compliance

See [FS Cloud Compliance](#3211-fs-cloud-considerations). Use of the cluster network will not be FS
Cloud compliant.

New hardware will all utilized signed firmware. The
[IBM Cloud Hardware Specifications](https://github.ibm.com/cloudlab/platform-inventory/tree/master/specifications/)
will be used to ensure only compliant firmware and configuration is deployed to the servers.

### 3.2.6 Faults and Disasters

#### 3.2.6.1 Fault Tolerance

Generally, if a GPU fails in the HGX board supporting the H100 drives, the system will be taken out
of commission and serviced. IBM will work a contract with our server vendor to ensure that we have
ample replacement GPUs in house and can diagnose on site.

This is a change from the initial Vela design where when a GPU failed, the whole HGX board (with all
8 GPUs) had to be sent back to the vendor in its entirety. The model we will move to will be similar
to replacing a CPU or memory DIMM on a vendors motherboard.

With our experience on Vela, we will assume a fairly significant initial fall out rate to occur with
the systems. As well as a few server failures per month thereafter.

Our existing Infrastructure Automation work will also stress tests for several hours the components
before being allowed to be added into the fleet.

#### 3.2.6.2 Business Continuity Disaster Recovery (BCDR)

The servers introduce no new fundamental components. The existing BCDR policies will be in place and
will be used natively for this environment.

GPUs and the instance storage are ephemeral. The primary data for these nodes is the block volumes,
which are stored on the common NetApp / Acadia backend.

The offering is a single zone offering for latency requirements. Users will be notified of this
limitation in the customer facing documentation.

### 3.2.7 Hardware Platform Support

Reference the [deployments](#261-deployments) for the individual costs of each deployment.

Clustering is only available in certain environments. At a high level, the following visual helps
demonstrate the scaling costs of the network.

![GPU Network Costs](./assets/NetworkCosts.png)

This helps visualize the difference between the fixed and variable costs, and why it's important to
understand a given deployments maximum size (even if it starts smaller).

A new
[Hardware Specification](https://github.ibm.com/cloudlab/platform-inventory/tree/master/specifications/solutions/vsi-hosts)
will be made for this configuration. Both a server and a rack specification. The firmware
configuration will be verified by John Rawlins team, and support will be added to the system
sequencer so that the system automation can bring up the node.

### 3.2.8 Component Level Development

Hardware

- [3.1.1.3 Physical Design](#3113-physical-design)
- [3.1.1.3.1 Server Node Design](#31131-server-node-design)
  - [3.1.1.3.2 Rack Design](#31132-rack-design)

Network Engineering / Backbone

- [3.1.1.3.3 Physical Network Architecture](#31133-physical-network-architecture)
  - [3.1.1.3.4 Network Latency](#31134-network-latency)

Network Engineering / SDN / Compute

- [3.1.1.4 Cluster Design](#3114-overall-cluster-design)
  - [3.1.1.4.1 Customer Pricing](#31141-customer-pricing)
  - [3.1.1.4.2 Global Catalog Design](#31142-global-catalog-design-for-clusters)
  - [3.1.1.4.3 Cluster API Design](#31143-cluster-api-design)
  - [3.1.1.4.4 Creation / Scheduling](#31144-creation--scheduling)

Fleetman / Operations / Platform Integration

- [3.1.1.5 Deployment Architecture](#3115-deployment-architecture)
  - [3.1.1.5.1 M-Zone Yaml Updates](#31151-m-zone-yaml-updates)
  - [3.1.1.5.2 Deployment File Updates](#31152-deployment-file-updates)
  - [3.1.1.5.3 Architecture File Updates](#31153-architecture-file-updates)
    - [3.1.1.5.4 System Specification Design](#31154-system-specification-design)
  - [3.1.1.5.5 HostOS Updates](#31155-hostos-updates)
  - [3.1.1.5.6 Kube Define (GravingYard)](#31156-kube-define-gravingyard)
- [3.1.1.6 Vela2 SDN Architecture](#3116-sdn-architecture)
  - [3.1.1.6.1 Network Configuration Updates](#31161-network-configuration-updates)
  - [3.1.1.6.2 Configuration of the VCN on Vela2](#31162-configuration-of-the-vcn-on-vela2)
  - [3.1.1.6.3 Configuration of vNICs for VMs](#31163-configuration-of-vnics-for-vms)

Compute

- [3.1.1.7 Compute Design](#3117-compute-design)
  - [3.1.1.7.1 Hypervisor](#31171-hypervisor)
    - [3.1.1.7.1.1 Hypervisor Overhead](#311711-hypervisor-overhead)
    - [3.1.1.7.1.2 Hypervisor Tagging](#311712-hypervisor-tagging)
    - [3.1.1.7.1.3 Q35 Support](#311713-q35-support)
    - [3.1.1.7.1.4 Bandwidth Pooling for Cluster Networks](#311714-bandwidth-pooling-for-cluster-networks)
  - [3.1.1.7.2 Resource Objects](#31172-resource-objects)
    - [3.1.1.7.2.1 Global Catalog Updates - Instance Profile](#311721-global-catalog-updates---instance-profile)
    - [3.1.1.7.2.2 Regional Instance Profile Structure](#311722-regional-instance-profile-structure)
    - [3.1.1.7.2.3 Additional Objects on the Instance](#311723-additional-objects-on-the-instance)
  - [3.1.1.7.3 VM Scheduler Evolution](#31173-vm-scheduler-evolution)
  - [3.1.1.7.4 H100 Instance Profile Design](#31174-h100-instance-profile-design)
  - [3.1.1.7.5 Compute Restrictions / Limitations](#31175-compute-restrictions--limitations)
  - [3.1.1.7.6 Dedicated Host Support](#31176-dedicated-host-support)
  - [3.1.1.7.7 Auto Scale Support / Instance Templates](#31177-auto-scale-support--instance-templates)

### 3.2.9 Network Topology and Segmentation Impact

See the [Physical Network Architecture](#31133-physical-network-architecture) and
[SDN Architecture](#3116-sdn-architecture) for information about how this design specification
isolates the network traffic on the east/west network.

Segmentation is affected. The Cluster traffic (GPU to GPU) can not route or egress to the super
spine. This is managed via the [cluster definition](#313-customer-facing-apis), which both ensures
that the correct physical NICs are used for provisioning of vNICs but also that the subnets they are
attached to are non-routable. A cluster network does not support flow logs, and therefore can not be
FS Cloud certified.

The cloud network is a full functioning network, capable of supporting a full feature set for the
cloud network. The NICs attached to the cloud network will have all of the SDN capabilities.

# 4. Testing

This project will utilize the existing smoke tests. However, due to the limited quantities of
[development hardware](./deployments/development.md), they will need to be run periodically for
these profiles.

The SDN team must verify that all deployments of new iobricks updates are validated, and do not
disrupt cluster traffic when an update is released. They will also collaborate with the performance
team to identify a baseline performance level that gets validated before each new SW release.

The performance team developed a set of tests that are constantly running in the staging environment
using 2-3 nodes in the staging environment. If the tests identify a failure, then the team evaluates
what configuration or software change introduced the issue. This is outlined in more detail in the
[performance](#322-performance-scalability-and-resource-consumption) of the SRB.

Cluster network QA test plan:

[Cluster Network Testing](http://confluence.swg.usma.ibm.com:8445/display/CATS/H100+Cluster+Vela2)
[Cluster Network API E2E Testing](https://confluence.swg.usma.ibm.com:8445/display/CATS/H100+Cluster+API+E2E+Network+testing)

## 4.1 CI Pipelines - Automation

No changes anticipating in the CI pipelines themselves.

Given the cost of each server, it is prohibitive to put them in every CI environment. As such, SDN
is ensuring that every SDN release will run a series of manual tests for the RoCE networking and
that compatibility testing will be run on the hosts.

Compute is focusing on adding a host to it's blast radius test environment after core development is
completed.

See section 4 for the ongoing staging tests as well.

# 5. Solution Space

## 5.1 Lessons Learned

In Vela, we learned that having an under provisioned network leads to severe degradations of
performance. Additionally, tuning of the hypervisor needs to be very prescriptive for these
workloads (ex. the Q35 work).

This proposal:

- Takes the cluster network concept and makes it customer consumable
- Makes the H100 offering a public offering
- Dramatic improvements to performance over the A100 generation

### 5.1.1 Release Structure

As part of this proposal, the team attempted to consolidate the deliveries of the server and the
cluster into a single SRB. The goal of this was to reduce the overhead of running independent
projects. However, as the project evolved the project split into two Service Frameworks. It made the
SRB brittle because a portion was enabled for the server, then a subsequent for the cluster.

The goal of a SRB is to be a deliverable unit. The SRB mentors met and determined that future SRBs
which have a likelihood of having independent deliveries (ex. Gaudi-3 server and Gaudi-3 cluster)
must be two separate SRBs. With the latter having a dependency on the former.

While deadlines preclude splitting up this proposal, subsequent projects must be split to align with
GA delivery dates.

## 5.2 Alternatives Considered

Independent of the Vela2 project, IBM is building an "on-prem" H100 training cluster with up to 6000
GPUs. This is known as the `BlueVelas` project.

It is expected that the per-GPU performance of Vela2 is close to, or matches that of `BlueVela`.
However there are some differences. First, `BlueVela` is built on Dell hardware. The second is that
the network is delivered via Infiniband instead of RoCE.

### 5.2.1 Server Alternative - Dell Hardware

The BlueVela project utilizes Dell hardware instead of Supermicro. Additionally, they have a
"white-glove" service from Dell which includes management of all of the infrastructure.

The Dell hardware being used was considered for the Vela2 project. It is similar in specification:

| Spec          | Vela2 Design                              | BlueVelas Design                   |
| ------------- | ----------------------------------------- | ---------------------------------- |
| Manufacturer  | SuperMicro                                | Dell                               |
| Model         | SYS-821GE-TNHR                            | PowerEdge XE9680                   |
| PCIe Topology | 4 independent PLX switches                | 4 independent PLX switches         |
| Processor     | 8474C (48 core SPR, 2.1 Ghz) - IBM Custom | 8468 (48 core SPR, 2.1 Ghz)        |
| Memory        | 32x 64 GB DDR5 4400 MHz DDR5 DIMMs        | 32x 64 GB DDR5 4400 MHz DDR5 DIMMs |
| Boot Drive    | N/A                                       | 2x M.2 RAID 480 GB                 |
| NVMe          | 8x 7.68 TB PCIe Gen 4 Micron 7450 Drives  | 8x 3.84 TB PCIe Gen 4 Drives       |
| Network Cards | 9x CX-7 Dual-Port 200 GigE                | 10x CX-7 Single Port 400 GigE      |

Notes:

- BlueVelas is looking to switch to the 8474C processor
- Memory is actually 4800 MHz but due to 2 DIMMs per Channel (2DPC), the platform lowers clock
  speeds to 4400 Mhz

#### 5.2.1.1 Storage Differences

The BlueVelas design has boot drives, but that is not applicable for the cloud design. The cloud
design performs a network boot, and therefore does not need the added cost of the RAID card.

Additionally, the cloud design has switched to the 7.68 TB Micron drives. These are sized to be more
competitive with cloud providers across the industry. Both designs utilize read optimized drives.
But if there is a performance advantage switching to write optimized, this can be done dynamically
with specific `nvme` commands that can be integrated at a later time.

#### 5.2.1.2 Networking Differences

This is where a more significant difference exists between the Vela2 design and BlueVelas. The
BlueVela design has 3 distinct physical networks:

- GPU Data Network (8x cards, connected to PLX switch - IB)
- Storage Network (1x cards)
- Management Network (1x cards)

In the Vela2 design, this is being consolidated down to a single set of CX-7 networks, which will
logically share the physical network. This is consistent with the original Vela design.

Traffic patterns to date have generally been such that the GPUs "micro stampede" traffic occurs at
different phases in the training life cycle than data inject or check pointing. Additionally the
management and other storage traffic have not been significant enough to separate out the networks.

Additionally, the BlueVelas design utilizes a single 400 GigE port for IB traffic. The benefit is
that a single GPU can send a flow at the full 400 GigE. However a drawback to this is that a single
port failure can render the whole card unusable. This may be acceptable for on prem, but when in a
cloud the rolling updates can not require that we coordinate across different tenants. We also build
with redundancy.

The dual 200-GigE ports allow for each card to achieve 400 GigE over a single vNIC, but an
individual flow will only be able to hit 200 GigE. This does put Vela2 at a disadvantage to
BlueVelas. But this was deemed important from a resiliency factor for the cloud offering. We do not
believe others in the industry are offering single port solutions.

### 5.2.2 Network Alternatives

During the design, the team spent a lot of time considering introducing a different network design
or extending the existing Vela network design. The design alternatives are listed here.

#### 5.2.2.1 Infiniband

At a high level, the team considered introducing a native InfiniBand network for the GPU training
network. In essence, a model similar to the following:

![Infiniband Alternative](./assets/IB-Alternative.png)

The intent of this design would be to keep the GPU side network similar to BlueVelas, but then
utilize a NIC like the Pensando Elba (as part of the Elba VSI project) to connect to the cloud
network. There are a few challenges with this approach:

1. The Vela2 design must support multi-tenancy. While Infiniband can support this, no one on the
   team has any experience in how to effectively support multiple tenants on IB.
2. There is no existing IB ecosystem or experience in the team
3. A new team (estimated at least 5 individuals) would be needed to support this effort - both dev
   and SRE
4. Concerns about ability to deliver to time frame
5. This network would be wholly independent of the broader cloud. No GPU Direct to anything outside
   of this model.

The primary concern here is item 2. Too much unknown, and therefore risk. While BlueVelas can accept
this as a single tenant model, there are cloud security concerns with a new channel in the cloud.

Additionally, the Vela network RDMA solution has proved to be quite effective with the 3 node
configuration. Competitive to the point of offering equivalent or better overall training
performance.

#### 5.2.2.2 RoCE Design with Different Vendor ToR Switches

In the existing Vela design, there are issues with "Fan In". The [RoCE design](../3856/README.md)
utilizes ECN as a mechanism to indicate to the senders of traffic that a port is congested. Upon
receiving the ECN packet, the senders are to slow down sending data.

The typical traffic pattern in Vela is:

```shell
NIC Port -> ToR1 Compute Port -> ToR1 Agg Port -> Agg -> ToR2 Agg Port -> ToR2 Compute Port -> NIC
```

The congestion can occur when multiple senders are sending to the destination port. When that
congestion occurs, this is considered a `fan in` situation. Additionally, the AI traffic also sends
micro bursts of data. Information on these traffic types is
[here](https://arista.my.site.com/AristaCommunity/s/article/how-to-troubleshoot-congestion#Comm_Kna_ka08C0000008TxHQAU_25).

During the time that the congestion occurs, data continues to flow through to the NIC. That NICs
port also starts using the `packet buffer` to ensure no packets get dropped while the ECN packet
gets back to the appropriate sources. This can look like the following.

| Step           | NIC Port     | ToR1 Svr. Port | ToR1 Agg Port | Agg                 | ToR2 Agg Port | ToR2 Svr. Port    | NIC Port     |
| -------------- | ------------ | -------------- | ------------- | ------------------- | ------------- | ----------------- | ------------ |
| 1. Data Send   | Sending Data | -->            | -->           | -->                 | -->           | -->               | Receive Data |
| 2. Congestion  | Sending Data | -->            | -->           | -->                 | -->           | Buffer & Send --> | Receive Data |
|                |              |                |               |                     | <-- Send ECN  |                   |              |
| 3. Fwd ECN     | Sending Data | -->            | -->           | -->                 | -->           | Buffer & Send --> | Receive Data |
|                |              |                |               | <-- Rec. & Send ECN |               |                   |              |
| 4. Fwd ECN     | Sending Data | -->            | -->           | -->                 | -->           | Buffer & Send --> | Receive Data |
|                |              | <-- Send ECN   | Rec. ECN      |                     |               |                   |              |
| 5. Receive ECN | Sending Data | -->            | -->           | -->                 | -->           | Buffer & Send --> | Receive Data |
|                | Rec ECN      |                |               |                     |               |                   |              |
| 6. Slow Send   | Slower Data  | -->            | -->           | -->                 | -->           | Buffer & Send --> | Receive Data |
| 7. Normalize   | Slower Data  | -->            | -->           | -->                 | -->           | Send -->          | Receive Data |

While this is a somewhat crude diagram, it's purpose is to show that the ECN packet must flow back
through the network to indicate to the sender to slow down their traffic. However, during that time,
the sender is continuing to send at full speed. That means the buffer in ToR2's port must weather
the storm until the ECN packet is received.

While this may only be a few microseconds, with line rates achieving 400 GigE, this can grow
quickly.

If that buffer overflows, traffic is lost. With RoCE traffic, if data is lost, things slow
dramatically. Without congestion, typically everything is fine. But dropped packets are very
stressful to RoCE traffic.

To put it in perspective, in Vela it was identified that when the 5th server was added to a rack,
the shared buffers on the switch were now smaller on a per port level. This was due to the
`Arista DCS-7050CX3-32S-R` which has a 32 MB _shared_ port buffer. Once the fifth server was active,
the per port buffers could not absorb the traffic.

This is being resolved by tuning the per queue buffer sizes, which till take unused queue buffers
and reallocates them to the appropriate queues. Additional tuning is done as the switch is
configured to start sending the ECN packet when the buffer exceeds a certain threshold on that port
(ex. 500 KB). Tuning and optimization can generally resolve the issue, but it requires a close
consultation and configuration with NetEng, Arista, and Research.

The Vela2 design utilizes `7060DX5-64`. The team was also asked to evaluate the `NVIDIA SN5600`. The
intent is to ensure that the Vela2 switch can support the added buffer demand for 400 GigE AI
networking. Both switches position themselves as capable of supporting
`demanding AI workloads at 400/800 GigE`. Additionally, due to the nature of the fan-in problem, the
team also investigated a deep buffer 400 GigE switch.

| Measurement     | Arista DCS-7050CX3 (Vela) | Requirement  | Arista 7060DX5-64 (Vela2) | NVIDIA SN5600 |
| --------------- | ------------------------- | ------------ | ------------------------- | ------------- |
| Ports           | 32x 100 GigE              | 36x 400 GigE | 64x 400 GigE              | 64x 800 GigE  |
| Latency         | 800 ns                    | <1 μs        | 825 ns                    | "low"         |
| Packet Buffer   | 32MB Dynamic              | >72 MB       | 114 MB Shared             | 128 MB        |
| Per port buffer | 2 MB (16 ports used)      | 2 MB         | 3.17 MB                   | 3.56 MB       |

Remember that the Vela2 design has each ToR switch having 12x 400 GigE uplink ports and 24x 200 GigE
downlink ports.

Note: per port buffer tuning is also a bit of a science, and having additional buffer allows us to
optimize how aggressive the ECN pull back needs to be.

The NVIDIA SN5600 and Arista 7060DX5-64 are very similar in specs. While the NVIDIA SN5600 can
achieve 128x 400 GigE ports, in order to do so it must use break out cables. It also further reduces
the per port buffer dramatically. However, each ToR in the Vela2 design only needs 36 ports, so the
buffer at a per port level will generally be similar between the SN5600 and the 7060DX5-64.

The biggest drawback to the NVIDIA switches are:

- Compatibility
  - IBM Cloud would be concerned about introducing an NVIDIA design that uses Arista as the agg. It
    is generally not done this way.
  - NVIDIA is also pushing IBM Cloud to move to a new Agg design, that uses SN5600's linked to
    simulate an Agg. Essentially stacking them together to act as an Agg. While this doesn't add any
    _layers_ to the design, it does add to the overall switches.
- Experience
  - IBM Cloud does have tremendous experience in operating and maintaining Arista switches.
  - While not unheard of for other switches to be used, it is obviously ideal to align around
    existing automation.
  - However, if deemed unable to achieve the needs of the workload, IBM Cloud will consider
    appropriate alternative designs.

Of additional note, it is understood how the buffers work on the Arista switches. Packet buffer
memory can be reallocated from the uplink ports to the downlink ports (or vice versa), can be moved
around and tuned deeply. It is not yet understood how that buffer tuning works on the SN5600.
However the IBM Research team is doing a significant deployment atop the SN5600 so that tuning
experience is pending.

The IBM team has decided to move forward with the Arista 7060DX5-64 switches. There are additional
technologies that Arista provides like Dynamic Load Balancing which further improve the ability to
reduce congestion and balance across the ports.

### 5.2.3 Logical vs. Physical North/South and East/West NIC design

See [the design alternative with Blue Velas](#5212-networking-differences). That design splits it's
north/south and east/west (GPU) traffic through physical NIC isolation. This is ultimately the
approach that will be used by this design (see [this section](#5233-research-proposed-design)).

#### 5.2.3.1 Original NIC design

The original design called for 8 NICs which are each attached to the GPU. The cluster API would then
allow for us to carve out independent _logical_ networks, however each logical network would sit on
top of the same physical network that the GPU data ran on. The concern from the research team is
that the corresponding storage traffic for AI workloads could degrade the GPU traffic.

Use the following for a visual of the concern:

![Network Bandwidth Orphan](./assets/NetworkBWOrphan.png)

Given the ring nature of the workload, having a bandwidth used on another NIC slows down the
remaining NICs correspondingly. To clarify, this is not just within the server but across the
cluster as all the GPUs are effectively synchronized.

We've seen evidence of this in Vela when a port goes down on the dual port CX-6, the bandwidth
across the cluster correspondingly decreased as well.

#### 5.2.3.2 Alternative Considered

An alternative consideration was to utilize VNIs across the physical NICs, and then create a load
balancer atop the cloud vNICs to balance the traffic across the physical PFs.

![Load Balancing Alternative](./assets/NetworkSpreadWithLA.png)

With this approach, the logical data traffic will spread across the NICs rather than be isolated
within a single NIC. This allows for graceful degradation of performance across the NICs.

This design has a practicality issue with it. In so far as it's not a practical approach to ask the
user to create 8 additional vNICs and hope they spread across the physical NICs appropriately.

#### 5.2.3.3 Research Proposed Design

The DGX Cloud Design spec and Oracle Cloud have separated the north/south and east/west into
different physical NICs (even if they're on the same aggregation / ToR layer). In effect it creates
this pattern:

![Separate NIC](./assets/NetworkSeparateNSEW.png)

The idea being use the Cluster API to logically steer the data/cloud traffic to the separate NIC.
There may actually be a pair of additional NICs to run the VCN on.

The GPU nics would still have SDN running on top of them for multi-tenant isolation.

The draw back to this approach was that it added an additional per-server connection to the TORs in
the racks. That resulted in one of the following:

- A slightly over committed network
- Adding ports between the ToR & Aggregations to under commit the network

Adding the ports to under commit the network will lead to an additional $2.4M cost to the WDC
network alone.

The team has decided to go with adding the north/south NIC, but will over commit the network
slightly to mitigate the cost increases that would result.

### 5.2.4 Single Port NICs versus Dual Port NICs

There is a concern that a Dual Port NIC will not be able to support the speeds to optimally deliver
the performance needs of the cluster. This is rooted based off the reference design architectures
from NVIDIA which all reference the single port NIC as the solution for AI clusters.

After careful consideration, and escalations with NVIDIA, IBM has made the decision that this design
must _maintain a dual port card_ for both the east/west and north/south traffic.

Additional data below.

#### 5.2.4.1 ToR maintenance with single port

When performing an update to the ToR switches, a single port on each physical NIC would experience
an outage. No fail over.

The cloud NIC would be dual port, so there would be redundancy for the VCN, block, file, CoS, server
to server. But the data network would experience a full data failure event.

In discussions with NetEng, the following was noted:

- ToR switch maintenance is expected to occur about twice a year
- We budget about 2 hours
  - 30 minutes per switch for updates
  - 1 hour buffer
  - ToRs are done in parallel - but if two ToRs in a rack they do the switches in a rack
    sequentially
- No deployments to date have been notified for switch maintenance

Given the nature of the workload, customers would lose their data up to their last checkpoint.
Maintenance windows would need to be announced - and complaints from customers would have to
essentially be ignored given sufficient posting. Too many tenants, no major ability to move because
if you move for one, they will all want it moved.

We may need to issue refunds on the infra for ~6 hours during a 2 hour maintenance, since they would
lose their work up to their last checkpoint.

#### 5.2.4.2 ToR maintenance with single port

When performing an update to the ToR switches, a dual port NIC would experience a degradation.

Performance on the NIC would be about 50% of overall throughput. However the duration would only be
about 60 minutes - 30 minutes per ToR set. Additionally the customer would not have a failure which
loses all progress up to the last checkpoint.

Dependent on the checkpoint window, this could be a significant savings.

In Vela, the checkpoint window is generally 3 hours because at scale - there is a fear of losing
servers in between.

#### 5.2.4.3 NIC Failure

The current VPC software stack gracefully degrades on it's dual port NICs when there is a port
failure. If the DAC cable or TOR port fails, all traffic gracefully fails over. An alert is
generated, and the DC Ops team can generally replace as needed.

However, in the case of an AI job when performance degrades it'll slow down the entire bus. IBM
Cloud will need to treat the AI systems as a special case. When a port fails, a notification will
need to be sent to the user identifying which VSI is degraded so that they can cordon it off for
their next run.

In the event of single port failure, the user should still get a message. However internal diagnosis
will be easier to detect as well.

An alert will be provided to the user in the event of a port failure.

### 5.2.5 Bare Metal versus VSI

There was a long standing discussion as to whether or not this solution should be developed on Bare
Metal (Elba + independent data NICs) or VSI.

A bare metal design would be similar to what Oracle delivers. It removes the entire virtualization
layer, allows for end users to do a host level reboot, and has a few other benefits. Effectively it
brings that workload closer to the metal.

Running a bare metal design would allow for the
[research proposed network design](#5233-research-proposed-design). However the GPU network there
would be independently managed. At the time it was even evaluated to potentially be an
[Infiniband](#5221-infiniband) private network.

There were significant network concerns with that. Such as it couldn't extend to broader network,
required significant investment to deliver secure multi-tenancy without SDN, etc..

VSI was chosen as the path forward due to:

- Extending the Vela design pattern, rather than make a new one-off design
  - Benefits within Vela2 will, in many cases, translate back into Vela.
- Integrates with the IBM Cloud ecosystem natively
  - VSI has much better overall PaaS and SaaS integration at the moment
  - Bare Metal does not have native volume support
  - Bare Metal is not supported with ROKS and IKS
- Fits within the existing overarching architectural design
  - Vela demonstrated that we can deliver performance at scale within VSIs
  - Can deliver this solution on a much tighter timeline than a major change

## 5.3 Competitive Offerings

[AWS UltraClusters](https://aws.amazon.com/ec2/ultraclusters/)
[Azure Accelerated Networking](https://learn.microsoft.com/en-us/azure/virtual-network/create-vm-accelerated-networking-cli?tabs=windows)

# 6. Known Issues

- When calling `POST /instances` the `instance.create`, if the request payload includes
  `cluster_network_attachments`, the AT event will exceed the 16k limit and be dropped. The fix can be
  tracked in [RCS-5341](https://jiracloud.swg.usma.ibm.com:8443/browse/RCS-5341). An executive exception
  was provided to move forward with release with the understanding that a fix would follow shortly after
  GA. This will be captured as a known issue in the public documentation for instances and cluster networks.

# 7. Patents

None

# 8. Meta

None

# 9. Glossary

- `physical cluster network`: The physical back end network dedicated to support a deployments. See
  section the [Cluster Network Physical Design](#31133-physical-network-architecture) section for
  more details.
- `cluster network`: The customer facing abstraction for a cluster network.
- `cluster network profile`: The definition of a cluster network. Defined within
  [Global Catalog](#31142-global-catalog-design-for-clusters), and then expressed to the end user
  through the API as a `cluster_network_profile`. Provides a definition to the user that explains
  the performance profile for a given cluster network.

- API objects
  - `cluster_network`: The API verb used to represent the customer facing `cluster network`. See the
    [API design](#313-customer-facing-apis) for more information.
  - `cluster_network_profile`: The object returned by the [API](#313-customer-facing-apis) which
    represents a cluster network profile.

# 10. References

- [AWS UltraClusters](https://aws.amazon.com/ec2/ultraclusters/)
- [Azure Accelerated Networking](https://learn.microsoft.com/en-us/azure/virtual-network/create-vm-accelerated-networking-cli?tabs=windows)
- [Arista - How to troubleshoot congestion](https://arista.my.site.com/AristaCommunity/s/article/how-to-troubleshoot-congestion#Comm_Kna_ka08C0000008TxHQAU_25)
- [Host Hardware Specification](https://github.ibm.com/cloudlab/platform-inventory/tree/master/specifications/solutions/vsi-hosts)
- [IBM Cloud Hardware Specifications](https://github.ibm.com/cloudlab/platform-inventory/tree/master/specifications/)
- [Cluster Network Testing](http://confluence.swg.usma.ibm.com:8445/display/CATS/H100+Cluster+Vela2)
- [Cluster Network API E2E Testing](https://confluence.swg.usma.ibm.com:8445/display/CATS/H100+Cluster+API+E2E+Network+testing)
- [NVMe disk qualifier](https://github.com/IBM/nvme-disk-qualifier/)
- [Accelerated Compute Section](https://cloud.ibm.com/docs/vpc?topic=vpc-about-advanced-virtual-servers&interface=ui)
- [Cluster Network Documentation](https://cloud.ibm.com/docs/vpc?topic=vpc-about-networking-for-vpc&interface=ui)
- [Instance Profiles](https://cloud.ibm.com/docs/vpc?topic=vpc-profiles)
- [VM Scheduler evolution](https://confluence.swg.usma.ibm.com:8445/display/CTL/Cluster+Scheduling)
- [BSS Onboarded Production Cluster Profiles](https://github.ibm.com/cloudlab/vpc-bss-onboarding/tree/master/resources/is.cluster-network/product_schema/prod)
- [Aha VSIVPC-690](https://bigblue.aha.io/features/VSIVPC-690)
- [Aha CNWVPC-495](https://bigblue.aha.io/features/CNWVPC-495)
- [DCGMI -r 4](https://docs.nvidia.com/datacenter/dcgm/latest/release-notes/changelog.html)
- [Single Node HPL](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/hpc-benchmarks)
- [MPI Cluster Kit](https://docs.nvidia.com/networking/display/hpcxv216/clusterkit)
- [NCCL Functional Test](https://github.com/NVIDIA/nccl-tests)
- [Multi Node HPL Functional Test](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/hpc-benchmarks)
- [Server Specification](https://github.ibm.com/cloudlab/platform-inventory/blob/master/specifications/solutions/vsi-hosts/gx3d-h100-smc.yaml)
- [Rack Specification](https://github.ibm.com/cloudlab/platform-inventory/blob/master/specifications/racks/compute_h100_s12_s42_7010tx-48_pdu_down.yaml)
- [Mellanox NIC](https://github.ibm.com/cloudlab/platform-inventory/blob/master/specifications/components/mellanox.yaml)
- [NVIDIA GPU](https://github.ibm.com/cloudlab/platform-inventory/blob/master/specifications/components/nvidia.yaml)
- [Supermicro Server Chassis](https://github.ibm.com/cloudlab/platform-inventory/blob/master/specifications/components/supermicro.yaml)
- [Micron 7450 NVMe Drives](https://github.ibm.com/cloudlab/platform-inventory/blob/master/specifications/components/micron.yaml)
- [NVIDIA H100](https://www.nvidia.com/en-us/data-center/h100/)
- [NVIDIA A100](https://www.nvidia.com/en-us/data-center/a100/)
- [AI models](https://www.ibm.com/artificial-intelligence)
- [VSIVPC-635 - 8 H100 per node based GPU Compute Profiles](https://bigblue.aha.io/features/VSIVPC-635)
- [CNWVPC-504 - Cluster API Support](https://bigblue.aha.io/features/CNWVPC-504)
- [Compute Req - VPCREQ-412](https://jiracloud.swg.usma.ibm.com:8443/browse/VPCREQ-412)
- [Platform Req - VPCREQ-415](https://jiracloud.swg.usma.ibm.com:8443/browse/VPCREQ-415)
- [PMCOM-5298 - H100 Server Enablement](https://jiracloud.swg.usma.ibm.com:8443/browse/PMCOM-5298)
- [CBLM SRB](../3488/README.md)

<!-- Emacs formatting setup -->
<!-- Local Variables: -->
<!-- fill-column: 100 -->
<!-- End: -->

<!-- Vim formatting setup. Must be in first or last 5 lines by default -->
<!-- vim: set textwidth=100 : -->

- [SRB Proposal Template](#srb-proposal-template)
- [1. Introduction](#1-introduction)
  - [1.1 Project Name](#11-project-name)
  - [1.2 Submitter and Contributors](#12-submitter-and-contributors)
  - [1.3 Mentor(s)](#13-mentors)
  - [1.4 Offering Manager(s)](#14-offering-managers)
  - [1.5 Review Flow (determined by mentor)](#15-review-flow-determined-by-mentor)
  - [1.6 Production Target Date](#16-production-target-date)
- [2. Project Overview](#2-project-overview)
  - [2.1 Overview](#21-overview)
  - [2.2 Consumer Interaction Model](#22-consumer-interaction-model)
  - [2.3 Requirements and Use Cases](#23-requirements-and-use-cases)
    - [2.3.1 Cluster Use Cases](#231-cluster-use-cases)
    - [2.3.2 Research Acceptance Criteria](#232-research-acceptance-criteria)
    - [2.3.3 Beta and GA phases](#233-beta-and-ga-phases)
  - [2.4 Out-of-Scope](#24-out-of-scope)
    - [Out of Scope - Compute](#out-of-scope---compute)
    - [Out of Scope - SDN](#out-of-scope---sdn)
  - [2.5 Dependencies, Dependents, and Incompatibilities](#25-dependencies-dependents-and-incompatibilities)
    - [2.5.1 Other VPC Projects Under Development](#251-other-vpc-projects-under-development)
    - [2.5.2 Other IBM Projects Under Development](#252-other-ibm-projects-under-development)
    - [2.5.3 Third-party](#253-third-party)
    - [2.5.4 External and/or Circular](#254-external-andor-circular)
    - [2.5.5 Brittle or Unusual](#255-brittle-or-unusual)
    - [2.5.6 Incompatibilities](#256-incompatibilities)
    - [2.5.7 Databases](#257-databases)
  - [2.6 Integration, Deployment, and Operations Considerations](#26-integration-deployment-and-operations-considerations)
    - [2.6.1 Deployments](#261-deployments)
  - [2.7 Big Rules](#27-big-rules)
    - [2.7.1 Consistent Customer Experience](#271-consistent-customer-experience)
      - [2.7.1.2 Allow-lists](#2712-allow-lists)
    - [2.7.2 VPC Big Rules](#272-vpc-big-rules)
    - [2.7.3 Cloud Security Baseline Requirements](#273-cloud-security-baseline-requirements)
  - [2.8 Assumptions, Risks, and Constraints](#28-assumptions-risks-and-constraints)
- [3. Architecture, Interfaces, and Impact](#3-architecture-interfaces-and-impact)
  - [3.1 Architecture and Interfaces](#31-architecture-and-interfaces)
    - [3.1.1 Architecture and Technical Design](#311-architecture-and-technical-design)
      - [3.1.1.1 Workload Deployments](#3111-workload-deployments)
      - [3.1.1.2 High Level Concepts for Vela2](#3112-high-level-concepts-for-vela2)
        - [3.1.1.2.1 Cluster Networks](#31121-cluster-networks)
        - [3.1.1.2.2 Physical Mapping of Network in Node](#31122-physical-mapping-of-network-in-node)
        - [3.1.1.2.3 Cluster Network to Physical Cluster Implementation Mapping](#31123-cluster-network-to-physical-cluster-implementation-mapping)
        - [3.1.1.2.4 Limited Zonal Availability](#31124-limited-zonal-availability)
        - [3.1.1.2.5 Zonelet Integration](#31125-zonelet-integration)
      - [3.1.1.3 Physical Design](#3113-physical-design)
      - [3.1.1.3.1 Server Node Design](#31131-server-node-design)
        - [3.1.1.3.2 Rack Design](#31132-rack-design)
      - [3.1.1.3.3 Physical Network Architecture](#31133-physical-network-architecture)
        - [3.1.1.3.3.1 VRF and Logical Rails](#311331-vrf-and-logical-rails)
      - [3.1.1.3.4 Network Latency](#31134-network-latency)
      - [3.1.1.4 Overall Cluster Design](#3114-overall-cluster-design)
        - [3.1.1.4.1 Customer Pricing](#31141-customer-pricing)
        - [3.1.1.4.2 Global Catalog Design for Clusters](#31142-global-catalog-design-for-clusters)
        - [3.1.1.4.3 Cluster API Design](#31143-cluster-api-design)
        - [3.1.1.4.4 Creation / Scheduling](#31144-creation--scheduling)
      - [3.1.1.5 Deployment Architecture](#3115-deployment-architecture)
        - [3.1.1.5.1 M-Zone Yaml Updates](#31151-m-zone-yaml-updates)
        - [3.1.1.5.2 Deployment File Updates](#31152-deployment-file-updates)
        - [3.1.1.5.3 Architecture File Updates](#31153-architecture-file-updates)
          - [3.1.1.5.4 System Specification Design](#31154-system-specification-design)
        - [3.1.1.5.5 HostOS Updates](#31155-hostos-updates)
        - [3.1.1.5.6 Kube Define (GravingYard)](#31156-kube-define-gravingyard)
      - [3.1.1.6 SDN Architecture](#3116-sdn-architecture)
        - [3.1.1.6.1 Network Configuration Updates](#31161-network-configuration-updates)
        - [3.1.1.6.2 Configuration of the VCN on Vela2](#31162-configuration-of-the-vcn-on-vela2)
        - [3.1.1.6.3 Configuration of vNICs for VMs](#31163-configuration-of-vnics-for-vms)
      - [3.1.1.7 Compute Design](#3117-compute-design)
        - [3.1.1.7.1 Hypervisor](#31171-hypervisor)
          - [3.1.1.7.1.1 Hypervisor Overhead](#311711-hypervisor-overhead)
          - [3.1.1.7.1.2 Hypervisor Tagging](#311712-hypervisor-tagging)
          - [3.1.1.7.1.3 Q35 Support](#311713-q35-support)
          - [3.1.1.7.1.4 Bandwidth Pooling for Cluster Networks](#311714-bandwidth-pooling-for-cluster-networks)
          - [3.1.1.7.1.5 VSI Resize](#311715-vsi-resize)
        - [3.1.1.7.2 Resource Objects](#31172-resource-objects)
          - [3.1.1.7.2.1 Global Catalog Updates - Instance Profile](#311721-global-catalog-updates---instance-profile)
          - [3.1.1.7.2.2 Regional Instance Profile Structure](#311722-regional-instance-profile-structure)
          - [3.1.1.7.2.3 Additional Objects on the Instance](#311723-additional-objects-on-the-instance)
        - [3.1.1.7.3 VM Scheduler Evolution](#31173-vm-scheduler-evolution)
        - [3.1.1.7.4 H100 Instance Profile Design](#31174-h100-instance-profile-design)
        - [3.1.1.7.5 Compute Restrictions / Limitations](#31175-compute-restrictions--limitations)
        - [3.1.1.7.6 Dedicated Host Support](#31176-dedicated-host-support)
        - [3.1.1.7.7 Auto Scale Support / Instance Templates](#31177-auto-scale-support--instance-templates)
    - [3.1.2 Customer Experience](#312-customer-experience)
      - [3.1.2.1 User Interface (UI)](#3121-user-interface-ui)
      - [3.1.2.2 Command Line Interface (CLI)](#3122-command-line-interface-cli)
      - [3.1.2.3 Terraform and Packer](#3123-terraform-and-packer)
        - [`ibm_is_cluster_network_profile`](#ibm_is_cluster_network_profile)
        - [`ibm_is_cluster_network`](#ibm_is_cluster_network)
        - [`ibm_is_cluster_network_subnet`](#ibm_is_cluster_network_subnet)
        - [`ibm_is_cluster_network_subnet_reserved_ips`](#ibm_is_cluster_network_subnet_reserved_ips)
        - [`ibm_is_cluster_network_interface`](#ibm_is_cluster_network_interface)
        - [`ibm_is_instance_cluster_network_attachments`](#ibm_is_instance_cluster_network_attachments)
      - [3.1.2.4 Customer-facing Documentation](#3124-customer-facing-documentation)
    - [3.1.3 Customer-facing APIs](#313-customer-facing-apis)
      - [Updated Public Operations](#updated-public-operations)
      - [New Metadata Operations](#new-metadata-operations)
      - [Updated Metadata Operations](#updated-metadata-operations)
      - [New customer-facing concepts](#new-customer-facing-concepts)
      - [3.1.3.1 Activity Tracker (AT)](#3131-activity-tracker-at)
      - [AT Changes: public](#at-changes-public)
      - [AT Changes: internal](#at-changes-internal)
      - [AT Changes: metadata](#at-changes-metadata)
      - [3.1.3.2 Global Search and Tagging (GhoST)](#3132-global-search-and-tagging-ghost)
      - [3.1.3.3 IBM Cloud Compliance and Security Center (IBM SCC)](#3133-ibm-cloud-compliance-and-security-center-ibm-scc)
    - [3.1.4 Internal APIs and Schemas](#314-internal-apis-and-schemas)
      - [New Internal Operations](#new-internal-operations)
      - [Updated Internal Operations](#updated-internal-operations)
      - [Updated Partner Operations](#updated-partner-operations)
    - [3.1.5 Deployment Architecture](#315-deployment-architecture)
  - [3.2 Architectural Considerations and Impact](#32-architectural-considerations-and-impact)
    - [3.2.1 Security](#321-security)
      - [3.2.1.1 FS Cloud Considerations](#3211-fs-cloud-considerations)
      - [3.2.1.2 Secrets](#3212-secrets)
      - [3.2.1.3 Control Plane Hardening](#3213-control-plane-hardening)
      - [3.2.1.4 Significant Change Considerations](#3214-significant-change-considerations)
      - [3.2.1.5 Identity and Access Management](#3215-identity-and-access-management)
      - [IAM Changes](#iam-changes)
      - [IAM Changes: public](#iam-changes-public)
      - [IAM Changes: internal](#iam-changes-internal)
    - [3.2.2 Performance, Scalability, and Resource Consumption](#322-performance-scalability-and-resource-consumption)
      - [3.2.2.1 Performance Test](#3221-performance-test)
      - [3.2.2.2 Always-On Staging Test](#3222-always-on-staging-test)
    - [3.2.3 Monitoring, Diagnostics, and Troubleshooting](#323-monitoring-diagnostics-and-troubleshooting)
    - [3.2.4 Lifecycle, Metering, and Billing](#324-lifecycle-metering-and-billing)
      - [Cluster Network Lifecycle States](#cluster-network-lifecycle-states)
        - [Cluster Network Future Lifecycle States](#cluster-network-future-lifecycle-states)
      - [Cluster Network Lifecycle Transitions](#cluster-network-lifecycle-transitions)
        - [Cluster Network Future Lifecycle Transitions](#cluster-network-future-lifecycle-transitions)
      - [Other New Resources](#other-new-resources)
      - [Billing/Metering](#billingmetering)
        - [BSS-Integration Onboarding](#bss-integration-onboarding)
          - [Resource Management Console (RMC)](#resource-management-console-rmc)
          - [Global Catalog onboarding](#global-catalog-onboarding)
        - [Quotas](#quotas)
      - [Fraud and Abuse](#fraud-and-abuse)
      - [IAM, GhoST, and ReX](#iam-ghost-and-rex)
    - [3.2.5 Compliance](#325-compliance)
    - [3.2.6 Faults and Disasters](#326-faults-and-disasters)
      - [3.2.6.1 Fault Tolerance](#3261-fault-tolerance)
      - [3.2.6.2 Business Continuity Disaster Recovery (BCDR)](#3262-business-continuity-disaster-recovery-bcdr)
    - [3.2.7 Hardware Platform Support](#327-hardware-platform-support)
    - [3.2.8 Component Level Development](#328-component-level-development)
    - [3.2.9 Network Topology and Segmentation Impact](#329-network-topology-and-segmentation-impact)
- [4. Testing](#4-testing)
  - [4.1 CI Pipelines - Automation](#41-ci-pipelines---automation)
- [5. Solution Space](#5-solution-space)
  - [5.1 Lessons Learned](#51-lessons-learned)
    - [5.1.1 Release Structure](#511-release-structure)
  - [5.2 Alternatives Considered](#52-alternatives-considered)
    - [5.2.1 Server Alternative - Dell Hardware](#521-server-alternative---dell-hardware)
      - [5.2.1.1 Storage Differences](#5211-storage-differences)
      - [5.2.1.2 Networking Differences](#5212-networking-differences)
    - [5.2.2 Network Alternatives](#522-network-alternatives)
      - [5.2.2.1 Infiniband](#5221-infiniband)
      - [5.2.2.2 RoCE Design with Different Vendor ToR Switches](#5222-roce-design-with-different-vendor-tor-switches)
    - [5.2.3 Logical vs. Physical North/South and East/West NIC design](#523-logical-vs-physical-northsouth-and-eastwest-nic-design)
      - [5.2.3.1 Original NIC design](#5231-original-nic-design)
      - [5.2.3.2 Alternative Considered](#5232-alternative-considered)
      - [5.2.3.3 Research Proposed Design](#5233-research-proposed-design)
    - [5.2.4 Single Port NICs versus Dual Port NICs](#524-single-port-nics-versus-dual-port-nics)
      - [5.2.4.1 ToR maintenance with single port](#5241-tor-maintenance-with-single-port)
      - [5.2.4.2 ToR maintenance with single port](#5242-tor-maintenance-with-single-port)
      - [5.2.4.3 NIC Failure](#5243-nic-failure)
    - [5.2.5 Bare Metal versus VSI](#525-bare-metal-versus-vsi)
  - [5.3 Competitive Offerings](#53-competitive-offerings)
- [6. Known Issues](#6-known-issues)
- [7. Patents](#7-patents)
- [8. Meta](#8-meta)
- [9. Glossary](#9-glossary)
- [10. References](#10-references)
