---
template_version: 2021-04-22

state: closed
status_label: in-production
last_update: 2021-10-18
---

# GPU Integration into VPC Gen 2 for x86

# 1. Introduction

## 1.1 Project Name

GPU Integration into VPC Gen 2 for x86 based Instances

[Aha Reference](https://bigblue.aha.io/features/GNVSI-7)

## 1.2 Submitter and Contributors



## 1.3 Mentor(s)


## 1.4 Offering Manager(s)



## 1.5 Review Flow (determined by mentor)

Full

## 1.6 Production Target Date

2Q2021

# 2. Project Overview

## 2.1 Overview

GPUs were originally introduced as part of the Power in VPC integration. The integration was done in
such a way that supported Power and x86 based VSIs, however only the Power VSIs were originally
brought to market. Given that the Power support was withdrawn from market, the GPU solution never
fully GA'd. This SRB replaces the
[original GPU SRB](https://github.ibm.com/cloudlab/srb/tree/master/proposals/356).

The attachments of GPUs to VSIs will be via a KVM passthrough. This allows for near native speed
within the VSIs. Virtualization of this link would significantly hamper the communication speeds and
take away differentiation.

This offering will provide new GPU Profiles that include a set of GPUs to be attached to a VSI. The
family of profiles will be the `gx2` family.

Note that the GPU attachment to the VSIs prevents live migration. Direct
attach devices can not support live migration. The alternatives section elaborates why vGPU is not a
viable option.

## 2.2 Consumer Interaction Model

The customer will be exposed to a series of new VSI profiles. The new VSI profiles specify the same
characteristics that are expected in existing profiles - vCPU, memory, etc... However, these
profiles will also include a type of GPU, number of GPUs and network speeds.

The following GPU profiles will exist for customers and target both Skylake and Cascade Lake Nodes:

| Instance Profile Name | vCPU | Memory | GPU                        | Network Speed | Storage |
| --------------------- | ---- | ------ | -------------------------- | ------------- | ------- |
| gx2-8x64x1v100        | 8    | 64     | (1) NVIDIA 16 GB PCIe v100 | 16 Gb/s       | N/A     |
| gx2-16x128x1v100      | 16   | 128    | (1) NVIDIA 16 GB PCIe v100 | 32 Gb/s       | N/A     |
| gx2-16x128x2v100      | 16   | 128    | (2) NVIDIA 16 GB PCIe v100 | 32 Gb/s       | N/A     |
| gx2-32x256x2v100      | 32   | 256    | (2) NVIDIA 16 GB PCIe v100 | 64 Gb/s       | N/A     |

With these profiles, customers will be able to choose any image that they wish. The profile will
not limit the selection of the image.

There will also be an additional profile provided in WDC07 for Research only shortly after the
v100 GPU GA. This will be our first attempt to provide the A100 GPUs and the design and flow
will work the same as for the V100 GPUs, so is being mentioned in this document also. There will
be in the future an additional GA with A100 GPU profiles for external customers, but this initial
profile will only have the research accounts in the allow list.

| Instance Profile Name      | vCPU | Memory | GPU                        | Network Speed | Storage|
| -------------------------- | ---- | ------ | -------------------------- | ------------- | -------|
| gx2-80x1280x8a100-internal | 80   | 1280   | (8) NVIDIA 80 GB SXM4 A100 | 200 Gb/s      | 4x3200 |

There will be no dedicated host profiles at this time, rationale describe in a later section.

## 2.3 Requirements and Use Cases

- GPUs must be attached to a VSI at the time of the creation of the VSI. They can not be modified
  after VSI creation.
- The VSI profiles must have metadata associated with them to describe the types of accelerators
  that are attached (what brand, quantity, memory size in gibibytes and model of GPU).
- GPUs are a part that need to be monitored within the system. Failure of this part should trigger
  the same flows as other hardware failures (ex. memory or NIC failure).
  - This leverages the HostDevice model introduced in the
    [Instance Storage SRB](https://github.ibm.com/cloudlab/srb/tree/master/proposals/787).
- The GPU must be attached to the same socket as the vCPUs and Memory Pinning to maximize overall
  bandwidth.
  - All pinning is transparent to the user.
  - Pinning in general reduces the overhead of any processors socket to socket bus.
  - If the user purchases multiple GPUs, and they are spread across sockets:
    - It must be an even distribution of GPUs across the sockets
    - It must have an equal amount of vCPUs and Memory attached across the sockets
- GPU memory must be physically cleared off the cards after a VSI is destroyed or shutdown.
  - Auditable evidence must be produced via Activity Tracker that the GPU's memory was wiped on VSI
    destroy or shutdown.
- In order to make the Tesla V100 GPUs financially viable, the excess capacity on a node not used by
  the GPU profiles must be sellable for non-GPU profiles.
  - The scheduler will reserve enough cpu, memory, network capacity such that GPU profiles can fully
    take advantage of both GPUs on the node. The scheduler will allow the rest of the capacity to be
    filled by non-GPU profiles.

## 2.4 Out-of-Scope

- VSIs with GPUs will not be able to live migrate.
  - This is due to the nature of passthrough
  - There are interesting technologies like vGPU that may solve this at some point. However, that
    solution does not work completely, may not be certified for use with all GPU software solutions,
    and has additional complexity.
  - A future SRB proposal will be needed to add in GPU Live Migration support, however it will
    always be a limitation for supporting the latest/greatest GPUs at _native_ speed.
- Images with the GPU drivers pre-installed will not be available at GA.
  - Licensing prevents us from including the NVIDIA drivers in the image without presenting license.
  - A dependency is placed on the
    [007 project](https://bigblue.aha.io/products/PUBCLOUD/strategic_imperatives/PUBCLOUD-G-44) for
    the mechanism to get 3rd party images that will contain the GPU drivers.
  - The 3rd party images with GPU drivers will be delivered after GA.
  - Customers will currently need to install the drivers on the VSI, or custom image, themselves.
- Dedicated Host profiles will not be provided at GA.
  - Offering has determined that in order to make GPUs financially viable, we must sell the excess
    capacity on the system.
  - As such, we will not offer dedicated host profiles for the Tesla V100 GPUs.
  - When a fit-for-purpose GPU node comes along, such as a future A100 GA, we will provide dedicated
    host profiles.

## 2.5 Dependencies, Dependents, and Incompatibilities

### 2.5.1 Other VPC Projects Under Development

Dependencies:

- [Instance Storage](https://github.ibm.com/cloudlab/srb/tree/master/proposals/787)
  - Note: The dependency exists because GPUs will use the DeviceChunk model introduced by Instance
    Storage.

### 2.5.2 Other IBM Projects Under Development

These services are dependent on GPUs being available in the VPC catalog.

- IKS
- Watson

### 2.5.3 Third-party

#### NVIDIA

The customer VMs will need the GPU drivers, provided by NVIDIA, installed on them to make use of the
GPUs. The stock images provided in VPC will not contain these drivers, so for those images the
customers will need to install the drivers themselves.

After GA, 3rd party images will be provided that already have the GPU drivers embedded
in them. These images will be provided through the 007 project support.

For clearing GPU memory, the NVIDIA driver will not be used on the host as it was for Power.
However, a python utility provided by NVIDIA, will be used for clearing the GPU memory through the
host.

### 2.5.4 External and/or Circular

No new circular dependencies. No external service dependencies.

### 2.5.5 Brittle or Unusual

Lack of Live Migration support makes this solution unusual.

### 2.5.6 Incompatibilities

N/A

### 2.5.7 Databases

N/A

## 2.6 Integration, Deployment, and Operations Considerations

The GPUs will show up in the operations dashboard as part of the compute host's inventory. The GPUs
are directly integrated into the Instance Profile, so a customer can not dynamically add or remove
GPUs.

GPU failures will be detected when a user shuts down an instance. A health check will be run as part
of the GPU memory wipe method. If the GPU is unhealthy, the control plane will isolate it and flag
the device as failed. A sysdig event will be raised, and the SREs can then resolve the hardware
issue at their leisure. A quick device check will occur before allocation occurs as well. The
scheduler will also not allow any VSIs to be deployed using a GPU that has had a failure detected.

When a disruptive host failure occurs, lost node remediation will occur. The VSI will be started up
on a new host (unless the customer opts out of lost node remediation).

During disruptive maintenance (host kernel upgrade, new system firmware, etc...) these VMs will
follow the cold migration maintenance path. The customer will be notified that they need to
shutdown/start up their instance within a certain number of days. If they fail to do so before a
given date, the control plane will restart it on their behalf. This is consistent with our
maintenance strategy for anything that does not live migrate.

## 2.7 Big Rules

### 2.7.1 Consistent Customer Experience

GPUs will only be brought forward with the x86 platform, as there is not support for Z at this time.
It is an optional part.

The GPUs may be offered on bare metal after the initial offering comes to market.

GPUs will not be purchasable with Dedicated Hosts for the Tesla V100s. When a fit-for-purpose GPU
node, such as the A100 GA, is introduced, then there will be a consistent solution using dedicated
hosts.

### 2.7.2 VPC Big Rules

1. YES: All components must be able to be upgraded live (without service degredation/outage) and
   without manual intervention
   - NOTE: Host disruptive maintenance will cause a customer outage. That is described in section
     2.6, but is consistent with other solutions in this space.
1. NO: All components must present usage metrics
   - We are unable to obtain usage metrics for passed thru devices. The device chunk maintains state
     on the device, but not usage statistics.
1. YES: All components must use the
   [VPC/NG monitoring framework](https://github.ibm.com/cloudlab/srb/tree/master/architecture/telemetry/monitoring)
1. YES: All components must use existing VPC/NG control planes -- no new control planes
1. YES: All components must be controlled via well-defined APIs
1. YES: All components must provide functional parity across all modalities (UI, CLI, API,
   Terraform)
1. NO: All components must provide functional parity across all targeted platform architectures (x86
   and Z)
   - No Z GPU capability for GPUs
1. YES: All publicly facing UIs and CLIs must use RIAS API to operate on VPC/NG resources
1. YES: All publicly facing APIs must adhere to the
   [API handbook guidelines](https://pages.github.ibm.com/CloudEngineering/api_handbook/)
1. YES: All publicly facing APIs must be the same in all regions (this implies equivalent
   functionality in all regions)
1. YES: All components must perform authorization using the
   [VPC/NG IAM services](architecture/platformIntegration/genesis-iam)
1. YES: All components must be internally stateless and recreate their runtime state from external
   sources
1. YES: Any caching or shadowing of state must never result in an API consumer observing
   inconsistent or incorrect behavior
1. YES: All hardware components must use the
   [VPC/NG diagnosability framework](https://github.ibm.com/cloudlab/srb/tree/master/architecture/telemetry/diagnostics)
   and implement the on-demand diagnostic API
1. YES: All services must present an API for get/set/watch of state
1. YES: All connections between components must be mutually authenticated, and encrypted using
   AES128 or higher

### 2.7.3 Cloud Security Baseline Requirements

Responses to
[baseline requirements](https://pages.github.ibm.com/ibmcloud/Security/guidance/Cloud-Security-Baseline.html).

- Authentication & access control: Y
- Secrets management: Y
- Compute isolation: Y
- Network isolation: N/A
- Encryption at rest & secure deletion:
  - Secure Deletion: Y
  - Encryption at Rest: N/A
- Encryption in transit: N/A (all sensitive data local to server)
- Encrypt to standards: Y
- Logging & non-repudiation: Y
- Logging of credentials: Y
- Injection prevention: Y
- Secure development practices: Y
- Update & secure configuration: Y

## 2.8 Assumptions, Risks, and Constraints

No major assumption, risks or constraints. This was vetted through the Power GPU Beta.

# 3. Architecture, Interfaces, and Impact

## 3.1 Architecture and Interfaces

### 3.1.1 Architecture and Technical Design

#### 3.1.1.1 Hardware Design

The GPU will be directly attached to free PCIe slots in the compute host. In the future this may be
a SXM4 or non-PCI based connector.

![GPU Host](./materials/GPUHost.png)

The Tesla V100 GPU profiles being GA'd will not contain any Instance Storage. However, the A100
profile being provided for research-only will have NVMe disks attached that are passed through to
the VM. The diagram above shows how NVMe based Instance Storage can optionally be used with GPUs.

Note that the following PCIe GPUs are being considered.

| Capability           | T4         | V100          | A100        |
| -------------------- | ---------- | ------------- | ----------- |
| GPU Memory Size      | 16 GB DDR6 | 16/32 GB HBM2 | 40 GB HBM2  |
| GPU Memory Bandwidth | 320 GB/s   | 900 GB/s      | 1,555 GB/s  |
| FP32 TFLOPS          | 8.1        | 14            | 19.5        |
| Wattage              | 70         | 300           | 250         |
| Unique Capabilities  | None       | NVLink        | MIG, NVLink |
| Connection           | PCIe       | PCIe          | SXM4        |

### 3.1.2 Customer Experience

The customer will simply have a new set of VSI profiles which include the GPU parts within them.
There are no new POST or GET APIs. The GET APIs for Instances or InstanceProfiles will return the
GPU information.

The following is a sample Instance Profile in the global catalog:

<!-- cSpell:ignore globalcatalog provisionable unprovisioning updateable -->

```json
{
    "id": "gx2-32x256x2v",
    "catalog_crn": "crn:v1:bluemix:public:globalcatalog::::instance.profile:gx2-32x256x2v",
    "url": "https://globalcatalog.cloud.ibm.com/api/v1/gx2-32x256x2v?complete=true",
    "name": "gx2-32x256x2v",
    "kind": "instance.profile",
    "overview_ui": {
        "en": {
            "display_name": "GPU 32x256x2 V100",
            "description": "GPU 32x256x2 V100",
            "long_description": "GPU Profile, 32 Core, 256GiB, 64Gbps, 2 Tesla V100 GPU"
        }
    },
    "images": {
        "image": "https://resource-catalog.stage1.bluemix.net/static/images/uploaded/profile.png"
    },
    "children_url": "https://globalcatalog.cloud.ibm.com/api/v1/gx2-32x256x2v/%2A",
    "disabled": false,
    "active": true,
    "tags": [
        "ibm_created",
        "is.instance:beta",
        "is.instance:gpu",
        "is.instance:gt"
    ],
    "pricing_tags": null,
    "provider": {
        "email": "dummy@ibm.com",
        "name": "IBM"
    },
    "created": "2020-11-13T14:27:29.354Z",
    "updated": "2020-12-03T16:35:08.529Z",
    "metadata": {
        "rc_compatible": false,
        "service": {
            "type": "",
            "iam_compatible": false,
            "unique_api_key": false,
            "rc_provisionable": false,
            "bindable": false,
            "async_provisioning_supported": false,
            "async_unprovisioning_supported": false,
            "user_defined_service": null,
            "parameters": null,
            "extension": null,
            "plan_updateable": false,
            "state": "",
            "service_check_enabled": false,
            "test_check_interval": 0,
            "service_key_supported": false,
            "custom_create_page_hybrid_enabled": false
        },
        "plan": {
            "bindable": false,
            "reservable": false,
            "allow_internal_users": false,
            "async_provisioning_supported": false,
            "async_unprovisioning_supported": false,
            "test_check_interval": 0,
            "single_scope_instance": "",
            "service_check_enabled": false
        },
        "deployment": {
            "broker": {}
        },
        "alias": {},
        "template": {
            "source": {}
        },
        "ui": {
            "urls": {},
            "end_of_service_time": "0001-01-01T00:00:00Z"
        },
        "extension_point": {
            "requires_approval": false
        },
        "extension": {},
        "pricing": {
            "origin": "global_catalog",
            "starting_price": {},
            "metrics": null
        },
        "sla": {
            "dr": {
                "dr": false
            }
        },
        "callbacks": {},
        "other": {
            "profile": {
                "default_config": {
                    "bandwidth": 64000,
                    "cpu": 32,
                    "cpu_family": "CascadeLake",
                    "gpu_count": 2,
                    "gpu_manufacturer": "NVIDIA",
                    "gpu_memory": 16,
                    "gpu_model": "Tesla V100",
                    "frequency": 2000,
                    "port_speed": 25000,
                    "ram": 256
                },
                "family": "gpu",
                "generation": "gt",
                "resource_type": "instance"
            }
        }
    }
}
```

A non GPU profile will remove the `other:profile:default_config:gpu*` attributes. The family will
also not be `gpu`. The RIAS Instance Profile is noted in Section 3.1.3.

#### 3.1.2.1 User Interface (UI)

The UI has been previously integrated as part of the Power Beta that included GPUs. The UI has the
user pick the Image first. The user sees the high level Operating Systems - ex. Red Hat, CentOS,
Windows, Ubuntu, etc... From within there, the user can choose a specific version. The user will be
able to select a GPU profile for any image that they choose.

A new family will be added to the Instance Profile Selector - `GPU`. No features will be added.

#### 3.1.2.2 Command Line Interface (CLI)

The CLI will be updated so that the Instance Profiles show the GPUs.

![Sample CLI](./materials/cli.png)

#### 3.1.2.3 Terraform Provider

No changes are needed to the terraform provider. These will continue to work, and users that wish to
take advantage of GPUs will need to choose the new profile.

#### 3.1.2.4 Customer-facing Documentation

GPU topics will need to be developed to help the users utilize the new resources. This will be in
the VPC proper documentation.

The main explanation of GPUs is available on the
[Managing GPUs](https://cloud.ibm.com/docs/vpc?topic=vpc-managing-gpus) page and the list of GPU
profiles is available on the
[Instance Profiles](https://cloud.ibm.com/docs/vpc?topic=vpc-profiles) page.

Other minor updates were made to other pages such as Activity Tracker events and Quotas.

### 3.1.3 Customer-facing APIs

The additions to the API were done in
[SRB 356](https://github.ibm.com/cloudlab/srb/blob/a3f93e072d6727665aa77b17c86d6cfef5d9e772/proposals/356/README.md).
However, that was targeting POWER integration. While no new changes are required for the x86
integration, it is important to revisit that since SRB 356 will be removed.

- **/instance/profiles/ (/{name})** (GET only)

  - **gpu_count** - **InstanceProfileGPU**: Quantity of the GPUs
  - **gpu_manufacturer** - **InstanceProfileGPUManufacturer**: Manufacturer of the GPU (ex.
    'nvidia')
  - **gpu_memory** - **InstanceProfileGPUMemory**: Quantity of the overall memory for all of the
    GPUs within this profile (ex. 64). Value is in GiB.
  - **gpu_model** - **InstanceProfileGPUModel**: The model of the GPUs attached to this instance
    (ex. 'Tesla V100')

- **/instance/ (/{id})** (GET only)

  - **InstanceGPU**
    - **count** - **InstanceProfileGPU**: Quantity of the GPUs
    - **manufacturer** - **InstanceProfileGPUManufacturer**: Manufacturer of the GPU (ex. 'nvidia')
    - **memory** - **InstanceProfileGPUMemory**: Quantity of the overall memory for all of the GPUs
      within this profile (ex. 64). Value is in GiB.
    - **model** - **InstanceProfileGPUModel**: The model of the GPUs attached to this instance (ex.
      'Tesla V100')

GPUs are always a child of the instance that they are part of, and have no independent lifecycle.

#### 3.1.3.1 Activity Tracker (AT)

Since the GPUs are defined in the profile and attached at the time the VSI is deployed and cannot
be changed after the VSI is created, there is no need for any additional activity tracker events
to track the lifecycle of the GPU itself.

However, there will be an additional `is.instance.gpu.wipe` event that will provide an audit trail
to the customer that the GPU memory was wiped after they stop or delete the VSI. This event will be
very much like the is.instance.disk.wipe event that was defined for Instance Storage, where the
event will follow the standard payload format and be tied to the VSI that has the GPU attached.

#### 3.1.3.2 Global Search and Tagging (GhoST)

Since the GPU is part of the Instance Profile and the VSI, there is no separate resource that
exists that would need to be added to GhoST. The additional GPU attributes on the Instance will
automatically be picked up by GhoST.

There will be one additional field added to the Resource Explorer (ReX) to represent the GPU
information for the VSI.  This will be very similar to what was done for Instance Storage and
represents a concatenation of the GPU attributes into a single string, such as
`2 x NVIDIA 16 GB Tesla V100`.

### 3.1.4 Internal APIs and Schemas

The communication between RIAS and GenCTL will be well defined. At its core, the GenCTL scheduler
needs to know the type of GPU (model and memory size) as well as the number of GPUs needed.

- [RIAS Compute Instance Spec](https://github.ibm.com/genctl/regional-k8s-types/blob/ab69569ff75ab391a5b5a8e1505e95e9fe0f56e5/apis/compute/v1alpha1/instance_spec.go#L227)
- [GenCTL Compute VM Spec](https://github.ibm.com/genctl/apis/blob/b3c87b8929e2c9c1e390133f51901e31f1d19764/compute/virtualmachine.go#L50)

### 3.1.5 Deployment Architecture

Deployment will be handled through the use of feature flags for addition of the servers to
environments. The feature flags will be a regional roll out.

There will be the following feature flags:

- `vpc-feature-pcie-gpu-v100-cascade`
- `vpc-feature-pcie-gpu-v100-skylake`
- `vpc-feature-pcie-gpu-t4`
- `vpc-feature-pcie-gpu-a100`

Each GPU type will get its own feature flag. This is because each region may have different GPUs
available due to inventory. This allows us to maintain scoping to specific inventory. It also hides
GPUs from the zones that won't have a full load out of every GPU (ex. CI may only have v100's and
not T4's). The T4 is added solely as an example, as offering is making a decision on whether or
not to include the T4 as a PCIe based offering.

Initially, the v100 feature flag will be used. This is due to the inventory available to us. The
A100 feature flag will be used for the A100 GPU serves being enabled in WDC for research-only. As
additional GPUs become available, new feature flags will be added. These three will be added
initially, where the two v100 flags are already enabled in Prod and the A100 will be in the coming
months.

In general, dynamic addition of servers to environments is standardized as part of the NextGen
project overall. This is needed to grow capacity, deal with faults, etc... Accelerated Compute VSIs
servers simply utilize these existing flows.

Also of note, is the geo-tagging that is available in the global catalog. This may be used by the
Offering Managers as a way to say what production regions have particular GPUs available in them.
This however is not granular enough to handle non-production environments and also does not allow
the release management team to have control over which regions the support is deployed to and will
be activated. Because of this, feature flags are still required for every new feature, and the
geo-tagging will be used in addition to feature flags, not as a replacement.

#### HW Additions into existing Data Centers

GPU nodes have been added to the fleet using the Deployment Tools add (or remove) nodes
capabilities.

The hardware will generally be dynamically added to the environment (though it can also be there for
initial environment stand up).

- HW Operations rolls servers with GPUs into the m-zone
- HW Operations runs stress tests against the system to properly burn in the GPUs
- HW Operations turns system over to Fleet team to add to m-zones
- Fleet team updates m-zone definitions
- Deployment tools are run to dynamically add nodes in (functionally complete)

This is consistent with the general environment grow process.

GPU nodes are marked in the m-zone.yaml as `gpu: true` to indicate that they have GPUs. While this
can be figured out dynamically, this flag allows the fleet / release engineering team to identify if
they wish to release a server as GPU capable or not. This is needed for capacity management
purposes. In the future the fleet APIs will dynamically create the m-zone.yaml and will build these
attributes based off of inventory data contained within it. This is tied to the broader FedRAMP
initiative.

There will also be a `profileClass` attribute set on the nodes in the m-zone yaml and an
`allowed_profile_classes` attribute set on the profiles in the global catalog. The profile class
will be set to `gx1` for Skylake nodes and `gx2` for Cascade Lake nodes, and the profiles will list
both of these classes in the allowed list. This will allow for the fleet metrics to better surface
what types of nodes are available for GPUs and will allow for better limiting what profiles end up
on what type of nodes. This concept isn't unique to the GPU nodes and profiles, and just uses the
existing support.

For the initial A100 GPU servers being provided for research-only, we will use a `gx2-a100` profile
class. Depending on the timing of the future GA of the A100, it may use this same profile class or
a different one. That decision will be made when the GA of the A100 is taken forward later.

The scheduler will understand the make/model/quantities of GPUs in the servers. It will match the
InstanceProfile to a host that supports that configuration.

#### Image Deployment

As stated earlier, there will be no images provided initially that contain the GPU drivers installed
on them. This will follow-on later via the 007 project support.

All existing images, those both provided by default by VPC and custom images, will be allowed to be
used with GPU profiles. The users will have the opportunity to install the GPU drivers on any image
that they wish to select.

### 3.1.6 GenCTL Architecture

Much of the GenCTL architecture will stay consistent to what was provided for the GPU support for
Power. The VM Scheduler will keep track of which GPUs are in-use for a given Node and only schedule
GPU profiles to nodes that have available GPUs and only to the sockets with those GPUs. The
compute-agent will then attach the GPU to the VM through PCI passthrough. Most of the rest of the VM
lifecycle is consistent with non-GPU VSIs.

The key difference from the design implemented for Power than that being provided for x86 here is in
the flow of wiping GPU memory and handling detecting failures to the GPUs. As stated earlier, to
handle these aspects, this design will leverage the HostDevice/DeviceChunk model introduced in the
[Instance Storage SRB](https://github.ibm.com/cloudlab/srb/tree/master/proposals/787).

In addition to the HostDevice and device-agent model provided for Instance Storage, there is the
need for introducing a new controller micro-service as the bridge between the GPU specification in
the VirtualMachine object and the DeviceChunk object that is used by the device-agent. For Instance
Storage this was the LocalDisk object and local-disk-controller.

Rather than keep adding new controllers for each PCI Device that has support added, we are
introducing a more generic PciDevice object and pci-device-controller that will handle any
additional PCI devices that are added, such as GPU and HSM. That flow is represented in the diagram
below:

![Common PCI Device Controller](./materials/pci-device-controller.png)

Beyond that, the flow through the device-agent will be very similar to what was done for Instance
Storage where there is an extension in the compute-proxy that will handle collecting GPU inventory
and clearing GPU memory via a script.

In addition, there will be updates to the vm-scheduler to make use of these HostDevice objects to
determine what GPUs are on a node rather than the kubernetes annotations. Also, the scheduler will
have enhanced logic for placing VSIs with respect to GPUs, consisting of the following criteria:

- For 1 GPU profiles, all vCPUs will be placed on the socket (NUMA node) containing the GPU.
- For 2 GPU profiles, the vCPUs will be spread evenly across the 2 NUMA nodes for the sockets that
  have the GPUs. This represents both sockets for Skylake but for Cascade Lake will only place vCPUs
  for GPU profiles on 2 of the 4 sockets.
- On each NUMA node with GPUs attached, 16 vCPUs and 128 GB of memory (and 32 Gbps network bandwidth
  will be reserved for GPU profile, the rest will be allowed to be filled with non-GPU profiles. For
  Cascade Lake, the 2 NUMA nodes without GPUs on them will allowed to be completely filled with
  non-GPU profiles.
- No GPU HostDevice marked as failed will be allowed to be have a VSI scheduled to use it.
- The 8 A100 GPU profile for research, will split evenly across the 2 NUMA Nodes, each with 4 GPUs.

## 3.2 Architectural Considerations and Impact

### 3.2.1 Security

There are no new secrets that need to be defined for GPU integration. That said, there are a few
factors for security that need to be considered.

#### User Upgrading Firmware

Since GPU devices are being passed through to the VSI, the customer owns the device. In theory they
could put rogue firmware on the device and create new attack vectors. Doing so would be exceedingly
difficult, but not impossible.

In order to prevent customers from doing so there are two mechanisms that will be put in place. The
first is that NVIDIA provides through the `nvflash` command a `protect-lock` option that will lock
the firmware so that it cannot be updated until the host is powered off and on again. Since we will
only upgrade the firmware by taking the node out of the m-zone, we will enforce this lock before
the first VM is added to the host after it is booted. This will prevent not only customer VSIs from
being able to update the firmware, but also anyone from updating it from the host until the next
power off.

The second mechanism as an extra safety check is that we will detect what firmware level is on the
GPU when stopping/starting the VSI, and mark the GPU as failed if it one of the acceptable levels.
NVIDIA has a way to not only check the version of the firmware on the card, but also provides a
checksum/hash to compare that none of the bits are different. This allows us to detect if the
customer has put rogue firmware on the card, and mark it as failed so no other customer VSIs will
get scheduled to use that GPU. This will also cause an alert and LogDNA and Ops Dashboard entries
to be updated, same as the other scenarios when a Host Device is marked as failed.

#### GPU Memory

The GPU memory is DRAM. When a GPU driver is loaded, it wipes all of the memory on the GPU.

In theory, someone could boot a custom image without a GPU driver installed, write custom logic to
inspect the memory and identify the previous customers data.

To prevent any issues, NVIDIA has provided a script to IBM that will wipe the memory on the GPUs. It
does not require any specific device driver to be loaded, and directly interacts with the PCIe bus.
It works across multiple different card revisions and only requires python be on the host system
(already available). It wipes the GPU memory in ~2 seconds for V100, and slightly longer for A100s,
since there is more memory to clear there.

In addition to wiping the memory, the script also provides a test option to verify that the entire
memory was really wiped. We will use this option to do a verification that everything was zeroed.

#### 3.2.1.1 FS Cloud Considerations

No component of this project will require elevated privileges or produce, persist, or operate on
sensitive data.

#### 3.2.1.2 Secrets

The new pci-device-controller microservice (for HSMs and GPUs) uses a new certificate for the
diagnostics server API. This is
[documented](https://github.ibm.com/gensec/platform-inventory/blob/main/secops/docs/genctl-genctl-pci-device-controller.md)
in the vault secret documentation. It must be rotated every 365 days, but we will target rotation
every 180 days.

#### 3.2.1.3 Control Plane Hardening

A new pci-device-controller microservice will be added for HSM, GPU, and future PCI device support.
See
[hardening status](../architecture/security/hardening/genctl/compute/pci-device-controller-hardening-status.md)

#### 3.2.1.4 NIST Considerations

- Addition of a new major/minor Linux distribution or base image?  No
- Changes to the "interconnections" in IBM operated components/control planes?  No
- Changes to the open network ports on any IBM operated system?  No
- Changes to FIPS certified libraries or modules?  No
- Addition of a privileged process (a new system process running with sudo/root)?  No
- Changes to the methods of remote access and network encryption?  No
- Changes to the network perimeter protection?  No
- Changes to the existing authentication and authorization mechanisms or audit?  No
- Changes to credential rotation or session timeout policies?  No
- Changes to 3rd party hardware or software packages deployed in the inventory located inside the
  FedRAMP boundary?  No

### 3.2.2 Performance, Scalability, and Resource Consumption

#### Performance

The control plane will have small delays in provisioning GPU instances. The PCI passthrough takes 30
additional seconds compared to a standard VSI deploy. That scales per card. This is needed to
properly quiesce the card from the compute host and attach to the VSI. The GPU memory wipe only adds
a few seconds to the overall provision flow.

These instance profiles are considered "specialty" VMs and are not required for scaling in the
Gartner criteria.

The systems management team will determine the power profile to run the systems in. It is expected
that the v100's will run at peak performance due to how they are spread across the racks. Other GPU
integrations may have more variation.

There will be an activity tracker event to indicate that the GPU memory has been wiped. This is in
addition to the existing `DELETE /instances/<id>` event and is specific to identifying that the GPU
memory has been wiped. This is similar to the AT event for Instance Storage.

#### Scalability

Only two GPUs per VSI will be supported initially for the v100. This is due to hardware constraints.

No expected control plane scalability constraints.

For the initial A100 offering for research, there will be 8 GPUs per VSI which causes a proportional
increase in VSI provision time at 2 levels. The clearing of GPU memory is done sequentially to each
GPU so will take longer for 8 GPUs than 2. Also, for libvirt and qemu attaching additional PCIe
devices to the VM during VM creation, there is a proportional increase per device.

#### Resource Consumption

The v100 GPUs only fit into the x86 Skylake and Cascade Lake sleds. In both of these cases, the GPU
profiles will not fill up the entire cpu and memory for the sled. Due to this, the offering team has
required that the rest of the capacity be allowed to be used for non-GPU profiles. Enough capacity
will be reserved by the scheduler to allow both GPUs on the sled to be utilized, and the rest will
be allowed for non-GPU profile.

The initial A100 profile for research will use all GPUs on the sled, so they will be given all of
the CPUs and Memory on the box that isn't reserved for the control plane, and the entire network
bandwidth.

### 3.2.3 Monitoring, Diagnostics, and Troubleshooting

The capacity monitoring dashboard will be updated to show new capacity pools for the GPU systems.

The operations dashboard will show the GPUs on the hosts, as well as the instances.

Logs flow through to LogDNA for issues. Activity Tracker will be used for the auditable memory wipe
events.

As mentioned earlier, a GPU failure will be raised both to Ops Dashboard and through a sysdig alert
when a GPU failure is detected performing operations such as clearing the GPU memory. However, when
the GPU is attached to a VM, we have no current mechanism to detect failures, since the GPU is
attached to the VM rather than the Host. There are possible out-of-band options to determine various
status conditions that we will consider in the future for integrating into Zabbix, but at this time
the failures will only be detected when the GPU is not attached to a VM.

### 3.2.4 Lifecycle, Metering, and Billing

The lifecycle of the GPUs is tied to the Instance. There are no new high level resources being
created.

Quotas will exist for GPUs. The current quota support that was added for Power is at a total number
of GPUs per account across all types of GPUs. This is sufficient at this time since we are only
supporting 1 type of GPU, Tesla V100. However, when a second type of GPU is supported this will need
to be enhanced to be per GPU-type, to allow for Offering Management to properly capacity plan each
GPU type in a given region. This is needed because different GPUs have significantly different price
points.

Billing will flow through as a new part. Each GPU model and memory configuration will map to a new
BSS billing part. That will then flow through as a part price.

The pricing has been defined and the OMs have created the `part-is.v100-hours` part for the initial
Tesla V100 GPU delivery.

Since the A100 is currently only being provided for Research and through the DOU they are making
monthly payments rather than the normal billing flow, a special one-off `Research-nlp-Cloud` billing
plan has been created, such that any VSIs they create with the `gx2-80x1280x8a100-internal` profile
will be billed to that plan so there is no charge for cpus, memory, gpus, instance storage, etc.
Any VSIs they create with any other profile that will target other servers will be billed through
the normal billing plan.

In the future when the A100 GPU does GA, there will be support put in place to use the standard
billing plan that will use the A100 GPU billing part number `part-is.a100-hours`.

### 3.2.5 Compliance

The GPUs are already part of the following inventory systems:

- IMS
- Fleet
  - [RIF (Rack Information File)](https://github.ibm.com/cloudlab/platform-inventory/tree/master/region/rif)
  - APIs
- Kube-Deploy
- Instance Scheduler

### 3.2.6 Fault Tolerance

When a GPU fails, the customer will need to shut down their VSI and start it back up. As part of the
shut down, the control plane will detect that the GPU is failed and will isolate the GPU from future
reservations.

The data on the GPUs is generally volatile. Losing a given node is impactful, but persistent storage
will not be impacted.

### 3.2.7 Hardware Platform Support

No impact to the host kernel.

Fleet's APIs have been updated to return the GPU information on a per node basis.

### 3.2.8 Component Level Development

| Area                            | SRB Mentor(s)                       | Impacted | Notes                                                                     |
| ------------------------------- | ----------------------------------- | -------- | ------------------------------------------------------------------------- |
| CI/CD (NG)                      | Zack Grossbart                      | Yes      | Include GPU nodes in pool. Update hostos payloads for GPU driver support. |
| Compute (NG)                    | Drew Thorstensen, Dan Flowers       | Yes      | Integration of GPU nodes into scheduling mechanics, compute code, etc...  |
| Fleetman (NG)                   | Malcolm Allen-Ware                  | Yes      | Updating of mzone.yaml to identify GPU node types, support in FMR, etc... |
| Hardware (NG)                   | Malcolm Allen-Ware, Ed Seminaro     | Yes      | Deployment of GPU nodes.                                                  |
| Genctl                          | Drew Thorstensen, Dan Flowers       | Yes      | Support for new profiles and provisioning.                                |
| Operations (NG)                 | Alex King                           | Yes      | Support for new pool of nodes.                                            |
| Performance                     | Weiming Gu                          | Yes      | Verification of the performance of the GPU nodes, AI specific workloads.  |
| Platform Integration            | Yvonne Chan, meem, Dan Moravec      | Yes      | Billing, Activity Tracker, GHoST                                          |
| RIAS (API, Ingress, Frameworks) | Dan Hudlow, meem                    | Yes      | New metadata in API (profiles and images). Needs to be passed to Genctl.  |
| Security                        | Vugranam (VC) Sreedhar, Yi-Hsiu Wei | Yes      | Validation of new components and impact to security.                      |
| Software Defined Power (SDP)    | Malcolm Allen-Ware                  | Yes      | Determine GPU impact to SDP.                                              |
| Support                         | Majad Ahmed                         | Yes      | New profiles to support, possible questions on how to utilize GPUs.       |
| Telemetry                       | Yvonne Chan                         | Yes      | Limited impact due to VSI profile creation. Needs to be verified.         |
| Test                            | Bill Goleman                        | Yes      | New testing of GPU components                                             |
| UI/CLI                          | Eduardo Abe                         | Yes      | New profiles to support.                                                  |

# 4. Testing

GPU specific tests will be added to the System Test, Smoke and Performance team areas. These will be
an optional opt-in set of tests as not all environments will have GPUs available to them. These
tests will be fairly basic initially, testing the provisioning and some light evaluation of the GPUs
themselves.

Verification will occur that all of the operating system stock images work properly with the GPUs
(per documentation available on how to set them up). This will be added to the public image
verification process for each new image rev.

The main test plan for the tests executed by QA are available on the
[V100 Test Plan](https://confluence.swg.usma.ibm.com:8445/display/CATS/Accelerated+Compute+%28V100+GPU%29%3A+Test+Coverage+Overview)
wiki page and the results on the
[V100 Test Results](https://confluence.swg.usma.ibm.com:8445/display/CATS/Accelerated+Compute+%28V100+GPU%29%3A+Test+Results)
wiki page.

A set of tests has also been added by the HW operations team to burn in the GPU based systems prior
to handing off to the Fleet team.

Internal sponsor users will also validate the solution.

## 4.1 CI Pipelines - Automation

The primary focus for getting GPUs in automation m-zones will be first for the key blast radius test
m-zones (such as compute). From there, there will be a focus on including GPUs in the CI Pipeline
m-zones. Given there will be 15 of these going forward, it is likely not all CI m-zones will be able
to have GPU nodes.

Which and how many CI m-zones will get a single GPU node will depend on whether there is a direction
shift of targeting certain CI pipelines to certain CI m-zones. This will not be available from day
one and is intended to be tackled in the next several months after the QA tests are written.

The automated tests written to provision GPUs will be first executed in development, integration
and staging environments, and then extended into the blast radius test environments, and then
extended for regression in the CI m-zones.

# 5. Solution Space

## 5.1 Lessons Learned

Many lessons were learned through the Power Beta. The primary is that the GPU drivers can crash the
host kernel as part of the GPU memory wipe. As such, we are switching to use a python utility
provided by NVIDIA for clearing GPU memory.

We also learned that Instance Storage is a key pre-requisite for many GPU workloads, and will be
included in future profiles for additional GPUs added. In order to get these Tesla V100 GPUs
available sooner, Instance Storage will not be provided for these GPUs. The Instance Storage will
be first attempted with the A100 GPU profile being provided for research-only in WDC.

## 5.2 Alternatives Considered

One immediate question is why start with the last generation GPUs instead of catapulting to the
A100s. There are two factors.

1. Inventory - we have plenty of these GPUs in stock, currently depreciating and not making money.
   They need to get into the fleet.
2. Optimization - This design supports PCIe A100s and lays the foundation for the work around a SXM4
   A100. A quick follow on will be added to support the A100 NVLink GPUs (along with partial GPU
   support).

## 5.3 Competitive Offerings

AWS, GCP and Azure all have broad GPU offerings. This is our first step to achieve parity.

# 6. Known Issues

None

# 7. Patents

None

# 8. Meta

None

# 9. Glossary

# 10. References

- [NVIDIA Tesla V100](https://www.nvidia.com/en-us/data-center/v100/)
  - 16 GB Variant, PCIe Gen 3 connection
- [NVIDIA T4](https://www.nvidia.com/en-us/data-center/tesla-t4/)
  - 16 GB, PCIe Gen 3
- [NVIDIA A100](https://www.nvidia.com/en-us/data-center/a100/)
  - 40 GB, PCIe Gen 4 available
- [007 Project](https://bigblue.aha.io/products/PUBCLOUD/strategic_imperatives/PUBCLOUD-G-44)

<!-- Emacs formatting setup -->
<!-- Local Variables: -->
<!-- fill-column: 100 -->
<!-- End: -->

<!-- Vim formatting setup. Must be in first or last 5 lines by default -->
<!-- vim: set textwidth=100 : -->
