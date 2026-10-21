+++
date = '2026-09-30T13:38:08+02:00'
draft = false
title = 'Hacking K0S Part 2 : Start patching and hacking'
+++

## Create the test image

In the previous article, we've built an image using the image builder.
So let's create a new branch and a new version of the image builder.

```sh
$ cd image-builder
$ git checkout -b test
$ cd images/apiserver-network-proxy-agent/
$ v0.36.0-k0s.1 test
```

## Clone and branch the softwares

Let's clone the [Kubernetes-sigs repo](https://github.com/kubernetes-sigs/apiserver-network-proxy.git) and create a tgz archive for the next step

```sh
$ git clone https://github.com/kubernetes-sigs/apiserver-network-proxy.git k8ssigs-apiserver-network-proxy
$ cd k8ssigs-apiserver-network-proxy
$ tar cfz ../image-builder/images/apiserver-network-proxy-agent/test/k8ssigs-apiserver-network-proxy.tgz .
```

## Replace release with local test branch

First, replace the wget from Github to use locally generated archive instead.

```dockerfile
WORKDIR /go/src/kubernetes-sigs/apiserver-network-proxy

ARG VERSION HASH
RUN --mount=type=cache,id=konnectivity-go-modcache,target=/go/pkg/mod \
  --mount=type=tmpfs,target=/tmp \
  set -e \
  && wget -qO /tmp/src.tar.gz https://github.com/kubernetes-sigs/apiserver-network-proxy/archive/refs/tags/v$VERSION.tar.gz \
  && { echo $HASH /tmp/src.tar.gz | sha256sum -c -; } || { sha256sum /tmp/src.tar.gz; exit 1; } \
  && tar xf /tmp/src.tar.gz --strip-components=1
```

Add the COPY step and use local file instead of wget

```dockerfile
COPY k8ssigs-apiserver-network-proxy.tgz /root

WORKDIR /go/src/kubernetes-sigs/apiserver-network-proxy

RUN --mount=type=cache,id=konnectivity-go-modcache,target=/go/pkg/mod \
  --mount=type=tmpfs,target=/tmp \
  set -e \
  && tar xfz /root/k8ssigs-apiserver-network-proxy.tgz && \
  rm -vf /root/k8ssigs-apiserver-network-proxy.tgz

```

<!--more-->

Then, updates the Go modules version as needed

```dockerfile
ARG XNET_VERSION=v0.59.0 XTEXT_VERSION=v0.42.0 GRPC_VERSION=v1.84.0
```

## Build and deploy the test image

As before, you build and push the image with **Docker Buildx**

```sh
docker buildx build \
  --builder multiarch \
  --progress tty \
  --platform linux/amd64,linux/arm64 \
  --tag myserver.home.local:5000/test/k0s-apiserver-network-proxy-agent:test \
  --push \
  .
```

And then update the daemon set to use the new image

```sh
$ kubectl -n kube-system set image ds/konnectivity-agent konnectivity-agent=myserver.home.local:5000/test/k0s-apiserver-network-proxy-agent:test
$ kubectl -n kube-system rollout restart ds/konnectivity-agent
$ kubectl -n kube-system rollout status ds/konnectivity-agent
```

## Wrap up all in a single script

```sh
#! /bin/bash
########################################################################################################################

set -x -e -o pipefail

########################################################################################################################

__base="/home/user/DEV-CONTRIB/K0S"

DIR_K8SSIG="${__base}/k8ssigs-apiserver-network-proxy/"
DIR_K0S_IMG="${__base}/image-builder/images/apiserver-network-proxy-agent/test/"

IMG_REPO="myserver.home.local:5000/test"
IMG_NAME="k0s-apiserver-network-proxy-agent"
IMG_TAG="test-$(date +%Y%m%d-%H%M%S)"
IMG_FULLNAME="${IMG_REPO}/${IMG_NAME}:${IMG_TAG}"

KUBE_DS_NAME="konnectivity-agent"

########################################################################################################################
cd "${DIR_K8SSIG}"
pwd
ls -la
tar cfz ${DIR_K0S_IMG}/k8ssig-apiserver-network-proxy.tgz --ignore-failed-read .

########################################################################################################################
cd "${DIR_K0S_IMG}"
docker buildx build \
  --builder multiarch \
  --progress tty \
  --platform linux/amd64,linux/arm64 \
  --tag ${IMG_FULLNAME} \
  --push \
  .

########################################################################################################################

kubectl -n kube-system set image ds/${KUBE_DS_NAME} konnectivity-agent=${IMG_FULLNAME}
kubectl -n kube-system rollout restart ds/${KUBE_DS_NAME}
kubectl -n kube-system rollout status ds/${KUBE_DS_NAME}
```

Now you are ready to create and test your fixes and features.
