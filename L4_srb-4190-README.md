---
template_version: 2023-04-07 # DO NOT CHANGE -- this comes from the copied template.md

## These *MUST* be updated by the mentor as the proposal goes through the SRB process.
state: closed # open or closed
status_label: in-production # See https://github.ibm.com/cloudlab/srb#status-labels
last_update: 2024-03-20 # Set to today's date whenever the proposal is updated
---

<!-- cSpell:ignore - MTFDKCC TNHR Belog Suraksha Vidarthi watsonx Texel GFLOPS CBLM Pitchappa -->
<!-- cSpell:ignore -  Indish CUDA Metruck GDDR Welp -->

# Inference Infrastructure Enablement - NVIDIA L4 24GB PCI

# 1. Introduction

## 1.1 Project Name

Inference Infrastructure Enablement

NVIDIA L4 24GB PCI

## 1.2 Submitter and Contributors


## 1.3 Mentor(s)

## 1.4 Offering Manager(s)


## 1.5 Review Flow (determined by mentor)

Lightweight

## 1.6 Production Target Date

4Q2023 SA in WDC, FRA, TOK & LON regions.

# 2. Project Overview

Aha! [VSIVPC-636](https://bigblue.aha.io/features/VSIVPC-636)

VPC-REQ [VPCREQ-321](https://jiracloud.swg.usma.ibm.com:8443/browse/VPCREQ-321)

## 2.1 Overview

IBM is in the business of building [AI models](https://www.ibm.com/artificial-intelligence). AI
models are built on training infrastructure, and used on inference infrastructure. In order to build
and use models of different sizes from millions of parameters to those using hundreds of billion
parameter models, large scale clusters are needed to do so with state-of-the-art networking,
accelerators, and servers.

This SRB is one of a number of inference infrastructure accelerators that will be enabled by IBM
Cloud. Currently, IBM Cloud provides the
[NVIDIA V100 16GB](https://www.nvidia.com/en-us/data-center/v100/) in a VSI as a Commercial offering
for Inference; Additionally, the NVIDIA V100 32GB is provided in an allow listed profile for use as
an inference node for watsonx AI project. These cards are highly effective for their use cases,
none-the-less, there is a need for a larger pool of available GPU instances and a more diverse
select of inference accelerators, especially in terms of greater memory in order to contain larger
models.

This project is to enable the [NVIDIA L4 24GB](https://www.nvidia.com/en-us/data-center/l4/) PCIe
based card for use as part of the Inference Infrastructure fueled by IBM AI projects and Customer
demand. This project builds upon the work completed as part of the
[GPU Integration into VPC Gen 2 for x86 SRB](https://github.ibm.com/cloudlab/srb/tree/master/proposals/1292).
This offering will provide new GPU Profiles that include a set of GPUs to be attached to a VSI. And
as with the NVIDIA V100's, the attachments of GPUs to VSIs will be via a KVM pass-through to allow
near native speeds within the VSIs. The family of profiles will be the `gx3` family.

## 2.2 Consumer Interaction Model

Customers will be able to purchase a series of new VSI profiles. The new VSI profiles specify the
same characteristics that are expected in existing profiles - vCPU, memory, etc... However, these
profiles will also include a type of GPU, number of GPUs and network speeds.

| Instance Profile Name | vCPU | RAM (GiB) | GPU Qty | GPU Type     | Bandwidth - Max | Instance Storage (GiB) |
| --------------------- | ---- | --------- | ------- | ------------ | --------------- | ---------------------- |
| gx3-16x80x1l4         | 16   | 80        | 1       | L4 PCIe 24GB | 32              | NA                     |
| gx3-32x160x2l4        | 32   | 160       | 2       | L4 PCIe 24GB | 64              | NA                     |
| gx3-64x320x4l4        | 64   | 320       | 4       | L4 PCIe 24GB | 128             | NA                     |

with these profiles, the customer will need to select an image that will support the NVIDIA Software
stack. These include Redhat Enterprise and Ubuntu.

## 2.3 Requirements and Use Cases

- The L4 PCIe card is limited in quantity, as such, OM has determined that this project will be
  delivered in a Gen3 SPR based VPC server configuration building on a proven, certified, and
  validated platform.
  - Using a known platform will decrease the amount of time to qualify the system and expedite the
    time to market.
  - The L4 Accelerator node will be based on a bx3d Lenovo based Gen 3 server, and it should be
    noted that this server does have NVMe drives as part of its BOM. The NVMe drives will need to
    remain in the server, even though the profile will not use them. There are a number of reasons
    for this. First, these servers are L10 servers and are shipped as a qualified system, and these
    drives cannot be removed, otherwise, a new system would need to be qualified. Secondly, these
    systems are leased, and the server will be returned. When the server is returned, all original
    components must be returned with the associated server. Storing and tracking individual drives
    and then returning them to the correct server at the end of the lease would incur too much
    operational overhead. The qualified configuration for the hardware profiles. The qualified config
    framework will ensure firmware configuration for nodes based on the hardware profile `gx3-l4`.
- GPUs must be attached to a VSI at the time of the creation of the VSI. They can not be modified
  after VSI creation.
- The VSI profiles must have metadata associated with them to describe the types of accelerators
  that are attached (what brand, quantity, memory size in gibibytes and model of GPU).
- GPUs are a part that need to be monitored within the system. Failure of this part should trigger
  the same flows as other hardware failures (ex. memory or NIC failure).
  - This leverages the HostDevice model introduced in the
    [Instance Storage SRB](https://github.ibm.com/cloudlab/srb/tree/master/proposals/787).
- GPU memory must be physically cleared off the cards after a VSI is destroyed or shutdown.
  - Auditable evidence must be produced via Activity Tracker that the GPU's memory was wiped on VSI
    destroy or shutdown.

## 2.4 Out-of-Scope

- Full support of the device model by surfacing the PCI topology will not be done in this SRB.
- When both GPUs are fully utilized on the node, there will be excess CPU and Memory resources. At
  this time, there will be no changes to the Scheduler to utilize these excess resources.
  - Offering Management has confirmed that there is still a margin on the offering even with
    stranded resources.
  - There are currently limited quantities of GPU Training and Inference Accelerator nodes, making
    changes to Scheduler of limited value.
  - If at a future date, the excess resources are needed to be optimized, a new SRB will be created
- VSIs with GPUs will not be able to live migrate.
  - This is due to the nature of PCIe pass-through
  - A future SRB proposal will be needed to add in GPU Live Migration support, however it will
    always be a limitation for supporting the latest/greatest GPUs at _native_ speed.
  - There are interesting technologies like vGPU that may solve this at some point. However, that
    solution does not work completely, may not be certified for use with all GPU software solutions,
    and has additional complexity.
- Images with the GPU drivers pre-installed will not be available.
  - Licensing prevents us from including the NVIDIA drivers in the image without presenting license.
  - Customers will currently need to install the drivers on the VSI, or custom image, themselves.
  - Customers can create custom images with GPU drives and reuse them.
  - Partners could provide images with 3rd party images with pre-installed devices drivers.
- Dedicated Host profiles will not be provided with this SRB.
- gx3 L4 GPU profiles will not hold reservation policy. From offering perspective, customers are
  not allowed to reserve gx3 L4 VSIs as the GPUs are limited and rare resources.
- Expanding Observability and Monitoring will be out of scope for this SRB. Currently, GPU usage
  metrics are not provided from the hypervisor. A project is beginning to expose these metrics so
  that they can be collected. An epic for this has been created
  [XLR8R-68](https://jiracloud.swg.usma.ibm.com:8443/browse/XLR8R-68)

## 2.5 Dependencies, Dependents, and Incompatibilities

### 2.5.1 Other VPC Projects Under Development

There are no other VPC projects that are under development that are required for this project.
Dependencies were full-filled with the completion of
[GPU Integration into VPC Gen 2 for x86](https://github.ibm.com/cloudlab/srb/tree/master/proposals/1292).

### 2.5.2 Other IBM Projects Under Development

None.

### 2.5.3 Third-party

There are several hardware dependencies that are needed for this project.

<!-- cSpell: disable -- disables checking -->

| Manufacturer | Type           | Model     | Status in Fleet |
| ------------ | -------------- | --------- | --------------- |
| Lenovo       | System / Board | SR 650 v3 | Existing        |
| NVIDIA       | GPU            | L4 PCIe   | New             |

<!-- cSpell: enable -- enables checking -->

The customer VSIs will need the GPU drivers, provided by NVIDIA, installed on them to make use of
the GPUs. The stock images provided in VPC will not contain these drivers, so for those images the
customers will need to install the drivers themselves.

For clearing GPU memory, a python utility provided by NVIDIA, will be used for clearing the GPU
memory through the host. This is described more fully in the `Security` section of the SRB
[GPU Integration into VPC Gen 2 for x86](https://github.ibm.com/cloudlab/srb/tree/master/proposals/1292#321-security).

### 2.5.4 External and/or Circular

No new circular dependencies. No external service dependencies.

### 2.5.5 Brittle or Unusual

Lack of Live Migration support makes this solution unusual.

These systems will not be Secure Booted:

- The workload that is run here requires PCIe tuning within the VSI which cannot be done under
  Secure Boot
- The kernel module which is used by the GPU memory cleanup utility at hypervisor during
  L4 GPU VSI de-provisioning, is not a signed binary module. Hence host booted with Secure Boot mode,
  will not be able to cleanup the GPU memory. To make sure that the GPU memory is cleaned up while
  VSI de-provisioning, we need to boot L4 GPU compute nodes in non-secure boot mode.

### 2.5.6 Incompatibilities

- Live migration cannot be used for the VSIs on L4 PCIe nodes.
- SGX/TDX is not supported.
- VSI Secure Boot is not supported.
- Gen 3 OVMF is not used with L4 VSIs. Using the Gen 3 OVMF generates an invalid PCI I/O region.

### 2.5.7 Databases

None

## 2.6 Integration, Deployment, and Operations Considerations

All of these nodes will be managed with the existing tool set that we have operationally. This
includes:

- System Firmware Bring-Up Release bundle
- Governed Qualified Configurations
- HostOS / Kube Release Bundle deployment
- Integration with existing compute-proxy / agent to provision the devices and GPUs
- Existing monitoring / alerting will be utilized

The NVIDIA L4 GPUs will be released as SA (Selected Availability) in WDC, FRA, TOK & LON region.
Initially a minimum capacity ( 2 racks ) of Gen3 compute nodes with NVIDIA L4 GPUs will be deployed
in one zone of the SA regions.

The GPUs will show up in the operations dashboard as part of the compute host's inventory. The GPUs
are directly integrated into the Instance Profile, so a customer can not dynamically add or remove
GPUs.

During disruptive maintenance (host kernel upgrade, new system firmware, etc...) these VMs will
follow the cold migration maintenance path. The customer will be notified that they need to
shutdown/start up their instance within a certain number of days. If they fail to do so before a
given date, the control plane will restart it on their behalf. This is consistent with our
maintenance strategy for anything that does not live migrate.

NVIDIA L4 GPU offering doesn't support live migration of VSIs. We need 20% overhead for cold
migration considering 5 passes of reboot waves in each zone.
During the 30 day customer notification window for node maintenance schedule/reboot wave, customers
will need to either initiate the stop/start on their own, or be forced to have this happen in the
time-frame that we dictate.

## 2.7 Big Rules

### 2.7.1 Consistent Customer Experience

This infrastructure will only be offered on x86 VSI. It is a very specific workload type for a
client need. This is not a cross platform feature.

### 2.7.2 VPC Big Rules

- [ ] All components must be able to be upgraded live (without service degradation/outage) and
      without manual intervention
  - No - firmware of the specific devices requires that the server be taken out of the pool. This is
    consistent with server firmware updates today.
- [ ] All components must present usage metrics
  - No - Usage metrics for the GPUs are currently not exposed.
    See Section [2.5.1 Other VPC ProjectsUnder Development](#251-other-vpc-projects-under-development).
- [x] All components must use the
      [VPC/NG monitoring framework](https://github.ibm.com/cloudlab/srb/tree/master/architecture/telemetry/monitoring)
- [x] All components must use existing VPC/NG control planes -- no new control planes
- [x] All components must be controlled via well-defined APIs
- [x] All components must provide functional parity across all modalities (UI, CLI, API, Terraform)
- [ ] All components must provide functional parity across all targeted platform architectures (x86
      and Z)
  - The only targeted architecture for this is x86 VSI.
- [ ] All customer-facing GA features must interoperate with all other customer-facing GA features
  - SGX, TDX, secure boot, reservation GA featured will not be available for L4 GPU profiles.
- [x] All customer-facing UIs and CLIs must use the customer-facing VPC API to operate on VPC/NG
  resources
  - No changes at this level
- [x] All customer-facing APIs must adhere to the
  [API handbook guidelines](https://pages.github.ibm.com/CloudEngineering/api_handbook/)
  - No new APIs in this proposal
- [x] All regional APIs must be the same in all regions (this implies equivalent functionality in
  all regions)
  - No new regional APIs
- [x] All components must perform authorization using the
      [VPC/NG IAM services](architecture/platformIntegration/genesis-iam)
- [x] All components must be internally stateless and recreate their runtime state from external
      sources
- [x] Any caching or shadowing of state must never result in an API consumer observing inconsistent
      or incorrect behavior
- [ ] All hardware components must use the
      [VPC/NG diagnosability framework](https://github.ibm.com/cloudlab/srb/tree/master/architecture/telemetry/diagnostics)
      and implement the on-demand diagnostic API
  - No changes/addition to diagnostic APIs or Operator APIs.
- [x] All services must present an API for get/set/watch of state
- [x] All connections between components must be mutually authenticated, and encrypted using AES128
      or higher
- [x] All components must integrate into the existing operations tooling for deployment and
      maintenance

### 2.7.3 Cloud Security Baseline Requirements

> Does this project meet all
> [Cloud Security Baseline Requirements](https://pages.github.ibm.com/ibmcloud/Security/guidance/Cloud-Security-Baseline.html)?
> Please list each bullet item from the above link with a "yes" or "no" and if the answer is "no",
> provide rationale. (If the answer is "no" because this project depends on out-of-scope work that
> is incomplete, provide references to open issues for that work.)

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

The L4 PCIe is assumed to be of limited quantity and the current on-hand inventory will be the total
inventory for the life of this project. This assumption has driven the design of the instance
profiles outlined in this SRB utilizing the Gen3 Sapphire Rapids platform to expedite the delivery
of this project. Further, the IBM Cloud team is working closely with the watsonx teams in deploying
the hardware to specific geographies based on the projected needs and availability of hardware.
Additionally, Offering management is working to determine how external customers may use the L4 GPU node.

If the availability of the hardware changes, there may be an opportunity to implement an optimized
platform for the L4 PCIe accelerator. If that occurs, an updated SRB will be introduced.

Existing GPU quotas will be inherited. No new GPU quotas (ex. quotas specific to this model of GPU)
will be implemented.

# 3. Architecture, Interfaces, and Impact

## 3.1 Architecture and Interfaces

### 3.1.1 Architecture and Technical Design

#### 3.1.1.1 Hardware Design

The GPU will be directly attached to free PCIe slots in the compute host.

![GPU Host](./assets/GPUHost.png)

The L4 PCIe in the diagram above shows how NVMe based Instance Storage can optionally be used with
GPUs. The original NVIDIA V100 16GB node did not contain instance storage, it was introduced with
the NVIDIA A100 SMX node.

The following provides some comparisons of NVIDIA PCIe Inference Accelerator Cards

|                    | L4 PCIe        | Tesla V100 PCIe | A100 PCIe      |
| ------------------ | -------------- | --------------- | -------------- |
| Architecture       | Ada Lovelace   | Volta           | Ampere         |
| Processor Size     | 5 nm           | 12 nm           | 7 nm           |
| Transistors        | 35,800 million | 21,100 million  | 54,200 million |
| Bus Interface      | PCIe 4.0 x16   | PCIe 3.0 x16    | PCIe 4.0 x16   |
| Clock Speed - Base | 795 Mhz        | 1230 Mhz        | 1065 Mhz       |
| Memory size        | 24GB           | 32GB            | 80GB           |
| Memory Type        | GDDR6          | HBM2            | HBM2e          |
| Memory Clock Speed | 1563 MHz       | 876 MHz         | 1593 MHz       |
| Memory Bus         | 192bit         | 4096bit         | 5120bit        |
| Wattage            | 72             | 250             | 250            |
| Pixel Rate         | 163.2 GPixel/s | 176.6 GPixel/s  | 225.6 GPixel/s |
| Texture Rate       | 489.6 GTexel/s | 441.6 GTexel/s  | 609.1 GTexel/s |
| FP16 (half)        | 31.33 TFLOPS   | 28.26 TFLOPS    | 77.97 TFLOPS   |
| FP32 (float)       | 31.33 TFLOPS   | 14.13 TFLOPS    | 19.49 TFLOPS   |
| FP64 (double)      | 489.6 GFLOPS   | 7.066 TFLOPS    | 9.746 TFLOPS   |

#### 3.1.1.2 Hypervisor Architecture

The hypervisor will be HostOS 6.x and no linux kernel revision/version requirements to support L4 GPU
cards in passthrough mode for VSIs.

#### 3.1.1.3 Profile Class

The qualified configuration for the compute nodes having L4 GPUs will be `gx3-l4`.
The gx3 L4 GPU VSI profiles will hold `gx3-l4` in `allowed_profile_classes` list which will help the
VSIs to be scheduled on these hardware tagged compute nodes.
The compute node roles in the cluster will have `compute: true` and `gpu: true`.

### 3.1.2 Customer Experience

Customers will simply have a new set of VSI profiles which include the GPU parts within them.
There are no new POST or GET APIs. The existing GET APIs for Instances or InstanceProfiles will
return the GPU information.

The following is a sample Instance Profile yaml in global-catalog-profiles repo:

<!-- cSpell: disable -- disables checking -->
```json
{
    "id": "gx3-16x80x1l4",
    "_id": "gx3-16x80x1l4",
    "_rev": "13-e5fa6676dc985cf8bacb0c0416a4fbd4",
    "catalog_crn": "crn:v1:staging:public:globalcatalog::::instance.profile:gx3-16x80x1l4",
    "url": "https://globalcatalog.test.cloud.ibm.com/api/v1/gx3-16x80x1l4?complete=true",
    "name": "gx3-16x80x1l4",
    "kind": "instance.profile",
    "overview_ui": {
        "en": {
            "display_name": "GPU 16x80x1L4 - Gen3",
            "long_description": "GPU Profile, 16 vCPU, 80 GiB, 1L4 GPU  - Gen3",
            "description": "GPU 16x80x1L4 - Gen3"
        }
    },
    "images": {
        "image": "https://cache.globalcatalog.test.cloud.ibm.com/static/images/uploaded/profile.png"
    },
    "children_url": "https://globalcatalog.test.cloud.ibm.com/api/v1/gx3-16x80x1l4/%2A",
    "disabled": false,
    "active": true,
    "tags": [
        "ibm_created",
        "is.instance:beta",
        "is.instance:gpu",
        "is.instance:gt"
    ],
    "provider": {
        "email": "vpccomp@us.ibm.com",
        "name": "IBM"
    },
    "created": "2024-01-02T15:02:31.270777213-05:00",
    "updated": "2024-02-16T09:08:58.785088677-05:00",
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
            "starting_price": {
                "plan_id": "6bed29d7-67ba-43ca-83d0-9576a379a1ce"
            },
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
                "capabilities": {
                    "confidential_compute_modes": {
                        "default": "disabled",
                        "options": [
                            {
                                "value": "disabled"
                            }
                        ]
                    },
                    "secure_boot_modes": {
                        "default": false,
                        "options": [
                            {
                                "value": false
                            }
                        ]
                    }
                },
                "default_config": {
                    "allowed_profile_classes": [
                        "gx3-l4"
                    ],
                    "bandwidth": 32000,
                    "cpu": 16,
                    "cpu_family": "Sapphirerapids",
                    "freqency": 2000,
                    "gpu_count": 1,
                    "gpu_manufacturer": "nvidia",
                    "gpu_memory": 24,
                    "gpu_model": "L4",
                    "max_nics": 5,
                    "numa_count": 1,
                    "os_architecture": [
                        "amd64"
                    ],
                    "port_speed": 32000,
                    "provisioning_timeout_seconds": 7200,
                    "ram": 80,
                    "status": "current",
                    "vcpu_architecture": "amd64",
                    "vcpu_manufacturer": "Intel"
                },
                "family": "gpu-l4",
                "generation": "gt",
                "measures": [
                    {
                        "component": "instance",
                        "deployments": [
                            {
                                "meters": [
                                    {
                                        "quantity": "1",
                                        "unit": "INSTANCE_MULTI_TENANT"
                                    },
                                    {
                                        "quantity": "0",
                                        "unit": "INSTANCE_DEDICATED_HOST"
                                    },
                                    {
                                        "quantity": "0",
                                        "unit": "INSTANCE_RESERVATION"
                                    },
                                    {
                                        "quantity": "1",
                                        "unit": "INSTANCE"
                                    },
                                    {
                                        "quantity": "16",
                                        "unit": "VCPU"
                                    },
                                    {
                                        "quantity": "80",
                                        "unit": "MEMORY"
                                    },
                                    {
                                        "quantity": "gx3-16x80x1l4",
                                        "unit": "PROFILE"
                                    }
                                ],
                                "plan": "6bed29d7-67ba-43ca-83d0-9576a379a1ce",
                                "type": "multi-tenant"
                            }
                        ]
                    },
                ],
                "resource_type": "instance"
            }
        }
    },
    "complete": true
}
```
<!-- cSpell: enable -- enables checking -->

See the section [3.2.4.3 Billing](#3243-billing) for specifics on Billing for these profiles.

#### 3.1.2.1 User Interface (UI)

Three profiles will be made available. No new screens required.

#### 3.1.2.2 Command Line Interface (CLI)

No new CLI impacts.

#### 3.1.2.3 Terraform and Packer

No new Terraform / Packer requirements.

#### 3.1.2.4 Customer-facing Documentation

The GPU section of the documentation will need to be updated with the new profiles provided and
Release Notes will need to be added to announce the new Inference node.

The main explanation of GPUs is available on the
[Managing GPUs](https://cloud.ibm.com/docs/vpc?topic=vpc-managing-gpus) page and the list of GPU
profiles is available on the [Instance Profiles](https://cloud.ibm.com/docs/vpc?topic=vpc-profiles)
page.

### 3.1.3 Customer-facing APIs

No new / additional customer facing APIs.

#### 3.1.3.1 Activity Tracker (AT)

No new activity tracker events required.
While L4 GPU VSI is deleted, the associated GPU host device(s) also get deleted. While deleting the
GPU host devices, the memory must be cleared. The exiting activity tracker event implemented for V100
GPU  will be common for all GPU profiles including L4 VSIs.
Activity Tracker event `is.instance.gpu.wipe` produced whenever the GPU memory is cleared.

#### 3.1.3.2 Global Search and Tagging (GhoST)

No new / additional GhoST requirements

#### 3.1.3.3 IBM Cloud Compliance and Security Center (IBM SCC)

No changes or additions to the APIs. And hence no new instructions to IBM SCC.

### 3.1.4 Internal APIs and Schemas

No new Internal API or modification to the internal APIs required.

### 3.1.5 Deployment Architecture

No new controllers in regional as well as zonal space.

For provisioning VSIs, instance Profiles will act as allow list mechanism.
As the instance profiles themselves are allow listed. Further, the corresponding service and bss
plans will be public. These plans are public so that there are no billing issues if they are
restricted at a later date.

Customers will not will not be able to create a resource against those plans if they are not allow
listed to the instance profile.

For regional perspective, geo-tagging will be used by OM to restrict which production regions will
see the instance profiles for the particular GPUs that are available in them.

#### 3.1.6 HW Additions into existing Data Centers

GPU nodes have been added to the fleet using the Deployment Tools add (or remove) nodes
capabilities. This had been implemented as part of the original GPU SRB
[GPU Integration into VPC Gen 2 for x86](https://github.ibm.com/cloudlab/srb/tree/master/proposals/1292).

The hardware will generally be dynamically added to the environment (though it can also be there for
initial environment stand up).

- HW Operations rolls servers with GPUs into the m-zone
- HW Operations runs stress tests against the system to properly burn in the GPUs
- HW Operations turns system over to Fleet team to add to m-zones
- Fleet team updates m-zone definitions and allocations
- Deployment tools are run to dynamically add nodes in (functionally complete)

This is consistent with the general environment grow process. There is a corresponding process to
remove a node from the environment.

#### 3.1.6.1 Hardware Maintenance Overhead

As with any IBM Cloud offering, thought must be given to maintaining the health of the fleet of
systems in use. The number of L4 PCIe nodes will be limited due to the limited supply from the
manufacturer. Considerations to support maintenance tasks will include:

- Number of gx3-L4 nodes tainted for backup use in the event of capacity issues. Typically, this is
  maintained at approximately 30%; however, with the limited number of servers, this percentage
  should be lowered.
- Spares will be on hand in the data center for commodity parts for faster turn-around on break-fix.
  These include Memory, Power Supplies, Network Cards.
- A robust RMA process will be needed that has world-wide coverage. The main component, the A100
  PCIe GPU, is a limited availability part. When a GPU falls out, and the part will requires an RMA,
  the process should be robust enough to reduce the repair time and the return to service time.

#### 3.1.7 Image Deployment

As stated earlier, there will be no images provided initially that contain the GPU drivers installed
on them.

All existing images, those both provided by default by VPC and custom images, will be allowed to be
used with GPU profiles. However, the user will need to confirm that the NVIDIA software stack
required for the L4 PCIe Accelerator is usable with the chosen image OS.

## 3.2 Architectural Considerations and Impact

### 3.2.1 Security

There are no new secrets that need to be defined for GPU integration. That said, there are a few
factors for security that need to be considered.

#### 3.2.1.1 User Upgrading Firmware

This feature was delivered in the original V100 16GB GPU SRB; however, there is additional
development work to enable this feature to work with the L4 PCIe Accelerator.

Since GPU devices are being passed through to the VSI, the customer owns the device. In theory they
could put rogue firmware on the device and create new attack vectors. Doing so would be exceedingly
difficult, but not impossible.

In order to prevent customers from doing so there are two mechanisms that will be put in place. The
first is that NVIDIA provides through the `nvflash` command a `protect-lock` option that will lock
the firmware so that it cannot be updated until the host is powered off and on again. Since we will
only upgrade the firmware by taking the node out of the m-zone, we will enforce this lock before the
first VM is added to the host after it is booted. This will prevent not only customer VSIs from
being able to update the firmware, but also anyone from updating it from the host until the next
power off.

The second mechanism as an extra safety check is that we will detect what firmware level is on the
GPU when stopping/starting the VSI, and mark the GPU as failed if it is not one of the acceptable levels.
NVIDIA has a way to not only check the version of the firmware on the card, but also provides a
checksum/hash to compare that none of the bits are different. This allows us to detect if the
customer has put rogue firmware on the card, and mark it as failed so no other customer VSIs will
get scheduled to use that GPU. This will also cause an alert and LogDNA and Ops Dashboard entries to
be updated, same as the other scenarios when a Host Device is marked as failed.

#### 3.2.1.2 GPU Memory

This feature was delivered in the original V100 16GB GPU SRB; however, there is additional
development work to enable this feature to work with the L4 PCIe Accelerator as there is a new
nvidia_gpu_tool.py utility version that will be delivered with this new Accelerator.

The GPU memory is DRAM. When a GPU driver is loaded, it wipes all the memory on the GPU.

In theory, someone could boot a custom image without a GPU driver installed, write custom logic to
inspect the memory and identify the previous customers' data.

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

No new secrets are required, this was completed in the original [V100 16GB SRB](https://github.ibm.com/cloudlab/srb/tree/master/proposals/1292).
It is included, for completeness and awareness:

The pci-device-controller microservice (for HSMs and GPUs) uses a new certificate for the
diagnostics server API. This is
[documented](https://github.ibm.com/gensec/platform-inventory/blob/main/secops/docs/genctl-genctl-pci-device-controller.md)
in the vault secret documentation. It must be rotated every 365 days, but we will target rotation
every 180 days.

The same diagnostics server API is also used for NVMe localdisks too.

#### 3.2.1.3 Control Plane Hardening

No new hardening is needed for this SRB, this was completed in the original V100 16GB SRB. It is
included, for completeness and awareness:

A new pci-device-controller microservice has been added for NVMe, GPU, and other PCI devices. See
[hardening status](../architecture/security/hardening/genctl/compute/pci-device-controller-hardening-status.md)

#### 3.2.1.4 NIST Considerations

- Addition of a new major/minor Linux distribution or base image? No
- Changes to the "interconnections" in IBM operated components/control planes? No
- Changes to the open network ports on any IBM operated system? No
- Changes to FIPS certified libraries or modules? No
- Addition of a privileged process (a new system process running with sudo/root)? No
- Changes to the methods of remote access and network encryption? No
- Changes to the network perimeter protection? No
- Changes to the existing authentication and authorization mechanisms or audit? No
- Changes to credential rotation or session timeout policies? No
- Changes to 3rd party hardware or software packages deployed in the inventory located inside the
  FedRAMP boundary? No

#### 3.2.1.5 Identity and Access Management

No new services. Will inherit all existing IAM policies for VSI.

### 3.2.2 Performance, Scalability, and Resource Consumption

### 3.2.2.1 Performance

The control plane will have small delays in provisioning GPU instances. The PCI passthrough takes 30
additional seconds compared to a standard VSI deploy. That scales per card. This is needed to
properly quiesce the card from the compute host and attach to the VSI. The GPU memory wipe only adds
a few seconds to the overall VSI de-provision flow.

These instance profiles are considered "specialty" VMs and are not required for scaling in the
Gartner criteria.

The systems management team will determine the power profile to run the systems in. It is expected
that the L4's will run at peak performance due to how they are spread across the racks. Other GPU
integrations may have more variation.

There will be an activity tracker event `is.instance.gpu.wipe` to indicate that the GPU memory has
been wiped. This is in addition to the existing `DELETE /instances/<id>` event and is specific to
identifying that the GPU memory has been wiped during GPU VSI delete.

### 3.2.2.2 Scalability

Only Four GPUs per VSI will be supported initially for the L4. This is due to hardware constraints.

No expected control plane scalability constraints.

### 3.2.2.3 Resource Consumption

The L4 Accelerators will be used in a Lenovo SR650 V3. The designed GPU profiles will not utilize
the entirety of the available cpu and memory for the node. At this time, this excess capacity will
not be used. If optimizations are needed to utilize the excess capacity, a new SRB will be created
to identify the work necessary to make these changes. Note that capacity will be reserved by the
scheduler to allow both GPUs on the node to be utilized.

### 3.2.3 Monitoring, Diagnostics, and Troubleshooting

The capacity monitoring dashboard will be updated to show new capacity pools for the GPU systems.

The operations dashboard will show the GPUs on the hosts, as well as the instances.

Logs flow through to LogDNA for issues. Activity Tracker will be used for the auditable memory wipe
events.

As mentioned earlier, a GPU failure will be raised both to Ops Dashboard and through a sysdig alert
when a GPU failure is detected performing operations such as clearing the GPU memory. However, when
the GPU is attached to a VM, we have no current mechanism to detect failures, since the GPU is
attached to the VM rather than the Host. Another scenario that could occur could be degredation of
the PCIe Bus causing lower performance of the system.

An epic for observability and monitoring has been opened to capture requirements and implement these
enhancements in [XLR8R-68](https://jiracloud.swg.usma.ibm.com:8443/browse/XLR8R-68). All alert needs
are accounted for in [XLR8R-68](https://jiracloud.swg.usma.ibm.com:8443/browse/XLR8R-68).

### 3.2.4 Lifecycle, Metering, and Billing

### 3.2.4.1 Lifecycle

The lifecycle of the GPUs is tied to the Instance. There are no new high level resources being
created.

This SRB will reuse the existing quotas generic GPUs quotas.

### 3.2.4.2 Metering

There is no new Metering for the L4 GPU.

### 3.2.4.3 Billing

Billing for this profile will require a new BSS plan. It will follow the Gen 3 precedent. That
means:

- 3 new plans
- 1 new Instance-Hours part

These 3 plans:

- Will not support Dedicated Hosts
- Will not support Reservations

CBLM considerations:
The existing billing events from legacy as well as CBLM are sufficient.

### 3.2.5 Compliance

The GPUs are already part of the following inventory systems:

- IMS
- Fleet
  - [RIF (Rack Information File)](https://github.ibm.com/cloudlab/platform-inventory/tree/master/region/rif)
  - APIs
- Kube-Define
- Instance Scheduler

### 3.2.6 Faults and Disasters

#### 3.2.6.1 Fault Tolerance

When a GPU fails, the customer will need to shut down their VSI and start it back up. As part of the
shut down, the control plane will detect that the GPU is failed and will isolate the GPU from future
reservations.

The data on the GPUs is generally volatile. Losing a given node is impactful, but persistent storage
such as block volume or file share will not be impacted.

#### 3.2.6.2 Business Continuity Disaster Recovery (BCDR)

There are no new BCDR items for the L4 Accelerator.

### 3.2.7 Hardware Platform Support

GPU nodes are marked in the m-zone.yaml as `gpu: true` to indicate that they have GPUs. While this
can be figured out dynamically, this flag allows the fleet / release engineering team to identify if
they wish to release a server as GPU capable or not. This is needed for capacity management
purposes.

There will also be a `profileClass` attribute set on the nodes in the m-zone yaml and an
`allowed_profile_classes` attribute set on the profiles in the global catalog. The profile class
will be set to `gx3-l4` for Sapphire Rapids PCIe Accelerator nodes, and the instance profiles will
list this classes in the allowed list. This will allow for the fleet metrics to better surface what
types of nodes are available for GPUs and will allow for better limiting what profiles end up on
what type of nodes. This concept isn't unique to the GPU nodes and profiles, and just uses the
existing support.

For installation and bring up, there will be a `class` attribute set in the allocations yaml files
to identify the qualified configuration that is deployed. For the L4 PCIe Accelerator, the qualified
configuration will be `gx3-l4-pcie` to designate a Sapphire Rapids Accelerator node installed with a
L4 GPU card.

Example vsi-host/gx3-l4-pcie.yaml qualified-configuration:

```yaml
name: gx3-l4-pcie
server_configurations:
  - id: SR650
---
gpu_configurations:
  - id: "l4-pcie-24gb"
    quantity: 2
    firmware:
      baseline: "TBD"
---
zone_config:
  - section: compute_node
    preferred: True
    role: vpc_compute
    workload: vpc_hypervisor
    nic_mode: all
    node_config:
      arch: amd64
      compute: true
      edge: false
      localdisk: true
      gpu: true
      profileClass: gx3-l4
```

Fleet's APIs have been updated to return the GPU information on a per node basis.

### 3.2.8 Component Level Development

Areas of development and Focals

| Area                 | Focal                   |
| -------------------- | ----------------------- |
| Compute              | Joel Belog              |
| Fleetman             | Malcolm Allen-Ware      |
| Hardware             | Rick Welp, Kirk Douglas |
| Support              | Majad Ahmed             |
| Telemetry BSS        | Michael Harrison        |
| QA Billing           | Michael Indish          |
| End-to-End Testing   | Emily Metruck           |

### 3.2.9 Network Topology and Segmentation Impact

No changes required for Network Topology

# 4. Testing

Testing of GPU nodes is comprised of a different levels.

At the development level, VSI startup, orchestration, and makeup of the instance is verified.
Further, for GPUs, the gpu capabilities are tested within the VSI by installing and testing GPU
specific drivers and test software.

Next, the Quality Engineering End-to-end team utilizes and extends existing GPU test suites to
ensure that the quality of the product delivered meets expectations and that coverage is maintained
to prevent issues further in the lifecycle. There the primary tests covered:

- GPU Basic Tests: validate core features
- GPU Life Cycle Tests: validate life cycle features
- GPU Validation Tests: validate CUDA drivers, nvme drivers
- GPU Memory Flush Test: validate memory was cleared
- GPU Activity Monitor testing
- GPU GhosT validation

For watsonx, an additional RedHat CoreOS image will be used for the GPU Validation Tests. This is
the primary OS used by the watsonx team.

## 4.1 CI Pipelines - Automation

No new controller in RIAS as well as Zonal layer. No addition for modification to CI Pipelines.

# 5. Solution Space

## 5.1 Lessons Learned

## 5.2 Alternatives Considered

### 5.2.1 Non secure boot hostOS

This SRB proposal originally asked for a PCE allowing L4 GPU compute nodes not to use HostOS secure boot.
The PCE has been deemed not necessary as there are no current controls requiring secure boot to be
supported and available on all hosts. The following points explain why GPU containing hosts can not
currently support secure boot.

While L4 VSI getting deprovisioned, it is necessary to clean the GPU device memory as the compute
nodes are shared and multi-tenant. The hostOS secure boot on L4 compute node cut the GPU memory
cleaning privilege, that is accessing BAR0 PCIe card address and resetting the memory to get zero values.

Considering hostOS secure boot and GPU memory clearing while VSI de-provisioning, the GPU memory
clearing requirement is a must have feature in the public cloud offering.
The HostOS/compute node secure boot is a general requirement for cloud operator to put one more level
of fence to protect the hostOS kernel/driver binaries produced from unknown source. And also to
protect the compute host from potential guest attacks on HostOS/kernel/hypervisor.

HostOS secure boot is one of the approach to protect the integrity of the host/compute
node boot kit binaries. A [PCE](https://github.ibm.com/cloud-security-reviews/exceptions/blob/master/is.vpc/PCE-VPC-000183.md)
was obtained in the past for not having secure boot on HostOS. This PCE is closed since there is no
security and compliance control requirement to implement secure boot HostOS.

The GPU device memory clearing by PCI BAR0 access is a general method to reset the PCI devices.
Once the L4 GPU vendor brings a right method via kernel enabled procedure, the L4 GPU compute nodes
will be enabled with secure boot.

## 5.3 Competitive Offerings

AWS, GCP, Azure, and Oracle all have broad GPU offerings.

# 6. Known Issues

# 7. Patents

# 8. Meta

# 9. Glossary

# 10. References

- [GPU Integration into VPC Gen 2 for x86 SRB](https://github.ibm.com/cloudlab/srb/tree/master/proposals/1292)
  - Enablement of NVIDIA V100 16GB PCI cards
- [original GPU SRB](https://github.ibm.com/cloudlab/srb/tree/master/proposals/356)
  - Superseded by SRB 1292 above
- [Instance Storage](https://github.ibm.com/cloudlab/srb/tree/master/proposals/787)
  - GPUs use the DeviceChunk model introduced by Instance Storage
- [Inference Infrastructure Enablement - NVIDIA A100 80GB PCI](https://github.ibm.com/cloudlab/srb/tree/master/proposals/4127)
- [H100 SRB](https://github.ibm.com/cloudlab/srb/tree/3978-h100-inception/proposals/3978)
  - H100 Inception SRB

<!-- Emacs formatting setup -->
<!-- Local Variables: -->
<!-- fill-column: 100 -->
<!-- End: -->

<!-- Vim formatting setup. Must be in first or last 5 lines by default -->
<!-- vim: set textwidth=100 : -->
