# Installing GTP5G Driver with the Driver Toolkit on OCP 4.15
To integrate [free5gc](https://free5gc.org/) with Openshift, the [gtp5g kernel driver](https://github.com/free5gc/gtp5g) must be loaded on the UPF worker node. Openshift offers the drivers toolkit, which includes all dependencies necessary for building the kernel module and installing it atop base images. The following steps outline the procedure using the Driver Toolkit for OCP 4.15, which should also be compatible with version 4.14:


**Note**: Please refer to the [product documentation](https://docs.openshift.com/container-platform/4.15/hardware_enablement/psap-driver-toolkit.html) for detailed usage instructions.


## Finding the Driver Toolkit image URL in the payload
````
$ oc adm release info quay.io/openshift-release-dev/ocp-release:4.15.0-x86_64 --image-for=driver-toolkit
quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:cd0ea5d8ec43c5b03bf362e0b595bafe3e97e222d4344a851453ebe8770df135

````

## Get the correct CoreOS image
````
$ oc adm release info quay.io/openshift-release-dev/ocp-release:4.15.0-x86_64 --image-for=rhel-coreos
quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:a0bbd3520a99b3cc5357bbe2ad0ba754768d305f47ea53491941b7e6d427d2e8

````

## Get the kerenl version
````
$ podman run -it quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:a0bbd3520a99b3cc5357bbe2ad0ba754768d305f47ea53491941b7e6d427d2e8 \
rpm -qa | grep kernel

kernel-modules-core-5.14.0-284.54.1.el9_2.x86_64
kernel-core-5.14.0-284.54.1.el9_2.x86_64
kernel-modules-5.14.0-284.54.1.el9_2.x86_64
kernel-5.14.0-284.54.1.el9_2.x86_64
kernel-modules-extra-5.14.0-284.54.1.el9_2.x86_64


`````

AEnsure to replace any placeholders with actual values and configurations specific to your setup.
Then build, gtp5g driver BuildConifg and Image Steram.

````
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  labels:
    app: gtp5g-driver-container
  name: gtp5g-driver-container
  namespace: gtp5g-demo
spec: {}
---
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  labels:
    app: gtp5g-driver-build
  name: gtp5g-driver-build
  namespace: gtp5g-demo
spec:
  nodeSelector:
    node-role.kubernetes.io/worker: ""
  runPolicy: "Serial"
  triggers:
    - type: "ConfigChange"
    - type: "ImageChange"
  source:
    dockerfile: |
      ARG DTK
      ARG RHCOS_IMAGE

      FROM ${DTK} as builder
      ARG KERNEL_VERSION

      WORKDIR /usr/src
      RUN git clone https://github.com/free5gc/gtp5g.git

      WORKDIR /usr/src/gtp5g
      RUN make KVER=${KERNEL_VERSION} clean
      RUN make KVER=${KERNEL_VERSION}

      FROM ${RHCOS_IMAGE}
      # FROM registry.redhat.io/ubi9/ubi-minimal  # this image doesn have kernel sources
      ARG KERNEL_VERSION

      # COPY --from=builder /usr/src/gtp5g/gtp5g.ko /lib/modules/${KVER}/kernel/drivers/net/
      COPY --from=builder /etc/driver-toolkit-release.json /etc/
      COPY --from=builder /usr/src/gtp5g/gtp5g.ko /root/
      RUN ln -s /root/gtp5g.ko /lib/modules/${KERNEL_VERSION}/kernel/drivers/net

      RUN depmod -a "${KERNEL_VERSION}" && echo gtp5g > /etc/modules-load.d/gtp5g.conf
  strategy:
    dockerStrategy:
      buildArgs:
        - name: DTK
          value: quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:b9cd86347ba410c90b4a34fe9c1b25951e0f0cd38ceca1d3ccd4bae96f084edb
        # - name: KVER
        #   value: 5.14.0-284.59.1.el9_2.x86_64
        - name: KERNEL_VERSION
          value: 5.14.0-284.59.1.el9_2.x86_64
        - name: RHCOS_IMAGE
          value: quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:557f27afcca35e6e31e14c7816adb5b7afa2dbe0ec382dc1d0783b17eb17ce95
  output:
    to:
      kind: ImageStreamTag
      name: gtp5g-driver-container:demo


# # ocp 4.15.6
# RHCOS_IMAGE=quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:557f27afcca35e6e31e14c7816adb5b7afa2dbe0ec382dc1d0783b17eb17ce95
# # DTK
# DTK=quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:b9cd86347ba410c90b4a34fe9c1b25951e0f0cd38ceca1d3ccd4bae96f084edb
# # Kernel
# KERNEL_VERSION=5.14.0-284.59.1.el9_2.x86_64

# 4.15.0
# toolkit
# export DTK_IMAGE=quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:cd0ea5d8ec43c5b03bf362e0b595bafe3e97e222d4344a851453ebe8770df135
# coreos
# export RHCOS_IMAGE=quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:a0bbd3520a99b3cc5357bbe2ad0ba754768d305f47ea53491941b7e6d427d2e8
# kernel
# export KERNEL_VERSION=5.14.0-284.54.1.el9_2.x86_64


````

Apply the RBAC and daemonset yaml

````
oc create -f 1000-drivercontainer.yaml
````

Verify the daemon set and driver.

````
$ oc get pod -A | grep gtp5g-driver-container
gtp5g-demo                                         gtp5g-driver-container-ztl9r                                      1/1     Running     1

````

````
$ oc exec -it gtp5g-driver-container-ztl9r -n gtp5g-demo -- lsmod | grep gtp
gtp5g                 184320  0
udp_tunnel             24576  2 gtp5g,sctp

````
The driver should be shown in the node as well. Good to go to install fee5gc!

