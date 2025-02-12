# kube-image-keeper (kuik)

[![Releases](https://github.com/enix/kube-image-keeper/actions/workflows/release.yml/badge.svg?branch=release)](https://github.com/enix/kube-image-keeper/releases)
[![Go report card](https://goreportcard.com/badge/github.com/enix/kube-image-keeper)](https://goreportcard.com/report/github.com/enix/kube-image-keeper)
[![MIT license](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
[![Brought to you by Enix](https://img.shields.io/badge/Brought%20to%20you%20by-ENIX-%23377dff?labelColor=888&logo=data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAA4AAAAOCAQAAAC1QeVaAAAABGdBTUEAALGPC/xhBQAAACBjSFJNAAB6JgAAgIQAAPoAAACA6AAAdTAAAOpgAAA6mAAAF3CculE8AAAAAmJLR0QA/4ePzL8AAAAHdElNRQfkBAkQIg/iouK/AAABZ0lEQVQY0yXBPU8TYQDA8f/zcu1RSDltKliD0BKNECYZmpjgIAOLiYtubn4EJxI/AImzg3E1+AGcYDIMJA7lxQQQQRAiSSFG2l457+655x4Gfz8B45zwipWJ8rPCQ0g3+p9Pj+AlHxHjnLHAbvPW2+GmLoBN+9/+vNlfGeU2Auokd8Y+VeYk/zk6O2fP9fcO8hGpN/TUbxpiUhJiEorTgy+6hUlU5N1flK+9oIJHiKNCkb5wMyOFw3V9o+zN69o0Exg6ePh4/GKr6s0H72Tc67YsdXbZ5gENNjmigaXbMj0tzEWrZNtqigva5NxjhFP6Wfw1N1pjqpFaZQ7FAY6An6zxTzHs0BGqY/NQSnxSBD6WkDRTf3O0wG2Ztl/7jaQEnGNxZMdy2yET/B2xfGlDagQE1OgRRvL93UOHqhLnesPKqJ4NxLLn2unJgVka/HBpbiIARlHFq1n/cWlMZMne1ZfyD5M/Aa4BiyGSwP4Jl3UAAAAldEVYdGRhdGU6Y3JlYXRlADIwMjAtMDQtMDlUMTQ6MzQ6MTUrMDI6MDDBq8/nAAAAJXRFWHRkYXRlOm1vZGlmeQAyMDIwLTA0LTA5VDE0OjM0OjE1KzAyOjAwsPZ3WwAAAABJRU5ErkJggg==)](https://enix.io)

kube-image-keeper (a.k.a. *kuik*, which is pronounced /kwɪk/, like "quick") is a container image caching system for Kubernetes.
It saves the container images used by your pods in its own local registry so that these images remain available if the original becomes unavailable.

## Why and when is it useful?

At [Enix](https://enix.io/), we manage production Kubernetes clusters both for our internal use and for various customers; sometimes on premises, sometimes in various clouds, public or private. We regularly run into image availability issues, for instance:

- the registry is unavailable or slow;
- a critical image was deleted from the registry (by accident or because of a misconfigured retention policy),
- the registry has pull quotas (or other rate-limiting mechanisms) and temporarily won't let us pull more images.

(The last point is a well-known challenge when pulling lots of images from the Docker Hub, and becomes particularly painful when private Kubernetes nodes access the registry through a single NAT gateway!)

We needed a solution that would:

- work across a wide range of Kubernetes versions, container engines, and image registries,
- preserve Kubernetes' out-of-the-box image caching behavior and [image pull policies](https://kubernetes.io/docs/concepts/containers/images/#image-pull-policy),
- have fairly minimal requirements,
- and be easy and quick to install.

We investigated other options, and we didn't find any that would quite fit our requirements, so we wrote kuik instead.

## Prerequisites

- A Kubernetes cluster¹ (duh!)
- Admin permissions²
- cert-manager³
- Helm⁴ >= 3.2.0
- CNI plugin with [port-mapper⁵](https://www.cni.dev/plugins/current/meta/portmap/) enabled
- In a production environment, we definitely recommend that you use persistent⁶ storage

¹A local development cluster like minikube or KinD is fine.
<br/>
²In addition to its own pods, kuik needs to register a MutatingWebhookConfiguration.
<br/>
³kuik uses cert-manager to issue and configure its webhook certificate. You don't need to configure cert-manager in a particular way (you don't even need to create an Issuer or ClusterIssuer). It's alright to just `kubectl apply` the YAML as shown in the [cert-manager installation instructions](https://cert-manager.io/docs/installation/).
<br/>
⁴If you prefer to install with "plain" YAML manifests, we'll tell you how to generate these manifests.
<br/>
⁵Most CNI plugins these days enable port-mapper out of the box, so this shouldn't be an issue, but we're mentioning it just in case.
<br/>
⁶You can use kuik without persistence, but if the pod running the registry gets deleted, you will lose your cached images. They will be automatically pulled again when needed, though.

## Supported Kubernetes versions

kuik has been developed for, and tested with, Kubernetes 1.21 to 1.24; but the code doesn't use any deprecated (or new) feature or API, and should work with newer versions as well. (Community users have reported success with Kubernetes 1.26).

## How it works

When a pod is created, kuik's **mutating webhook** rewrites its images on the fly, adding a `localhost:{port}/` prefix (the `port` is 7439 by default, and is configurable).

On `localhost:{port}`, there is an **image proxy** that serves images from kuik's **caching registry** (when the images have been cached) or directly from the original registry (when the images haven't been cached yet).

One **controller** watches pods, and when it notices new images, it creates `CachedImage` custom resources for these images.

Another **controller** watches these `CachedImage` custom resources, and copies images from source registries to kuik's caching registry accordingly.

Here is what our images look like when using kuik:
```bash
$ kubectl get pods -o custom-columns=NAME:metadata.name,IMAGES:spec.containers[*].image
NAME                   IMAGES
debugger               localhost:7439/registrish.s3.amazonaws.com/alpine
factori-0              localhost:7439/factoriotools/factorio:1.1
nvidiactk-b5f7m        localhost:7439/nvcr.io/nvidia/k8s/container-toolkit:v1.12.0-ubuntu20.04
sshd-8b8c6cfb6-l2tc9   localhost:7439/ghcr.io/jpetazzo/shpod
web-8667899c97-2v88h   localhost:7439/nginx
web-8667899c97-89j2h   localhost:7439/nginx
web-8667899c97-fl54b   localhost:7439/nginx
```

The kuik controllers keep track of how many pods use a given image. When an image isn't used anymore, it is flagged for deletion, and removed one month later. This expiration delay can be configured. You can see kuik's view of your images by looking at the `CachedImages` custom resource:

```bash
$ kubectl get cachedimages
NAME                                                       CACHED   EXPIRES AT             PODS COUNT   AGE
docker.io-dockercoins-hasher-v0.1                          true     2023-03-07T10:50:14Z                36m
docker.io-factoriotools-factorio-1.1                       true                            1            4m1s
docker.io-jpetazzo-shpod-latest                            true     2023-03-07T10:53:57Z                9m18s
docker.io-library-nginx-latest                             true                            3            36m
ghcr.io-jpetazzo-shpod-latest                              true                            1            36m
nvcr.io-nvidia-k8s-container-toolkit-v1.12.0-ubuntu20.04   true                            1            29m
registrish.s3.amazonaws.com-alpine-latest                                                  1            35m
```

## Architecture and components

In kuik's namespace, you will find:

- a `Deployment` to run kuik's controllers,
- a `DaemonSet` to run kuik's image proxy,
- a `StatefulSet` to run kuik's image cache, a `Deployment` is used instead when this component runs in HA mode.

The image cache will obviously require a bit of disk space to run (see [Garbage collection and limitations](#garbage-collection-and-limitations) below). Otherwise, kuik's components are fairly lightweight in terms of compute resources. This shows CPU and RAM usage with the default setup, featuring two controllers in HA mode:

```bash
$ kubectl top pods
NAME                                             CPU(cores)   MEMORY(bytes)
kube-image-keeper-0                              1m           86Mi
kube-image-keeper-controllers-5b5cc9fcc6-bv6cp   1m           16Mi
kube-image-keeper-controllers-5b5cc9fcc6-tjl7t   3m           24Mi
kube-image-keeper-proxy-54lzk                    1m           19Mi
```

![Architecture](./docs/architecture.jpg)

### Metrics

Refer to the [dedicated documentation](./docs/metrics.md).


## Installation

1. Make sure that you have cert-manager installed. If not, check its [installation page](https://cert-manager.io/docs/installation/) (it's fine to use the `kubectl apply` one-liner, and no further configuration is required).
1. Install kuik's Helm chart from the [enix/helm-charts](https://github.com/enix/helm-charts) repository:

```bash
helm upgrade --install \
     --create-namespace --namespace kuik-system \
     kube-image-keeper kube-image-keeper \
     --repo https://charts.enix.io/
```

That's it!

## Installation with plain YAML files

You can use Helm to generate plain YAML files and then deploy these YAML files with `kubectl apply` or whatever you want:

```bash
helm template --namespace kuik-system \
     kube-image-keeper kube-image-keeper \
     --repo https://charts.enix.io/ \
     > /tmp/kuik.yaml
kubectl create namespace kuik-system
kubectl apply -f /tmp/kuik.yaml --namespace kuik-system
```

## Configuration and customization

If you want to change e.g. the expiration delay, the port number used by the proxy, enable persistence (with a PVC) for the registry cache... You can do that with standard Helm values.

You can see the full list of parameters (along with their meaning and default values) in the chart's [values.yaml](/helm/kube-image-keeper/values.yaml) file, or on [kuik's page on the Artifact Hub](https://artifacthub.io/packages/helm/enix/kube-image-keeper).

For instance, to extend the expiration delay to 3 months (90 days), you can deploy kuik like this:

```bash
helm upgrade --install \
     --create-namespace --namespace kuik-system \
     kube-image-keeper kube-image-keeper \
     --repo https://charts.enix.io/ \
     --set cachedImagesExpiryDelay=90
```

## Advanced usage

### Pod filtering

There are 3 ways to tell kuik which pods it should manage (or, conversely, which ones it should ignore).

- If a pod has the label `kube-image-keeper.enix.io/image-caching-policy=ignore`, kuik will ignore the pod (it will not rewrite its image references).
- If a pod is in an ignored Namespace, it will also be ignored. Namespaces can be ignored by setting the Helm value `controllers.webhook.ignoredNamespaces`, which defaults to `[kube-system]`. (Note: this feature relies on the [NamespaceDefaultLabelName](https://kubernetes.io/docs/concepts/services-networking/network-policies/#targeting-a-namespace-by-its-name) feature gate to work.)
- Finally, kuik will only work on pods matching a specific selector. By default, the selector is empty, which means "match all the pods". The selector can be set with the Helm value `controllers.webhook.objectSelector.matchExpressions`.

This logic isn't implemented by the kuik controllers or webhook directly, but through Kubernetes' standard webhook object selectors. In other words, these parameters end up in the `MutatingWebhookConfiguration` template to filter which pods get presented to kuik's webhook. When the webhook rewrites the images for a pod, it adds a label to that pod, and the kuik controllers then rely on that label to know which `CachedImages` resources to create.

Keep in mind that kuik will ignore pods scheduled into its own namespace.

### Cache persistence & garbage collection

Persistence is disabled by default. You can enable it by setting the Helm value `registry.persistence.enabled=true` and setting `registry.persistence.size` to the desired size (20 GiB by default). Be aware that this don't allow for high availability of the registry, to enable HA please refer to our related guide [./docs/high-availability.md](./docs/high-availability.md).

Note that persistence requires that you have Persistent Volumes available on your cluster; otherwise, kuik's registry pod will remain `Pending` and your images won't be cached (but they will still be served transparently by kuik's image proxy).

## Garbage collection and limitations

When a CachedImage expires because it is not used anymore by the cluster, the image is deleted from the registry. However, since kuik uses [Docker's registry](https://docs.docker.com/registry/), this only deletes **reference files** like tags. It doesn't delete blobs, which account for most of the used disk space. [Garbage collection](https://docs.docker.com/registry/garbage-collection/) allows removing those blobs and free up space. The garbage collecting job can be configured to run thanks to the `registry.garbageCollectionSchedule` configuration in a cron-like format. It is disabled by default, because running garbage collection without persistence would just wipe out the cache registry.

Garbage collection can only run when the registry is read-only (or stopped), otherwise image corruption may happen. (This is described in the [registry documentation](https://docs.docker.com/registry/garbage-collection/).) Before running garbage collection, kuik stops the registry. During that time, all image pulls are automatically proxified to the source registry so that garbage collection is mostly transparent for cluster nodes.

Reminder: since garbage collection recreates the cache registry pod, if you run garbage collection without persistence, this will wipe out the cache registry. It is not recommended for production setups!

Currently, if the cache gets deleted, the `status.isCached` field of `CachedImages` isn't updated automatically, which means that `kubectl get cachedimages` will incorrectly report that images are cached. However, you can trigger a controller reconciliation with the following command, which will pull all images again:

```bash
kubectl annotate cachedimages --all --overwrite "timestamp=$(date +%s)"
```
