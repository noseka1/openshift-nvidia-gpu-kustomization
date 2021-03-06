# Kustomization for Deploying NVIDIA GPU Operator to OpenShift

This kustomization uses [NVIDIA GPU Operator](https://github.com/NVIDIA/gpu-operator). The installation process is documented in [OpenShift on NVIDIA GPU Accelerated Clusters](https://docs.nvidia.com/datacenter/kubernetes/openshift-on-gpu-install-guide/index.html).

This kustomization was tested on OCP 4.4.3.

## Prerequisites

OpenShift cluster with GPU-enabled nodes:

* [Amazon EC2 G4 Instances](https://aws.amazon.com/ec2/instance-types/g4/)
* [Amazon EC2 P3 Instances](https://aws.amazon.com/ec2/instance-types/p3/)

## Deploying Node Feature Discovery Operator

Deploy NFD operator using the command:

```
$ oc apply --kustomize nfd-operator/base
```

Verify that the NFD operator was deployed successfully:

```
$ oc get csv --namespace openshift-operators
NAME                     DISPLAY                  VERSION   REPLACES   PHASE
nfd.4.4.0-202004261927   Node Feature Discovery   4.4.0                Succeeded
```
Create an NFD master server instance:

```
$ oc apply --kustomize nfd-instance/base
```

Verify that the NFD pods are running on each of your cluster node:

```
$ oc get ds --namespace openshift-nfd
NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                     AGE
nfd-master   3         3         3       3            3           node-role.kubernetes.io/master=   3m7s
nfd-worker   6         6         6       6            6           node-role.kubernetes.io/worker=   3m7s
```

Verify that the GPU-enabled nodes have been labeled properly:

```
$ oc get nodes --selector feature.node.kubernetes.io/pci-10de.present=true
NAME                                         STATUS   ROLES    AGE   VERSION
ip-10-0-134-147.us-west-2.compute.internal   Ready    worker   75m   v1.17.1
ip-10-0-150-189.us-west-2.compute.internal   Ready    worker   75m   v1.17.1
ip-10-0-167-244.us-west-2.compute.internal   Ready    worker   75m   v1.17.1
```

## Installing RHEL Entitlements on OpenShift Nodes

This section is based on the instructions found at [How to use entitled image builds on Red Hat OpenShift Container Platform 4.x cluster ?](https://access.redhat.com/solutions/4908771) Follow the instructions in the Downloading certificates section of that article in order to obtain a xxxx_certificates.zip archive with certificates.

Insert the base-64 encoded content of the xxxx_certificates.zip/consumer_export.zip/export/entitlement_certificates/xxxxx.pem into the manifest ./rhel-entitlements/base/50-entitlement-key-pem-machineconfig.yaml and ./rhel-entitlements/base/50-entitlement-pem-machineconfig.yaml     .

> :exclamation: Applying the machineconfigs to the cluster will cause all worker nodes to restart.

Apply the machineconfigs to the cluster:

```
$ oc apply --kustomize rhel-entitlements/base
```

Allow some time for the worker nodes to restart. Verify the RHEL entitlements configuration:

```
$ oc create --filename https://raw.githubusercontent.com/openshift-psap/blog-artifacts/master/how-to-use-entitled-builds-with-ubi/0004-cluster-wide-entitled-pod.yaml
```

```
$ oc logs cluster-entitled-build-pod
Updating Subscription Management repositories.
Unable to read consumer identity
Subscription Manager is operating in container mode.
Red Hat Enterprise Linux 8 for x86_64 - AppStre 3.2 MB/s |  18 MB     00:05
Red Hat Enterprise Linux 8 for x86_64 - BaseOS   34 MB/s |  18 MB     00:00
Red Hat Universal Base Image 8 (RPMs) - BaseOS  1.6 MB/s | 760 kB     00:00
Red Hat Universal Base Image 8 (RPMs) - AppStre 1.6 MB/s | 3.8 MB     00:02
Red Hat Universal Base Image 8 (RPMs) - CodeRea  16 kB/s | 9.1 kB     00:00
====================== Name Exactly Matched: kernel-devel ======================
kernel-devel-4.18.0-80.1.2.el8_0.x86_64 : Development package for building kernel modules to match the kernel
kernel-devel-4.18.0-80.el8.x86_64 : Development package for building kernel modules to match the kernel
kernel-devel-4.18.0-80.4.2.el8_0.x86_64 : Development package for building kernel modules to match the kernel
kernel-devel-4.18.0-80.7.1.el8_0.x86_64 : Development package for building kernel modules to match the kernel
kernel-devel-4.18.0-80.11.1.el8_0.x86_64 : Development package for building kernel modules to match the kernel
kernel-devel-4.18.0-147.el8.x86_64 : Development package for building kernel modules to match the kernel
...
```

Remove the test pod:

```
$ oc delete pod cluster-entitled-build-pod
```

## Deploying NVIDIA GPU Operator

Deploy NVIDIA GPU operator using the command:

```
$ oc apply --kustomize nvidia-gpu-operator/base
```

Verify that the NVIDIA GPU operator was deployed successfully:

```
$ oc get csv --namespace gpu-operator-resources
NAME                               DISPLAY                  VERSION    REPLACES   PHASE
gpu-operator-certified.v1.1.5-r6   NVIDIA GPU Operator      1.1.5-r6              Succeeded
nfd.4.4.0-202004261927             Node Feature Discovery   4.4.0                 Succeeded
```

Create a NVIDIA GPU instance:

```
$ oc apply --kustomize nvidia-gpu-instance/base
```

Check the status of the NVIDIA GPU pods:

```
$ oc get pod --namespace gpu-operator-resources
NAME                                       READY   STATUS                 RESTARTS   AGE
gpu-operator-757877f9b-m86kx               1/1     Running                0          3m4s
nvidia-container-toolkit-daemonset-b2wxs   1/1     Running                0          97s
nvidia-container-toolkit-daemonset-bdjz4   1/1     Running                0          97s
nvidia-container-toolkit-daemonset-s748z   1/1     Running                0          97s
nvidia-driver-daemonset-pzpjz              1/1     Running                1          92s
nvidia-driver-daemonset-w7wh8              1/1     Running                1          92s
nvidia-driver-daemonset-xfkzk              1/1     Running                0          92s
nvidia-driver-validation                   0/1     CreateContainerError   0          87s
```

Note that the *nvidia-driver-validation* pod may show “CreateContainerError” for a period of time while the operator is starting up the components. Eventually, the status will change to *Completed*.

Verify that the NVIDIA GPU deployment completed successfully. The state of the *nvidia-gpu* clusterpolicy should change to *Ready*:

```
$ oc get clusterpolicy nvidia-gpu \
    --namespace gpu-operator-resources \
    --output jsonpath='{.status.state}'
Ready
```

## References

* [OpenShift on NVIDIA GPU Accelerated Clusters](https://docs.nvidia.com/datacenter/kubernetes/openshift-on-gpu-install-guide/index.html)
* [NVIDIA GPU Operator](https://github.com/NVIDIA/gpu-operator)
* [Creating a GPU-enabled node with OpenShift 4.2 in Amazon EC2](https://www.openshift.com/blog/creating-a-gpu-enabled-node-with-openshift-4-2-in-amazon-ec2)
* [Simplifying deployments of accelerated AI workloads on Red Hat OpenShift with NVIDIA GPU Operator](https://www.openshift.com/blog/simplifying-deployments-of-accelerated-ai-workloads-on-red-hat-openshift-with-nvidia-gpu-operator)
* [How to install the NVIDIA GPU Operator with OpenShift](https://access.redhat.com/solutions/4908611)
* [How to use entitled image builds on Red Hat OpenShift Container Platform 4.x cluster ?](https://access.redhat.com/solutions/4908771)
