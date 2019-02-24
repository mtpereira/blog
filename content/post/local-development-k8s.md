---
title: "Local development with Kubernetes"
aliases:
 - "local-development-k8s.html"
 - "local-development-k8s"
date: 2016-10-15T08:10:12Z
tags: ["kubernetes", "development", "minikube", "macos", "docker", "registry"]
draft: false
---

Lately I've been working with [Kubernetes](http://kubernetes.io/) in order to
provide a self-service deployment platform. This ranges from setting up a
production-ready cluster, to figuring out how to fit it in a development
workflow.

Although it's getting easier, cheaper and faster to use cloud-provisioned
environments for development, it can be a productivity-killer to have the
cognitive overhead of dealing with remote hosts and what-not for running your
code.

Also, when developing Docker containers, we often do not wish to
publish them, which can be a pain if you don't have a
private registry already setup. The following is the setup that I've found to
deal with these issues, using
[minikube](https://github.com/kubernetes/minikube) running on macOS.

# Setup minikube

We'll be using [xhyve](https://github.com/mist64/xhyve) to run minikube.
This alternative is lighter and smaller than the default VirtualBox VM and also
has the advantage of using Plan9 filesystem over Virtio for mounting your local
filesystem on the VM. So far, this setup has been much more reliable for me when
it comes to dealing with ownership and permissions of shared files than using
vboxfs.

Install all the required packages to run minikube on a xhyve VM:

{{< gist mtpereira 7362206e271de79c545956afcc77af3a "minikube-setup.sh" >}}

To start a cluster on minikube using xhyve, run the following:

{{< highlight bash >}}
minikube start --vm-driver xhyve --insecure-registry localhost:5000
{{</ highlight >}}

This will create a VM and configure a single-node Kubernetes cluster inside it.
We'll see in a bit why we need the `--insecure-registry` flag.

I'd recommend to add this command to your shell's aliases so that you do not
forget to start your cluster with these options:

{{< highlight bash >}}
alias minikube-start='minikube start --vm-driver xhyve --insecure-registry localhost:5000'
{{</ highlight >}}

# Using minikube's Docker daemon from our localhost

Since we're already running a Docker daemon inside minikube's VM, we should take
advantage of this instead of relying on another VM. This really saves up some
resources on your host and keeps things simple.

In order to do that, you just need to run the following and you're ready to go:

{{< highlight bash >}}
eval $(minikube docker-env)
{{</ highlight >}}

# Running a local private Docker registry

Kubernetes provides an add-on for running a private Docker registry inside your
cluster, which is documented
[here](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/registry).
However, there are a couple of caveats that you need to consider when running it
inside `minikube`.

Currently, there's [limited
support](https://github.com/kubernetes/minikube/issues/461#issuecomment-238326420)
for adding add-ons automatically to the Kubernetes cluster running on minikube. As
such, our registry manifest will be a little different from the official one:

{{< gist mtpereira 7362206e271de79c545956afcc77af3a "local-registry.yml" >}}

I've also decided to keep this local registry ephemeral, which is okay for me at
this point, given that I'm dealing with really small images. This can be
easily changed to use a persistent volume as described on the 
[registry add-on documentation](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/registry).

The `--insecure-registry localhost:5000` that we've used on the `minikube start`
command is relevant here, because it allows for the Docker daemon running on the
minikube node to connect to an insecure Docker registry, without using SSL. This
simplifies the setup process without any real disadvantage, since we'll be using
this same Docker daemon to build and push our images, so nothing will be sent
over external networks.

Execute this command to deploy the registry on your cluster:

{{< highlight bash >}}
kubectl apply -f local-registry.yml
{{</ highlight >}}


# TL;DR

{{< gist mtpereira 7362206e271de79c545956afcc77af3a "tldr.sh" >}}

# Conclusion

At this point, you'll now be able to build and push containers to a local
private registry, as well as deploy them to a local Kubernetes cluster. I
believe this a great setup to develop applications which will run on a similar
environment in production, simplifying the development flow and setup and, at
the same time, reducing the possibility of "works-on-my-machine" issues.

If you'd like to discuss this post, please drop me a line using [any of my contacts](about).

# Resources

1. ["Configuring the Ultimate Development Environment for Kubernetes"](http://thenewstack.io/tutorial-configuring-ultimate-development-environment-kubernetes/)
2. ["Private Docker Registry in Kubernetes"](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/registry)
