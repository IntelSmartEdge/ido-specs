```text
SPDX-License-Identifier: Apache-2.0
Copyright (c) 2020 Intel Corporation
```
<!-- omit in toc -->
# VM support in OpenNESS for Network Edge - Setup, deployment, and management considerations
- [Overview](#overview)
- [KubeVirt](#kubevirt)
- [Stateless vs Stateful VMs](#stateless-vs-stateful-vms)
  - [VMs with ephemeral storage](#vms-with-ephemeral-storage)
  - [VMs with persistent Local Storage](#vms-with-persistent-local-storage)
  - [VMs with Cloud Storage](#vms-with-cloud-storage)
  - [Creating Docker image for stateless VM](#creating-docker-image-for-stateless-vm)
- [Enabling in OpenNESS](#enabling-in-openness)
- [VM deployment](#vm-deployment)
  - [Stateless VM deployment](#stateless-vm-deployment)
  - [Stateful VM deployment](#stateful-vm-deployment)
  - [VM deployment with SRIOV NIC support](#vm-deployment-with-sriov-nic-support)
  - [VM snapshot](#vm-snapshot)
- [Limitations](#limitations)
  - [Cloud Storage](#cloud-storage)
  - [Storage Orchestration and PV/PVC management](#storage-orchestration-and-pvpvc-management)
  - [Snapshot Creation](#snapshot-creation)
- [Useful Commands and Troubleshooting](#useful-commands-and-troubleshooting)
  - [Commands](#commands)
  - [Troubleshooting](#troubleshooting)
- [Helpful Links](#helpful-links)

## Overview

This document explores the support of `VM` (Virtual machine) deployment in OpennNESS for Network Edge. Items covered include but are not limited to: design considerations and currently available solutions, limitations, configuration of the environment for OpenNESS, deployment of VMs with various requirements for storage and SRIOV support, and lastly troubleshooting.

When designing support for VM deployment for Network Edge, a key consideration was how this support will fit into a `K8s` (Kubernetes) based solution. Two popular open source projects allowing VM deployments within a K8s environment were identified; `KubeVirt` and `Virtlet`, both of these projects support deployment of the VMs running inside pods.

Virtlet VMs are treated as ordinary pods and can be controlled from `kubectl` natively but the deployment requires the introduction of an additional CRI (Container Runtime Interface) and CRI proxy into the cluster. In comparison KubeVirt VMs are enabled by extending the functionality of K8s via CRDs (Custom Resource Definition) and easy to deploy KubeVirt agents and controllers - no CRI proxy introduction is necessary in the cluster.

Due to easy deployment and no impact on the existing OpenNESS architecture, KubeVirt is the solution of choice in OpenNESS.

## KubeVirt

`KubeVirt` is an open source project extending K8s with VM support via the previously mentioned CRDs, agents and controllers. It addresses a need to allow non-containerizable applications/workloads inside VMs to be treated as K8s managed workloads. This allows for both VM and Container/Pod applications to coexist within a shared K8s environment, allowing for communication between the Pods, VMs, and services on same cluster. KubeVirt provides a command line utility (`virtctl`) which allows for management of the VM (Create, start, stop, etc). Additionally, it provides a `CDI` (Containerized Data Importer) utility which allows for loading existing `qcow2` VM images (and other data) into the created VM. Support for the K8s `CNI` (Container Network Interface) plugin Multus and SRIOV is also provided which allows to attach `NICs` (Network Interface Card) `VFs` (Virtual Functions) to a deployed VM. More information about KubeVirt can be found on the [official website](https://kubevirt.io/) and [github](https://github.com/kubevirt/kubevirt).

## Stateless vs Stateful VMs

The types of VM deployments can be split into two categories based on the storage required by the workload. Stateless VMs are backed by ephemeral storage - meaning that the data will disappear with VM restart/reboot. Stateful VMs are backed by persistent storage - meaning that data will persist after VM restart/reboot. The type of storage required will be determined based on the use-case. In OpenNESS support for both types of VM is available with aid of KubeVirt.

### VMs with ephemeral storage
These are VMs with ephemeral storage, such as ContainerDisk storage that would be deployed in similar manner to ordinary container pods. The data contained in the VM would be erased on each VM deletion/restart thus it is suitable for stateless applications running inside the pods. Although usually a better fit for such application would be running the workload in container pod, for various reasons such as a Legacy application a user may not want to containerize their workload. The advantage of this deployment from an K8s/OpenNESS perspective is that there is no additional storage configuration and the user only needs to have a cluster with KubeVirt deployed and a dockerized version of a VM image in order to spin up the VM.

### VMs with persistent Local Storage
These are VMs which require persistent storage, the data of this kind of VM stays persistent between restarts of the VM. In case of local storage, Kubernetes provides ‘local volume plugin' which can be used to define a `PV` (Persistent Volume). In the case of the local volume plugin there is no support for dynamic creation (k8s 1.17.0) and the PVs must be created by a user before a Persistent Volume Claim (`PVC`) can be claimed by the pod/VM. This manual creation of PVs must be taken into consideration for an OpenNESS deployment as a PV will need to be created per each VM per each node as the storage is local to the Edge Node. In case of persistent storage the VM image must be loaded to the PVC, this can be done via use of KubeVirt's Container Data Importer (CDI). This kind of storage is meant for use with stateful workloads, and being local to the node is suitable for Edge deployments.

### VMs with Cloud Storage
Support for persistent storage via Cloud Storage providers is possible in K8s but it is currently not supported in OpenNESS. There are no plans to support it from OpenNESS right now - this may change in the future depending on demand.

### Creating Docker image for stateless VM
In order to create a Docker image for a stateless VM, the VM image needs to be wrapped inside the Docker image. In order to do this the user needs to create a `Dockerfile` and place the VM image in the same directory, then build the Docker image as per the example below:
```
#Dockerfile
FROM scratch
ADD CentOS-7-x86_64-GenericCloud.qcow2 /disk/
```
```shell
docker build -t centosimage:1.0 .
```
## Enabling in OpenNESS

The KubeVirt role responsible for bringing up KubeVirt components is enabled by default in the OpenNESS experience kit via Ansible automation. In this default state it does not support SRIOV in a VM so additional steps are required to enable it. The following is a complete list of steps to bring up all components related to VM support in Network Edge. VM support also requires Virtualization and VT-d to be enabled in BIOS of the Edge Node.

 1. Configure Ansible for KubeVirt:
    KubeVirt is deployed by default but to provide SR-IOV support configure following settings:
      - Enable kubeovn CNI and SRIOV:
         ```yaml
         # group_vars/all/10-default.yml
         kubernetes_cnis:
         - kubeovn
         - sriov
         ```
      - Enable SRIOV for KubeVirt:
          ```yaml
          # group_vars/all/10-default.yml

          # SR-IOV support for kube-virt based Virtual Machines
          sriov_kubevirt_enable: true
          ```
      - Enable necessary Network Interfaces with SRIOV:
          ```yaml
          # host_vars/node01.yml
          sriov:
            network_interfaces: {<interface_name>: 1}
          ```
      - Set up the maximum number of stateful VMs and directory where the Virtual Disks will be stored on Edge Node:
          ```yaml
          # group_vars/all/10-default.yml
          kubevirt_default_pv_dir: /var/vd/
          kubevirt_default_pv_vol_name: vol
          kubevirt_pv_vm_max_num:  64
          ```
 2. Set up other common configurations for the cluster and enable other EPA features as needed and deploy the cluster using the `deploy_ne.sh` script in OpenNESS experience kit top level directory.

 3. On successful deployment the following pods will be in running state:
    ```shell
        [root@controller ~]# kubectl get pods -n kubevirt

        kubevirt      virt-api-684fdfbd57-9zwt4                 1/1     Running
        kubevirt      virt-api-684fdfbd57-nctfx                 1/1     Running
        kubevirt      virt-controller-64db8cd74c-cn44r          1/1     Running
        kubevirt      virt-controller-64db8cd74c-dbslw          1/1     Running
        kubevirt      virt-handler-jdsdx                        1/1     Running
        kubevirt      virt-operator-c5cbfb9ff-9957z             1/1     Running
        kubevirt      virt-operator-c5cbfb9ff-dj5zj             1/1     Running

        [root@controller ~]# kubectl get pods -n cdi

        cdi           cdi-apiserver-5f6457f4cb-8m9cb            1/1     Running
        cdi           cdi-deployment-db8c54f8d-t4559            1/1     Running
        cdi           cdi-operator-7796c886c5-sfmsb             1/1     Running
        cdi           cdi-uploadproxy-556bf8d455-f8hn4          1/1     Running
    ```

## VM deployment
Provided below are sample deployment instructions for different types of VMs.
Please use sample `.yaml` specification files provided in OpenNESS Edge Controller repo - [edgecontroller/kubevirt/examples/](https://github.com/otcshare/edgecontroller/tree/master/kubevirt/examples) in order to deploy the workloads - some of the files will require modification in order to suit the environment they will be deployed in, specific instructions on modifications are provided in steps below.

### Stateless VM deployment
To deploy a sample stateless VM with containerDisk storage:

  1. Deploy VM
      ```shell
      [root@controller ~]# kubectl create -f /opt/edgecontroller/kubevirt/examples/statelessVM.yaml
      ```
  2. Start the VM:
      ```shell
      [root@controller ~]# kubectl virt start cirros-stateless-vm
      ```
  3. Check that the VM pod got deployed and VM is deployed:
      ```shell
      [root@controller ~]# kubectl get pods | grep launcher
      [root@controller ~]# kubectl get vms
      ```
  4. Execute into the VM (pass/login cirros/gocubsgo):
      ```shell
      [root@controller ~]# kubectl virt console cirros-stateless-vm
      ```
  5. Check that IP address of OpenNESS/K8s overlay network is available in VM.
      ```shell
      [root@vm ~]# ip addr
      ```

### Stateful VM deployment
To deploy a sample stateful VM with persistent storage and additionally use Generic Cloud CentOS image which requires user to initially log in with ssh key instead of login/password over ssh:

> Note: Please note that each stateful VM with a new `PVC` (Persistent Volume Claim) requires a new `PV` (Persistent Volume) to be created - see more in [limitations section](#limitations). Also note that CDI needs two PVs when creating a PVC and loading VM image from qcow2 file, one PV for the actual PVC to be created and one PV to translate the qcow2 image to raw input.

  1. Create a persistent volume for the VM:

      - Edit the sample yaml with hostname of the worker node:
         ```yaml
         # /opt/edgecontroller/kubevirt/examples/persistentLocalVolume.yaml
         # For both kv-pv0 and kv-pv1 enter correct hostname
         - key: kubernetes.io/hostname
                  operator: In
                  values:
                  - <nodeName>
         ```
      - Create the PV:
         ```shell
         [root@controller ~]# kubectl create -f /opt/edgecontroller/kubevirt/examples/persistentLocalVolume.yaml
         ```
      - Check that PV is created:
         ```shell
         [root@controller ~]# kubectl get pv
         NAME   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS    REASON   AGE
         kv-pv0    15Gi       RWO            Retain           Available           local-storage            7s
         kv-pv1    15Gi       RWO            Retain           Available           local-storage            7s
         ```
  2. Download the Generic Cloud qcow image for CentOS 7
      ```shell
      [root@controller ~]# wget https://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud-1907.qcow2
      ```
  3. Get the address of the CDI upload proxy:
      ```shell
      [root@controller ~]# kubectl get services -A | grep cdi-uploadproxy
      ```
  4. Create and upload the image to PVC via CDI
      ```shell
      [root@controller ~]# kubectl virt image-upload dv centos-dv --image-path=/root/kubevirt/CentOS-7-x86_64-GenericCloud-1907.qcow2 --insecure --size=15Gi --storage-class=local-storage --uploadproxy-url=https://<cdi-proxy-ip>:443

      DataVolume default/centos-dv created
      Waiting for PVC centos-dv upload pod to be ready...
      Pod now ready
      Uploading data to https://<cdi-proxy-ip>:443

      898.75 MiB / 898.75 MiB [======================================================================================================================================================================================] 100.00% 15s

      Uploading data completed successfully, waiting for processing to complete, you can hit ctrl-c without interrupting the progress
      Processing completed successfully
      Uploading /root/kubevirt/CentOS-7-x86_64-GenericCloud-1907.qcow2 completed successfully
      ```
  5. Check that PV, DV, PVC are correctly created:
      ```shell
      [root@controller ~]# kubectl get pv
       kv-pv0    15Gi       RWO            Retain           Bound      default/centos-dv           local-storage            2m48s
       kv-pv1    15Gi       RWO            Retain           Released   default/   centos-dv-scratch   local-storage            2m48s
      [root@controller ~]# kubectl get dv
      centos-dv   Succeeded              105s
      [root@controller ~]# kubectl get pvc
      centos-dv   Bound    kv-pv0      15Gi       RWO            local-storage   109s
      ```
  6. Create ssh key:
      ```shell
      [root@controller ~]# ssh-keygen
      ```
  7. Get the Controllers public key:
      ```shell
      [root@controller ~]# cat ~/.ssh/id_rsa.pub
      ```
  8. Edit .yaml file for the VM with updated public key:
      ```yaml
          # /opt/edgecontroller/kubevirt/examples/cloudGenericVM.yaml
          users:
                - name: root
                  password: root
                  sudo: ALL=(ALL) NOPASSWD:ALL
                  ssh_authorized_keys:
                    - ssh-rsa <controller-public-key> <user>@<node>
      ```
  9.  Deploy the VM:
      ```shell
      [root@controller ~]# kubectl create -f /opt/edgecontroller/kubevirt/examples/cloudGenericVM.yaml
      ```
  10. Start the VM:
      ```shell
      [root@controller ~]# kubectl virt start centos-vm
      ```
  11. Check that the VM container has deployed:
      ```shell
      [root@controller ~]# kubectl get pods | grep virt-launcher-centos
      ```
  12. Get the IP address of the VM:
      ```shell
      [root@controller ~]# kubectl get vmi
      ```
  13. SSH into the VM:
      ```shell
      [root@controller ~]# ssh <vm_ip>
      ```

### VM deployment with SRIOV NIC support

To deploy a VM requesting SRIOV VF of NIC:
  1. Bind SRIOV interface to VFIO driver on Edge Node:
     ```shell
     [root@worker ~]# /opt/dpdk-18.11.6/usertools/dpdk-devbind.py --bind=vfio-pci <PCI.B.F.ID-of-VF>
     ```
  2. Delete/Restart SRIOV device plugin on the node:
     ```shell
     [root@controller ~]# kubectl delete pod sriov-release-kube-sriov-device-plugin-amd64-<podID> -n kube-system
     ```
  3. Check that the SRIOV VF for VM is available as allocatable resource for DP (wait a few seconds after restart):
     ```
     [root@controller ~]# kubectl get node <worker-node-name> -o json | jq '.status.allocatable'
     {
     "cpu": "79",
     "devices.kubevirt.io/kvm": "110",
     "devices.kubevirt.io/tun": "110",
     "devices.kubevirt.io/vhost-net": "110",
     "ephemeral-storage": "241473945201",
     "hugepages-1Gi": "0",
     "hugepages-2Mi": "2Gi",
     "intel.com/intel_sriov_dpdk": "0",
     "intel.com/intel_sriov_netdevice": "0",
     "intel.com/intel_sriov_vm": "1", <---- This one
     "memory": "194394212Ki",
     "pods": "110"
     }
     ```
  4. Deploy VM requesting SRIOV device (adjust the amount of Hugepages Required in .yaml if a smaller amount available on the platform):
     ```shell
      [root@controller ~]# kubectl create -f /opt/edgecontroller/kubevirt/examples/sriovVM.yaml
      ```
  5. Start VM:
     ```shell
     [root@controller ~]# kubectl virt start debian-sriov-vm
     ```
  6. Execute into VM (login/pass root/toor):
     ```shell
     [root@controller ~]# kubectl virt console debian-sriov-vm
     ```
  7. Fix networking in the VM for Eth1:
     ```shell
     [root@vm ~]# vim /etc/network/interfaces
     # Replace info for Eth1
     # Maybe the VM has 2 NICs?
     auto eth1
     iface eth1 inet static
     address 192.168.100.2
     netmask 255.255.255.0
     network 192.168.100.0
     broadcast 192.168.100.255
     gateway 192.168.100.1


     #restart networking service
     [root@vm ~]# sudo /etc/init.d/networking restart
     ```

  8.  Check the SRIOV interface has an assigned IP address.
      ```shell
      [root@vm ~]# ip addr
      1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
      inet 127.0.0.1/8 scope host lo
        valid_lft forever preferred_lft forever
      inet6 ::1/128 scope host
        valid_lft forever preferred_lft forever
      2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1400 qdisc pfifo_fast state UP group default qlen 1000
      link/ether aa:10:79:10:00:18 brd ff:ff:ff:ff:ff:ff
      inet 10.16.0.23/16 brd 10.16.255.255 scope global eth0
         valid_lft forever preferred_lft forever
      inet6 fe80::a810:79ff:fe10:18/64 scope link
         valid_lft forever preferred_lft forever
      3: eth1: <BROADCAST,MULTICAST> mtu 1500 qdisc mq state DOWN group default qlen 1000
      link/ether 4a:b0:80:d8:a9:b3 brd ff:ff:ff:ff:ff:ff
      inet 192.168.100.2/24 brd 192.168.100.255 scope global eth1
        valid_lft forever preferred_lft forever
      ```

### VM snapshot

Currently support for VM snapshot is allowed by manually taking snapshot of the VMs virtual disk with `QEMU` utility - see more in [limitations](#limitations). In order to restore the snapshot or create a new VM the user is required to copy new qcow2 (snapshot) file to controller and create the VM as per [stateless VM example](#Statefull-VM-deployment) using new qcow2 image instead of the one provided in example. The following are steps to create a snapshot:

  1. Log into the Edge Node
  2. Go to virtual disk directory for previously created VM:
     ```shell
     [root@worker ~]# cd /var/vd/vol0/ && ls
     ```
  3. Create a qcow2 snapshot image out of the virtual disk present in the directory (`disk.img`)
     ```shell
     /opt/qemu-4.0.0/qemu-img convert -f raw -O qcow2 disk.img ./my-vm-snapshot.qcow2
     ```

## Limitations

### Cloud Storage

Currently there is no support for Cloud Storage in OpenNESS.

### Storage Orchestration and PV/PVC management

Currently in Network Edge OpenNESS there is no mechanism to manage storage, the assumption is made that the default HDD/SSD of the Edge Node is used for storage. Additionally, various methods of optimizing storage (ie. by using various file system types etc.) for performance are not in scope of OpenNESS at this time. In Network Edge deployment when using K8s with local persistent volumes, there is a need to create a directory per each PV which will be used to store VMs' virtual disk. The creation of directories to store PV is automated from OpenNESS but the creation of the PV itself and keeping track of which PV corresponds to which VM is currently a responsibility of the user - this is due to the local volume plugin enabling local storage in K8s which does not provide dynamic PV creation when a PVC claim is made. A consideration of how to automate and simplify PV management for the user will be made in the future - an evaluation of currently available solutions is needed.

### Snapshot Creation

Currently snapshot creation of the stateful VM is to be done by the user manually using the QEMU utility. K8s does provide a Volume Snapshot and Volume Snapshot Restore functionality but at time of writing it is only available for out-off tree K8s device storage plugins supporting CSI driver - the local volume plugin used in this implementation is not yet supported as a CSI plugin. A consideration of how to automate and simplify a VM snapshot for the user will be made in the future.

## Useful Commands and Troubleshooting

### Commands

```
kubectl get pv                  # get Persistent Volumes
kubectl get pvc                 # get Persistent Volume Claims
kubectl get dv                  # get Data Volume
kubectl get sc                  # get Storage Classes
kubectl get vms                 # get VM state
kubectl get vmi                 # get VM IP
kubectl virt start <vm_name>    # start VM
kubectl virt restart <vm_name>  # restart VM
kubectl virt stop <vm_name>     # stop VM
kubectl virt pause <vm_name>    # pause VM
kubectl virt console <vm_name>  # Get console connection to VM
kubectl virt help               # See info about rest of virtctl commands
```
### Troubleshooting

1. PVC image not being uploaded through CDI - check that the IP address of cdi-upload-proxy is correct and that the Network Traffic policy for CDI is applied:
   ```shell
   kubectl get services -A | grep cdi-uploadproxy
   kubectl get networkpolicy | grep cdi-upload-proxy-policy
   ```

2. Cannot SSH to stateful VM with Cloud Generic Image due to public key being denied - double check that the public key provided in `/opt/edgecontroller/kubevirt/examples/cloudGenericVM.yaml` is valid and in a correct format. Example of a correct format:
   ```yaml
   # /opt/edgecontroller/kubevirt/examples/cloudGenericVM.yaml
   users:
         - name: root
           password: root
           sudo: ALL=(ALL) NOPASSWD:ALL
           ssh_authorized_keys:
             - ssh-rsa Askdfjdiisd?-SomeLongSHAkey-?dishdxxxx root@controller
   ```
3. Completely deleting a stateful VM - delete VM, DV, PV, PVC and the Virtual Disk related to VM from Edge Node.
   ```shell
   [controller]# kubectl delete vm <vm_name>
   [controller]# kubectl delete dv <dv_name>
   [controller]# kubectl delete pv <pv_names>
   [node]# rm /var/vd/vol<vol_num_related_to_pv>/disk.img
   ```

4. Cleanup script `cleanup_ne.sh` does not properly clean up KubeVirt/CDI components if user has intentionally/unintentionally deleted one of these components outside the script - The KubeVirt/CDI components must be cleaned up/deleted in a specific order to wipe them sucesfully and the cleanup script does that for the user. In an event when user tries to delete the KubeVirt/CDI operator in wrong order the namespace for the component may be stuck indefinately in a `terminating` state. This is not an issue if user runs the script to completely clean the cluster but might be troublesome if user wants to run cleanup for KubeVirt only. In order to fix this user must do the following:

   1. Check which namespace is stuck in `terminating` state:
      ```shell
      [controller]# kubectl get namespace
      NAME              STATUS       AGE
      cdi               Active       30m
      default           Active       6d1h
      kube-node-lease   Active       6d1h
      kube-ovn          Active       6d1h
      kube-public       Active       6d1h
      kube-system       Active       6d1h
      kubevirt          Terminating  31m
      openness          Active       6d1h
      ```

   2. Delete the finalizer for the terminating namespace:
      ```shell
      ##replace instances of `kubevirt` with 'cdi' in the command if CDI is the issue.
      [controller]# kubectl get namespace "kubevirt" -o json  | tr -d "\n" | sed "s/\"finalizers\": \[[^]]\+\]/\"finalizers\": []/" | kubectl replace --raw /api/v1/namespaces/kubevirt/finalize -f -
      ```

   3. Run clean up script for kubeVirt again:
      ```shell
      [controller]# ./cleanup_ne.sh
      ```

## Helpful Links

- [KubeVirt](https://kubevirt.io/)

- [KubeVirt components](https://github.com/kubevirt/kubevirt/blob/master/docs/components.md)
- [Container Storage Interface](https://kubernetes.io/blog/2019/01/15/container-storage-interface-ga/)
- [K8s Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Containerized Data Importer](https://github.com/kubevirt/containerized-data-importer/blob/master/README.md)
- [Local Volume Plugin](https://kubernetes.io/docs/concepts/storage/volumes/#local)
