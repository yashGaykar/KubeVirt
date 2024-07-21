# KubeVirt Explained: How to Deploy and Manage Virtual Machines on Kubernetes
# Simplifying VM Management on Kubernetes with KubeVirt

Kubernetes is a powerful platform for running and managing containerized applications, but still many organizations need traditional virtual machines for certain tasks. KubeVirt is a tool that lets Kubernetes run virtual machines (VMs) alongside containers, combining the best features of both.

With KubeVirt, we can manage VMs using the same Kubernetes tools and workflows that we use for containers. This integration allows us to benefit from Kubernetes robust scheduling, scaling, and lifecycle management capabilities for both containers and VMs.

In this blog, we'll explore how to deploy VMs on Kubernetes using KubeVirt, providing a step-by-step guide to get us started.


# Prerequisites
Before we begin, we must have:

* A running Kubernetes cluster. We can set up a cluster using tools like [minikube](https://kubernetes.io/docs/tasks/tools/#minikube) or any other Kubernetes provider
* [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl) configured to manage our Kubernetes cluster.
* Basic familiarity with Kubernetes concepts like pods, deployments, and services.


# Kube Virt Setup
First, we will select the latest version of KubeVirt and set the environment variable KUBEVIRT_LATEST_VERSION for future commands.
```sh
KUBEVIRT_LATEST_VERSION=$(curl -s https://api.github.com/repos/kubevirt/kubevirt/releases/latest | awk -F '[ \t":]+' '/tag_name/ {print $3}')
```

# Install KubeVirt with latest version
This command sets up a "kubevirt" namespace, installs Custom Resource Definitions for KubeVirt, and configures an operator to start the KubeVirt installation when a configuration resource is detected.
```
kubectl create -f https://github.com/kubevirt/kubevirt/releases/download/${KUBEVIRT_LATEST_VERSION}/kubevirt-operator.yaml
```

Now Here We will be adding the Kubevirt Custom Resource (named kubevirt). This will prompt the KubeVirt operator to install the remaining components of KubeVirt.
```
kubectl create -f https://github.com/kubevirt/kubevirt/releases/download/${KUBEVIRT_LATEST_VERSION}/kubevirt-cr.yaml 
```

NOTE: Working with minikube, direct access to local virtualization hardware might not be available. we may need to enable emulation mode.
```
kubectl -n kubevirt patch kubevirt kubevirt --type=merge --patch '{"spec":{"configuration":{"developerConfiguration":{"useEmulation":true}}}}'
```

# Install the KubeVirt client, virtctl.
Until we complete the setup, we can download the virtctl command line client for KubeVirt.
```
curl -Lo virtctl https://github.com/kubevirt/kubevirt/releases/download/${KUBEVIRT_LATEST_VERSION}/virtctl-${KUBEVIRT_LATEST_VERSION}-linux-amd64
```
To ensure the downloaded binary is executable, execute the following command:
```
chmod +x virtctl
```

Put the virtctl binary in a directory included in your shell's PATH
```
mv virtctl $HOME/.local/bin
mv virtctl /usr/local/bin
```

_Wait for KubeVirt to finish deploying._ 
We can check its deployment status by examining the phase of the Kubevirt Custom Resource (CR). Use the following command:
```
kubectl -n kubevirt get kubevirt
```

Once KubeVirt fully deploys, it will show:
```
NAME      AGE   PHASE
kubevirt  1m    Deployed
```

To get more details and pods deployed we can use
```
kubectl get pods -n kubevirt
```
Once the deployment completes, confirm all necessary Pods are running without issues.

# Launch a Instance

To create a virtual machine, install the VM manifest using the following command:
```
kubectl apply -f https://kubevirt.io/labs/manifests/vm.yaml
```
NAME_OF_VIRTUAL_MACHINE in above file is 'testvm'

Check the status of the virtual machine we just created:
```
kubectl get vm
```
wait until it gets RUNNING

If the VM is currently in a stopped state. We can use virtctl to change its state to running:
```
virtctl start testvm
```

If Failed we can see the detailed status using
```
kubectl describe vm testvm
```
We will see a message that the VM was scheduled to start.

When we create a VirtualMachine, it automatically generates a VirtualMachineInstance, similar to how a Deployment manages Pods. This chain of actions eventually creates a Pod. We can view all these entities using a single 'kubectl get' command.

#  Connect to VM
 Once the virtual machine is running, we can connect to its console using virtctl:
 ```
 virtctl console testvm
```

To Exit press **CTRL** and **"]"** keys together


# To stop and delete a virtual machine:

Stop the virtual machine using virtctl from outside the VM:
```
virtctl stop testvm
```
To delete the virtual machine
```
kubectl delete virtualmachine testvm
```

# Persistent Storage for Virtual Machines:
For persistent storage in virtual machines, the KubeVirt project offers a solution through the Containerized Data Importer (CDI) add-on. CDI facilitates the import of virtual machine disk images into PersistentVolumeClaims (PVCs), providing a declarative approach to managing persistent storage.

# Create a DataVolume:
CDI introduces a CustomResourceDefinition (CRD) called DataVolume (DV), which abstracts the creation and population of disk data onto PVCs in a Kubernetes cluster. Once a DV is created, CDI handles the process of copying a disk image from the specified source into a PVC. Various source types are supported, including external targets like images served over HTTP, internal sources like other PVCs, or container disks in a registry. Below is an example DataVolume that imports a CirrOS image into a PVC, enabling the underlying VM to have persistent storage across reboots.

# Install CDI
Get the Latest Version
```
export VERSION=$(curl -Ls https://github.com/kubevirt/containerized-data-importer/releases/latest | grep -m 1 -o "v[0-9]\.[0-9]*\.[0-9]*")
```

Install cdi
```
kubectl create -f https://github.com/kubevirt/containerized-data-importer/releases/download/$VERSION/cdi-operator.yaml
```

To trigger the deployment of CDI, create a CDI Custom Resource (CR). This will prompt the operator to deploy CDI.
```
kubectl create -f https://github.com/kubevirt/containerized-data-importer/releases/download/$VERSION/cdi-cr.yaml
```

Check the status and wait until it gets Deployed
```
kubectl -n cdi get cdi

NAME   AGE   PHASE
cdi    45h   Deployed   
```

Once Deployed we are Ready to Work Further
# Create a Data Volume from the required image:

ubuntu-dv.yaml
```
---                                                             
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: ubuntu
spec:
  source:
    http:
      url: https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img
  pvc:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 3Gi
```
Get it Executed using the following command
We can change the  url and storage according to our requirement
```
kubectl create -f ubuntu-dv.yaml
```

To check the DV status:
```
kubectl get dv
```

# Create a Instance from the Data Volume

Relace HASHED_PASSWORD, SSH_PUB_KEY with its values in the file below

ubuntu-vm.yaml
```
---
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  labels:
    kubevirt.io/os: linux
  name: ubuntu-vm
spec:
  running: true
  template:
    metadata:
      creationTimestamp: null
      labels:
        kubevirt.io/domain: ubuntu-vm
    spec:
      domain:
        cpu:
          cores: 2
        devices:
          disks:
          - disk:
              bus: virtio
            name: disk0
          - cdrom:
              bus: sata
              readonly: true
            name: cloudinitdisk
        resources:
          requests:
            memory: 512M
      volumes:
      - name: disk0
        persistentVolumeClaim:
          claimName: ubuntu
      - cloudInitNoCloud:
          userData: |
            #cloud-config
            users: 
              - default
              - name: yash
                passwd: HASHED_PASSWORD
                lock_passwd: false
                shell: /bin/bash
                ssh_pwauth: True
                chpasswd: { expire: False}
                sudo: ALL=(ALL) NOPASSWD:ALL
                groups: users, admin
                ssh_authorized_keys:
                  - ssh-rsa SSH_PUB_KEY
        name: cloudinitdisk
```
Get it Executed
```
kubectl create -f ubuntu-vm.yaml
```

We can get the created Resources:
```
kubectl get dv,pvc,vm
```

Once the status of vm becomes "RUNNING" and Ready is "True",

We could use the command to access the instance
```
virtctl console VM-NAME
```

If needs to connect through VNC
```
virtctl vnc VM-NAME
```



#
I have just explained this with an example for an Ubuntu VM. Similarly, you can apply the same process for other operating systems.
#

# Conclusion

The deployment of various operating system (OS) images using KubeVirt on Kubernetes demonstrates significant differences in setup times and resource requirements. From lightweight options like CirrOS to more substantial distributions such as CentOS and Ubuntu, each OS image offers unique benefits and considerations for deployment:

* [CirrOS](https://quay.io/repository/kubevirt/cirros-container-disk-demo): A lightweight server image of 15MB, typically ready in approximately 50 seconds, making it ideal for quick provisioning and testing.

* [CentOS](https://cloud.centos.org/centos/9-stream/x86_64/images/CentOS-Stream-GenericCloud-9-20240115.0.x86_64.qcow2): A robust server image at 1014 MB, requiring approximately 8 minutes for deployment, suited for enterprise applications and services.

* [Ubuntu Server](https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img): With an image size of 613 MB, deployment completes in around 7 minutes, balancing performance and resource efficiency.

* [Ubuntu Server](https://cloud-images.ubuntu.com/minimal/releases/focal/release-20231020/ubuntu-20.04-minimal-cloudimg-amd64.img) (Minimal Configuration): A lighter variant at 265 MB, ready in about 97 seconds, emphasizing rapid deployment for streamlined environments.

In conclusion, KubeVirt extends Kubernetes to include virtual machines (VMs) alongside containers, offering a unified platform for managing diverse workloads. By leveraging KubeVirt, organizations can seamlessly integrate VMs into their Kubernetes environments, benefiting from enhanced flexibility, efficiency, and compatibility. Whether for running legacy applications, facilitating hybrid cloud deployments, or supporting specialized workloads, KubeVirt bridges the gap between traditional virtualization and modern container orchestration, empowering teams to optimize their infrastructure effectively.
