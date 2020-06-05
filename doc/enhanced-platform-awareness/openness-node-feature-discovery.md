```text
SPDX-License-Identifier: Apache-2.0
Copyright (c) 2019 Intel Corporation
```

# Node Feature Discovery support in OpenNESS

- [Node Feature Discovery support in OpenNESS](#node-feature-discovery-support-in-openness)
  - [Overview of NFD and Edge usecase](#overview-of-nfd-and-edge-usecase)
  - [Details](#details)
    - [Node Feature Discovery support in OpenNESS Network Edge](#node-feature-discovery-support-in-openness-network-edge)
      - [Usage](#usage)
    - [Node Feature Discovery support in OpenNESS On Premises](#node-feature-discovery-support-in-openness-on-premises)
      - [Usage](#usage-1)
  - [Reference](#reference)

## Overview of NFD and Edge usecase

COTS Platforms used for edge deployment come with many features that enable workloads take advantage of, to provide better performance and meet the SLA. When such COTS platforms are deployed in a cluster as part of a Cloudnative deployment it becomes important to detect the hardware and software features on all nodes that are part of that cluster. It should also be noted that some of the nodes might have special accelerator hardware like FPGA, GPU, NVMe, etc.

Let us consider an edge application like CDN that needs to be deployed in the cloud native edge cloud. It would be favorable for a Container orchestrator like Kubernetes to detect the nodes that have CDN friendly hardware and software features like NVMe, media extensions and so on.

Now let us consider a Container Network Function (CNF) like 5G gNb that implements L1 5G NR base station. It would be favorable for the Container orchestrator like Kubernetes to detect nodes that have hardware and software features like FPGA acceleration for Forward error correction, Advanced vector instructions to implement math functions, real-time kernel and so on.

OpenNESS supports the discovery of such features using Node Feature Discovery (NFD). NFD is a Kubernetes add-on that detects and advertises hardware and software capabilities of a platform that can, in turn, be used to facilitate intelligent scheduling of a workload. Node Feature Discovery is one of the Intel technologies that supports targeting of intelligent configuration and capacity consumption of platform capabilities. NFD runs as a separate container on each individual node of the cluster, discovers capabilities of the node, and finally, publishes these as node labels using the Kubernetes API. NFD only handles non-allocatable features.

Some of the Node features that NFD can detect include:

![Sample NFD Features](nfd-images/nfd1.png)
![Sample NFD Features](nfd-images/nfd2.png)
_Figure - Sample NFD Features_

At its core, NFD detects hardware features available on each node in a Kubernetes cluster, and advertises those features using node labels.

NFD consists of two software components:

1) nfd-master is responsible for labeling Kubernetes node objects
2) nfd-worker detects features and communicates them to the nfd-master. One instance of nfd-worker should be run on each node of the cluster

The figure below illustrates how the CDN application will be deployed on the right platform when NFD is utilized, where the required key hardware like NVMe and AVX instruction set support is available.

![CDN app deployment with NFD Features](nfd-images/nfd0.png)

_Figure - CDN app deployment with NFD Features_

> Non-Volatile Memory Express* is a scalable non-volatile memory host interface that can help increase efficiency and reduce latency, while providing high speed access to storage media connected over PCIe.  NVMe overcomes SAS/SATA SSD performance limitations by optimizing hardware and software to take full advantage of NVM SSD technology

> AVX CPUID Features: Intel® Advances Vector Extensions 512 (Intel® AVX 512)

> UEFI Secure Boot: Boot Firmware verification and authorization of OS Loader/Kernel  components

## Details

### Node Feature Discovery support in OpenNESS Network Edge

Node Feature Discovery is enabled by default. It does not require any configuration or user input. It can be disabled by changing the `ne_nfd_enable` variable to `false` in the `group_vars/all/10-default.yml` before OpenNESS installation.

Connection between nfd-workers and nfd-master is secured by certificates generated before running nfd pods.

Node Feature Discovery is deployed in OpenNESS using Helm chart downloaded from [container-experience-kits](https://github.com/intel/container-experience-kits/tree/master/roles/nfd-install/charts/node-feature-discovery) repository.

#### Usage

NFD is working automatically and does not require any user action to collect the features from nodes. Features found by NFD and labeled in Kubernetes can be shown by command: `kubectl get no -o json | jq '.items[].metadata.labels'`.

Example output :
```json
{
  "beta.kubernetes.io/arch": "amd64",
  "beta.kubernetes.io/os": "linux",
  "feature.node.kubernetes.io/cpu-cpuid.ADX": "true",
  "feature.node.kubernetes.io/cpu-cpuid.AESNI": "true",
  "feature.node.kubernetes.io/cpu-cpuid.AVX": "true",
  "feature.node.kubernetes.io/cpu-cpuid.AVX2": "true",
  "feature.node.kubernetes.io/cpu-cpuid.FMA3": "true",
  "feature.node.kubernetes.io/cpu-cpuid.HLE": "true",
  "feature.node.kubernetes.io/cpu-cpuid.RTM": "true",
  "feature.node.kubernetes.io/cpu-rdt.RDTCMT": "true",
  "feature.node.kubernetes.io/cpu-rdt.RDTL3CA": "true",
  "feature.node.kubernetes.io/cpu-rdt.RDTMBM": "true",
  "feature.node.kubernetes.io/cpu-rdt.RDTMON": "true",
  "feature.node.kubernetes.io/iommu-enabled": "true",
  "feature.node.kubernetes.io/kernel-config.NO_HZ": "true",
  "feature.node.kubernetes.io/kernel-config.NO_HZ_FULL": "true",
  "feature.node.kubernetes.io/kernel-config.PREEMPT": "true",
  "feature.node.kubernetes.io/kernel-version.full": "3.10.0-957.21.3.rt56.935.el7.x86_64",
  "feature.node.kubernetes.io/kernel-version.major": "3",
  "feature.node.kubernetes.io/kernel-version.minor": "10",
  "feature.node.kubernetes.io/kernel-version.revision": "0",
  "feature.node.kubernetes.io/memory-numa": "true",
  "feature.node.kubernetes.io/network-sriov.capable": "true",
  "feature.node.kubernetes.io/pci-0300_102b.present": "true",
  "feature.node.kubernetes.io/system-os_release.ID": "centos",
  "feature.node.kubernetes.io/system-os_release.VERSION_ID": "7",
  "feature.node.kubernetes.io/system-os_release.VERSION_ID.major": "7",
  "feature.node.kubernetes.io/system-os_release.VERSION_ID.minor": "",
  "kubernetes.io/arch": "amd64",
  "kubernetes.io/hostname": "edgenode-kubeovn",
  "kubernetes.io/os": "linux",
  "node-role.kubernetes.io/worker": "worker"
}
```

To specify which features should be available by node at deploying pod time, `nodeSelector` field should be defined in pod `.yaml` file. Example pod `yaml` file:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    env: test
  name: golang-test
spec:
  containers:
    - image: golang
      name: go1
  nodeSelector:
    feature.node.kubernetes.io/cpu-pstate.turbo: 'true'
```

### Node Feature Discovery support in OpenNESS On Premises

Node Feature Discovery is enabled by default. It does not require any configuration or user input. It can be disabled by changing the `onprem_nfd_enable` variable to `false` in the `group_vars/all/10-default.yml` before OpenNESS installation.

NFD service in OpenNESS On Premises consists of two software components:

- *nfd-worker*, which is taken from https://github.com/kubernetes-sigs/node-feature-discovery (downloaded as image)
- *nfd-master*: stand alone service run on Edge Controller.

Nfd-worker connects to nfd-master server. Connection between nfd-workers and nfd-master is secured by TLS based certificates used in Edge Node enrollment: nfd-worker uses certificates of Edge Node, nfd-master generates certificate based on Edge Controller root certificate. Nfd-worker provides hardware features to nfd-master which stores that data to the controller mysql database. It can be used then as EPA Feature requirement while defining and deploying app on node.

#### Usage

NFD is working automatically and does not require any user action to collect the features from nodes.
Default version of nfd-worker downloaded by ansible scripts during deployment is v.0.5.0. It can be changed by setting variable `_nfd_version` in `roles/nfd/onprem/worker/defaults/main.yml`.

Features found by NFD are visible in Edge Controller UI in node's NFD tab. While defining edge application (Controller UI->APPLICATIONS->ADD APPLICATION), `EPA Feature` fields can be used as definition of NFD requirement for app deployment. Eg: if application requires Multi-Precision Add-Carry Instruction Extensions (ADX), user can set EPA Feature Key to `nfd:cpu-cpuid.ADX` and EPA Feature Value to `true`.

![Sample application with NFD Feature required](nfd-images/nfd3_onp_app.png)

Deployment of such application will fail for nodes that don't provide this feature with this particular value. List of features supported by nfd-worker service can be found: https://github.com/kubernetes-sigs/node-feature-discovery#feature-discovery. Please note that `nfd:` prefix always has to be added when used as EPA Feature Key.

## Reference
More details about NFD can be found here: https://github.com/Intel-Corp/node-feature-discovery
