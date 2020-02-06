---
description: >-
  This guide describes the steps to install OpenShift 4 using Libvirt on RHEL 8.
  Disclaimer: This is not supported by Red Hat!
---

# OpenShift 4 IPI KVM install with Libvirt on a Hetzner root server

## Libvirt pre reqs

Follow [Libvirt HOWTO](https://github.com/openshift/installer/tree/master/docs/dev/libvirt).

## Building installer binary

```text
$ go get github.com/openshift/installer
$ cd go/src/github.com/openshift/installer/
$ # choose branch according to version
$ git checkout release-4.3
$ TAGS=libvirt hack/build.sh
```

## Configure and create your cluster

Check [https://openshift-release.svc.ci.openshift.org/](https://openshift-release.svc.ci.openshift.org/) for latest release version

```text
$ env TF_VAR_libvirt_master_memory=16384 TF_VAR_libvirt_master_vcpu=8 \
OPENSHIFT_INSTALL_RELEASE_IMAGE_OVERRIDE=registry.svc.ci.openshift.org/ocp/release:4.3.1 \
openshift-install --dir=libvirt-install --log-level debug \
create install-config
```

Configure the installer for your cluster instance

```text
? Platform libvirt
? Libvirt Connection URI [? for help] (qemu+tcp://192.168.122.1/system) 
? Base Domain targz.it
? Cluster Name ocp4
```

Copy & Paste your Pull Secret from [https://cloud.redhat.com/openshift/](https://cloud.redhat.com/openshift/)

```text
? Pull Secret [? for help] ************.....
```

Before actually creating the cluster we need some tweaking. Edit the number of replicas which get created. In this case we will create 3 master and 2 worker nodes:

```text
$ vi libvirt-install/install-config.yaml
.
.
.
compute:
- hyperthreading: Enabled
  name: worker
  platform: {}
  replicas: 2
controlPlane:
  hyperthreading: Enabled
  name: master
  platform: {}
  replicas: 3
.
.
.
```

Now generate the manifests:

```text
$ env TF_VAR_libvirt_master_memory=16384 TF_VAR_libvirt_master_vcpu=8 \
OPENSHIFT_INSTALL_RELEASE_IMAGE_OVERRIDE=registry.svc.ci.openshift.org/ocp/release:4.3.1 \
openshift-install --dir=libvirt-install --log-level debug \
create manifests
```

Fix the domain of the Ingress object and remove the cluster name \(ocp4 in this case\):

```text
$ cat libvirt-install/manifests/cluster-ingress-02-config.yml
apiVersion: config.openshift.io/v1
kind: Ingress
metadata:
  creationTimestamp: null
  name: cluster
spec:
  domain: apps.ocp4.targz.it
status: {}
$ sed -i.bak '{/domain/s#ocp4\.##;}' libvirt-install/manifests/cluster-ingress-02-config.yml
$ diff -u libvirt-install/manifests/cluster-ingress-02-config.yml*
--- libvirt-install/manifests/cluster-ingress-02-config.yml     2020-02-06 18:25:47.805997481 +0100
+++ libvirt-install/manifests/cluster-ingress-02-config.yml.bak 2020-02-06 18:23:58.572038948 +0100
@@ -4,5 +4,5 @@
   creationTimestamp: null
   name: cluster
 spec:
-  domain: apps.targz.it
+  domain: apps.ocp4.targz.it
 status: {}
```

Increase the worker node memory to fully utilize your server:

In this case I have a system with a total of 256GB RAM ... minus 3x 16GB \(=48GB\) master memory ... so we can easily assign 96GB RAM \(96\*1024\) per worker node:

```text
$ grep Memory libvirt-install/openshift/99_openshift-cluster-api_worker-machineset-0.yaml 
          domainMemory: 7168
$ sed -i.bak '{/domainMemory/s#7168#98304#;}' libvirt-install/manifests/cluster-ingress-02-config.yml
$ diff -u libvirt-install/openshift/99_openshift-cluster-api_worker-machineset-0.yaml*
--- libvirt-install/openshift/99_openshift-cluster-api_worker-machineset-0.yaml 2020-02-06 18:46:21.076946167 +0100
+++ libvirt-install/openshift/99_openshift-cluster-api_worker-machineset-0.yaml.bak     2020-02-06 18:21:29.938291920 +0100
@@ -30,7 +30,7 @@
           apiVersion: libvirtproviderconfig.openshift.io/v1beta1
           autostart: false
           cloudInit: null
-          domainMemory: 98304
+          domainMemory: 7168
           domainVcpu: 4
           ignKey: ""
           ignition:
```

Let's create the cluster:

```text
$ env TF_VAR_libvirt_master_memory=16384 TF_VAR_libvirt_master_vcpu=8 \
OPENSHIFT_INSTALL_RELEASE_IMAGE_OVERRIDE=registry.svc.ci.openshift.org/ocp/release:4.3.1 \
openshift-install --dir=libvirt-install --log-level debug \
create cluster
```

