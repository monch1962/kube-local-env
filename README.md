# kube-local-env
Info about getting a local Kubernetes development environment installed on a laptop

## Overview

## Software components

To get a Kubernetes environment setup for local development, I'm using the following tools to create and control the Kubernetes stack
- `podman` (https://podman.io)
- `kind` (https://kind.sigs.k8s.io/)
- `kubectl` (https://kubernetes.io/docs/reference/kubectl/)
- `gatekeeper` (https://github.com/open-policy-agent/gatekeeper)
- `argocd` (https://argo-cd.readthedocs.io/)

For development against that environment, I'm using the following tools:
- VS Code
- a development container with a broad set of tools installed that I don't need to install natively on my laptop

### Why this combination of tools?

Several reasons:
- Docker Desktop now has licencing restrictions that can limit its use in certain business environments. Podman has no such licence restrictions
- Podman runs rootless by default, preventing creating pods that contain privileged containers. This is far more palatable to corporate security than Docker's daemon, which requires root access
- kubectl gives a command-line interface for applying changes to Kubernetes
- gatekeeper gives the ability to impose controls on what happens inside a Kubernetes cluster
- argocd gives a Kubernetes-specific solution for CI/CD, that runs entirely within a cluster. It coexists well with any non-Kubernetes CI/CD solutions
- using a development container means I don't need to install several tools natively on my laptop, and deal with corporate security concerns. A development container image can be locked down pretty tightly, and installed from a company's Artifactory registry if that's appropriate

Any or all of these tools can be substituted for other tools, but this particular stack is a good fit for most of my use cases

## Installation

Surprisingly, only 3 of these tools need to be installed directly on a laptop's OS. They are:
- `podman`
- `kind`
- VS Code

While `kubectl` would typically be installed natively as well (via e.g. `brew install kubectl` on a Mac), if necessary it can be run within podman via a Docker container such as `https://hub.docker.com/r/bitnami/kubectl`

Install podman and kind by running `brew install podman kind` on a Mac

Set up a Linux VM using podman by running `podman machine init`

Start that Linux VM by running `podman machine start`

Add the line `export KIND_EXPERIMENTAL_PROVIDER=podman` to the end of your `~/.zshrc` and/or `~/.bashrc` file on your Mac or Linux machine, then run `. ~/.zshrc` or `. ~/.bashrc` to expose that new environment variable
- this step is required as, by default, `kind` currently assumes it's running on top of Docker. This setting tells it to run on top of `podman` instead

Set up a Kubernetes cluster on the Linux VM by running `kind create cluster`

## FAQs

### How do I recover my Kube environment after a reboot?

1. restart podman with `podman machine start`
2. restart your Kubernetes cluster with `podman restart kind-control-plane`

Your Kubernetes cluster should now be recovered, including all namespaces and tools+configs inside those namespaces

### How do I update the versions of tools?

podman and kind can be updated by running `brew upgrade && brew cleanup` on a Mac

VS Code will alert you when a new version is available, and can then be upgraded inside VS Code itself.

### How can I set up a private Docker registry inside my Kubernetes local environment?

Depending on your workflow and use case, it can be handy to be able to push and pull Docker images to a private registry that you control. This allows you to build and test CI workflows, without potentially corrupting a shared Docker registry that others rely on.

Follow the instructions at https://www.linuxtechi.com/setup-private-docker-registry-kubernetes/ to setup your own private Docker registry running inside your local Kubernetes cluster
