+++
date = '2026-09-29T18:34:43+02:00'
draft = false
title = 'Hacking K0S Part 1 : Setup build environment'
+++

Lately, I've encounter some issues with my K0S Kubernetes distro running on my Raspberry Pi cluster. The Konnectivity agent is generating a lot of logs and error. This problem has already been reported (like [here](https://github.com/k0sproject/k0s/issues/6444)) but it has been closed without any real response. So let start digging this.

In this article, we will discuss about the build environment and dependencies. Namely, a container registry to store the artifacts to be tested, a multi architecture builder as the test environment runs on Raspberry Pi and the build environment is AMD64. Finally, validate all this by building and deploying an image using the code without any change.

## Deploy a local registry server with Docker

Running a registry is simple as running a single container, using the [registry image](https://hub.docker.com/_/registry)

Run an additional container for the user interface, using [registry-ui](https://hub.docker.com/r/joxit/docker-registry-ui) for easier management.

Adds some environment so the web interface can query the registry without being rejected and you get the following services for Docker Compose.

```yaml
services:
  registry:
    image: docker.io/library/registry:3.1.2
    ports:
      - 5000:5000
    hostname: registry
    restart: always
    userns_mode: host
    environment:
      REGISTRY_HTTP_HEADERS_Access-Control-Allow-Origin: '[http://myserver.home.local:5080]'
      REGISTRY_HTTP_HEADERS_Access-Control-Allow-Methods: '[HEAD,GET,OPTIONS,DELETE]'
      REGISTRY_HTTP_HEADERS_Access-Control-Allow-Credentials: '[true]'
      REGISTRY_HTTP_HEADERS_Access-Control-Allow-Headers: '[Authorization,Accept,Cache-Control]'
      REGISTRY_HTTP_HEADERS_Access-Control-Expose-Headers: '[Docker-Content-Digest]'
      REGISTRY_STORAGE_DELETE_ENABLED: 'true'
    volumes:
      - /srv/registry:/var/lib/registry

  registry-ui:
    image: joxit/docker-registry-ui:2.6.0
    container_name: docker-registry-ui
    ports:
      - 5080:80
    environment:
      - REGISTRY_TITLE=myserver.home.local
      - REGISTRY_URL=http://myserver.home.local:5000
      - SINGLE_REGISTRY=true
      - SHOW_CATALOG_NB_TAGS=true
    depends_on:
      - registry
    restart: always
```

## Create the Docker Buildx multi arch builder

To use the local unsecure registry you must create a custom **Docker Buildx** builder. Here are the steps.

<!--more-->

Create a buildkit config: **buildkit.toml**

```toml
[registry."myserver.home.local:5000"]
  http = true
```

And create the Docker Buildx builder. With, in this example, a remote Docker host.

```sh
$ export DOCKER_HOST=ssh://myserver.home.local
$ docker buildx create \
  --name debian-builder \
  --driver docker-container \
  --buildkitd-config buildkit.toml \
  --bootstrap \
  --use
```

Check the builder is running

```sh
$ export DOCKER_HOST=ssh://myserver.home.local
$ docker buildx ls
NAME/NODE        DRIVER/ENDPOINT                STATUS    BUILDKIT   PLATFORMS
multiarch*       docker-container                                    
 \_ multiarch0    \_ ssh://myserver.home.local   running   v0.32.2    linux/amd64 (+4), linux/386
default          docker                                              
 \_ default       \_ default                    running   v0.33.0    linux/amd64 (+4), linux/386
```

## Clone the K0S image repository

Now let's clone the image definitions Github repo.

All the K0S images are defined in the [image-builder Git repo](https://github.com/k0sproject/image-builder.git)

## Build the first image

The repository contains all the K0S images, let's pick the one.

First step: go to the proper directory 

```sh
cd images/apiserver-network-proxy-agent/v0.36.0-k0s.1
```

And build the image

```sh
docker buildx build \
  --builder multiarch \
  --platform linux/amd64,linux/arm64 \
  --tag myserver.home.local:5000/test/k0s-apiserver-network-proxy-agent:v0.36.0-k0s.1 \
  --push \
  .
```

## Add the local registry on K0S with k0sctl

To be able to use the local unsecure registry, you must add it on the K0S workers nodes.
This can be done with **k-sctl**.

Adds the registries directory using **insecure-registry.toml**:

```toml
version = 3

[plugins."io.containerd.cri.v1.images".registry]
  config_path = "/etc/k0s/containerd.d/certs.d"
```

And the registry definition **myserver.toml**

```toml
server = "http://myserver.home.local:5000"

[host."http://myserver.home.local:5000"]
  capabilities = ["pull", "resolve", "push"]
  skip_verify = true
```

Finaly deploy it on all the workers nodes

```yaml
apiVersion: k0sctl.k0sproject.io/v1beta1
kind: Cluster
metadata:
  name: fs-pi4-k0s
  user: admin
spec:
  hosts:
  - ssh:
      address: 192.168.100.160
      keyPath: ~/.ssh/id_ed25519.pub
    role: worker
    files:
      - src: ./insecure-registry.toml
        dst: /etc/k0s/containerd.d/insecure-registry.toml
        perm: 0644
      - src: ./myserver.toml
        dst: /etc/k0s/containerd.d/certs.d/myserver.home.local:5000/hosts.toml
        perm: 0644
```

## Deploy it on K0S

Final step: change the image used in the deployment set and start a new rollout.

```sh
$ kubectl -n kube-system set image ds/konnectivity-agent konnectivity-agent=myserver.home.local:5000/test/k0s-apiserver-network-proxy-agent:v0.36.0-k0s.1
$ kubectl -n kube-system rollout restart ds/konnectivity-agent
$ kubectl -n kube-system rollout status ds/konnectivity-agent
```

After a few minutes, all the nodes should run the new image.

The build environement is now ready and tested. This finish the first part.

The next step, change the image definition to use local software instead of the Github archive. Click [here](/post/2026-09-30-hacking-k0s-part2/)
