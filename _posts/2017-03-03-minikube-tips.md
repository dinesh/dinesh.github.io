---
layout: post
title: Minikube Workflows and Tips
---

#### Overview

If you ever used docker, You must have heard people raving about [Kubernetes](http://kubernetes.io/), but getting Kubernetes up and running locally was hard. Surely it was missing the great local development platform.

[Minikube](https://github.com/kubernetes/minikube) is the easiest way to run Kubernetes locally. It runs a single-node Kubernetes cluster inside a VM on your laptop for users looking to try out Kubernetes or develop with it day-to-day. 

Since Minikube runs locally instead of on a cloud provider, certain provider-specific features like LoadBalancers and PersistentVolumes will not work out-of-the-box. However, you can use NodePort, HostPath PersisentVolumes and several add-ons like DNS, dashboard, CNI, etc. to test your apps locally before pushing to production-grade cluster. 

#### Installation

Minikube is VM provider agnostic and supports several drivers like [VMware](http://www.vmware.com/products/fusion.html), [Virtualbox](https://www.virtualbox.org/), [xhype](https://github.com/mist64/xhyve), [Kvm](https://www.linux-kvm.org) etc. You will also need [`kubectl`](https://kubernetes.io/docs/user-guide/kubectl-overview/) to interact with cluster.

Installing Minikube is pretty easy since it's available as a single binary. At the time of writing this article 0.17.1 is the latest version, to install -

    $ export MINIKUBE_VERSION=0.17.1

    $ curl -Lo minikube https://storage.googleapis.com/minikube/releases/$MINIKUBE_VERSION/minikube-darwin-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/

Please visit this page for the latest release of [Minikube](https://github.com/kubernetes/minikube/releases)

In order to validate our deployment and check that everything went well, let’s start the local Kubernetes cluster:
  
    $ minikube start
      Starting local Kubernetes cluster...

Let’s start by first checking how many nodes are up:

    | => kubectl get nodes
    NAME       STATUS    AGE
    minikube   Ready     3d

You can checkout [other awesome blogs](https://kubeweekly.com/?s=minikube) regarding how to deploy your applications on newly created cluster.

#### Architecture

Minikube is built on top of Docker's [libmachine](https://github.com/docker/machine/tree/master/libmachine), and leverages the driver model to create, manage and interact with locally-run virtual machines.

Inspired from RedSpread's localkube, Minikube spins up a single-process Kubernetes cluster inside a VM. Localkube bundles etcd, DNS, the Kubelet and all the Kubernetes master components into a single Go binary, and runs them all via separate go-routines.

Limitations

Though community is working hard to make Minikube feature rich, it has certain missing features like - 

  - LoadBalancer
  - PersistentVolume
  - Ingress

#### Helping Tips & Gotchas

##### Sharing docker daemon

  Minikube contains a built-in Docker daemon for running containers. It is recommended to reuse the Docker daemon which Minikube uses for pure Docker use-cases as well. By using the same docker daemon as Minikube, you can speed up your local experiments. To share daemon in your shell:

    $ eval $(minikube docker-env)
    $ docker ps

  Now when you do Docker builds, the images you build will be available to Minikube.

##### Xhype Driver 
  if you are on Mac, it's better to use [xhype] hypervisor for speed and less momory usage.

    $ minikube start --vm-driver=xhyve
  
  Or set it permanently
  
    $ minikube config set vm-driver=xhyve

##### Addons
  Minikube aims to advance features via several [add-ons][https://github.com/kubernetes/minikube/tree/master/deploy/addons], like ingress, registry credentials, DNS, dashboard , heapster etc. 

  To enable an addon, run -
    minikube addons enable <addon>

  You can also edit the addon's config (YAML) file and apply against cluster to fulfill your usecase.

##### Switching cluster version
  
Minikube have relativelly unknown and very useful feature to switch cluster version to smoke test your app. To start minikube with a cluster version - 

    $ minikube start --kubernetes-version=v1.5.2

##### Using private images

Minikube is that it won’t always have credentials to pull from private container registries. You should use ImagePullSecrets in PodSpec as defined on offical [Image](https://github.com/kubernetes/kubernetes/blob/1f952b05d3e56cf233c94f1b2c65f05e6cde5061/docs/images.md#specifying-imagepullkeys-on-a-pod) page.

##### Port forwarding

When you are working locally, you may want to access cluster services using cli tools to quickly debug things. For example you can expose redis using

    => kubectl port-forward redis-master :6379:6379

Now you can simply use cli clients like ( `redis-cli` or `psql`, `mongo`) with the services running inside cluster. Forwarding a port with kubectl is fairly easy, however, it only works with single Pods and not with Services. Pretty neat.

##### Hot reloads via host mounts

It would be pretty neat if your code is reloaded when you change in editor, rather than building the image, and deploying again. On OSX, All you have to do is create a host mount to mount your local directory into your container like :


{% highlight yaml %}
volumeMounts:
- name: reload
  mountPath: /app
volumes:
  - name: reload
  hostPath:
path: /path/to/app/
{% endhighlight %}


Also enable your webserver to listen to file changes ( like `--reload` flag for gunicorn ). Unfortunately, host folder sharing is not implemented on Linux yet.

#### Conclusion

I hope this post helps people who are struggling setting up their own Kubernetes cluster and getting them quickly started. Here we got a single node 
Kubernetes cluster running locally, for purposes of learning, development and testing, and will be using Minikube in next article to set up applications. Thanks for reading through this tutorial! Let me know if you found it useful in the comments ;-)

