# kube-local-env
Info about getting a local Kubernetes development & test environment installed on a laptop

## Overview

More and more enterprise development work is moving to containers. Being able to build, test, and deploy containers in your own production-like, local Kubernetes environment can be very handy.

The approach documented here allows you to set up a private, local Kubernetes environment on your own laptop. This helps address the following challenges:
- there's often a steep Kubernetes learning curve for developers/testers who are new to the platform. Having your own local Kube environment allows you to experiment and learn by trial & error, which typically isn't appropriate when working in shared environments
- production Kubernetes environments are typically hosted in a remote data centre, and making changes to those environments can take some time to execute. Having your own local Kube environment minimises these delays, and shortens the dev/test/fix/retry cycle
- contention for access to shared dev/test/staging environments can be a real problem when many people are working on shared code bases
- being able to tear down & rebuild your own private dev/test environment whenever you like can be very empowering for feature teams. 'Environment drift' is a real problem for shared dev/test environments, and this approach gives individual feature team members a workaround
- the workflow for _deploying_ applications to Kubernetes can be quite complex to navigate from a dev/test perspective. That's because production Kubernetes platforms are typically tightly controlled. This solution allows you to deploy the same production controls to your local Kube dev/test environment, and ensure that your application deploys correctly before getting to a shared environment
- if you follow generally-accepted best practice by deploying & testing your application in its own Kube namespace, you can immediately blow away a corrupted namespace using the command `kubectl delete namespace NAME_OF_YOUR_NAMESPACE`. This can be a simple way to ensure your application is being deployed to a pristine environment

Finally, this solution _empowers_ developers & testers, and gives them more autonomy over their day-to-day work habits.

## Software components

To get a Kubernetes environment setup for local development, I'm using the following tools to create and control the Kubernetes stack
- `podman` (https://podman.io)
- `kind` (https://kind.sigs.k8s.io/)
- `kubectl` (https://kubernetes.io/docs/reference/kubectl/)
- `gatekeeper` (https://github.com/open-policy-agent/gatekeeper)
- `argocd` (https://argo-cd.readthedocs.io/)

For development against that environment, I'm using the following tools:
- VS Code
- a development container with a broad set of tools installed that I don't need to install natively on my laptop. The config of the dev container is being captured at https://github.com/monch1962/vscode-dev-env-spike, and again it can be adapted to different work situations

### Why this combination of tools?

Several reasons:
- `Docker Desktop` now has licencing restrictions that can limit its use in certain business environments. Podman has no such licence restrictions
- `podman` runs rootless by default, preventing creating pods that contain privileged containers. This is far more palatable to corporate security than Docker's daemon, which requires root access
- `kubectl` gives a command-line interface for applying changes to Kubernetes
- `gatekeeper` gives the ability to impose controls on what happens inside a Kubernetes cluster
- `argocd` gives a Kubernetes-specific solution for CI/CD, that runs entirely within a cluster. It coexists well with any non-Kubernetes CI/CD solutions
- using a development container means I don't need to install several tools natively on my laptop, and deal with corporate security concerns. A development container image can be locked down pretty tightly, and installed from a company's Artifactory registry if that's appropriate

Any or all of these tools can be substituted for other tools, but this particular stack is a good fit for most of my use cases. In particular, if you're using Docker Desktop, you should be able to replace `podman` in the following instructions with `docker` (or `alias podman=docker` before running any of the following steps) and things should work fine

## Installation

Surprisingly, only 3 of these tools need to be installed directly on a laptop's OS. They are:
- `podman`
- `kind`
- VS Code

While `kubectl` would typically be installed natively as well (via e.g. `brew install kubectl` on a Mac), if necessary it can be run within podman via a Docker container such as `https://hub.docker.com/r/bitnami/kubectl`

### Set up the cluster

Install podman and kind by running `brew install podman kind` on a Mac

Set up a Linux VM using podman by running `podman machine init -v $HOME:$HOME`
- as Podman doesn't give you access to your complete file system by default, the above command will give Podman access to anything under your $HOME directory. Feel free to change it as you see fit, but this is probably a useful starting point for most use cases

Start that Linux VM by running `podman machine start`

Add the line `export KIND_EXPERIMENTAL_PROVIDER=podman` to the end of your `~/.zshrc` and/or `~/.bashrc` file on your Mac or Linux machine, then run `. ~/.zshrc` or `. ~/.bashrc` to expose that new environment variable
- this step is required as, by default, `kind` currently assumes it's running on top of Docker. This setting tells it to run on top of `podman` instead

Set up a Kubernetes cluster on the Linux VM by running `kind create cluster`, then run `kubectl cluster-info --context kind-kind` to connect kubectl to your new cluster.

---

### (optional) Run an Ollama LLM server & web UI

Running your own Ollama server & web UI is becoming a fairly common use case.

First set up a pod to contain all the comtainers. This allows us to manage both the Ollama server & the web UI together, as it probably doesn't make sense to manage them separately:
- `podman pod create --label ollama-web --name ollama-web -p 11434:11434 -p 3000:8080`

Next set up the Ollama container inside the pod, and start serving a LLM:
- `podman run --pod ollama-web -d --name ollama -v ollama_volume:/root/.ollama ollama/ollama:latest`
- `podman exec ollama ollama run phi3`
- `podman exec ollama ollama run nomic-embed-text`

Now you can check that Ollama is running and accessible:
- `curl http://localhost:11434/`
and you should see `Ollama is running` as a response

Next set up Open WebUI for Ollama. This gives you a convenient UI to converse with Ollama's LLM:
- `podman run --pod ollama-web -d -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:main`

---

### (optional) Set up a local container registry

A local container registry can be very handy for development, where you may want to be able to quickly pull the same container images over & over without storing them where someone else can access them

Setting up a local container registry is pretty simple
- `export REGISTRY_PORT=5000`
- `podman container run -dt -p ${REGISTRY_PORT}:5000 --name registry --volume registry:/var/lib/registry:Z docker.io/library/registry:2`
  - if this fails with an error such as `Error: something went wrong with the request: "listen tcp :5000: bind: address already in use"`, then you may instead replacing the previous command with e.g. `export REGISTRY_PORT=5555` then re-run this step 

That's it!

To test your local registry is working, try:
- `podman image pull docker.io/library/alpine`
- `podman image tag docker.io/library/alpine:latest localhost:${REGISTRY_PORT}/alpine:latest`
- `podman image push localhost:${REGISTRY_PORT}/alpine:latest --tls-verify=false`
- `podman image search localhost:${REGISTRY_PORT}/ --tls-verify=false`

If you want to clean up your registry later, you can:
- `podman container rm -f registry`
- `podman volume rm registry`
- `podman image rm docker.io/library/registry:2`

Personally I'd probably want to leave my local container images intact so I can use them again later, in which case you would simply run `podman container rm -f registry`

---

### (optional) Set up Argo CD

You can set up Argo CD by following the instructions at https://argo-cd.readthedocs.io/en/stable/getting_started/ A generic installation requires only 3 steps:
- `kubectl create namespace argocd`
- `kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml`
- `kubectl port-forward svc/argocd-server -n argocd 8080:443`

Note that the last step will lock up a terminal session, so maybe run `nohup kubectl port-forward svc/argocd-server -n argocd 8080:443 &` instead if you don't want to keep a zombie session around.

You can now access Argo CD at `https://localhost:8080`

Your Argo CD instance can be accessed via the username `admin` and a manufactured password. To get the password, run `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo` and your password will be displayed on the screen

Alternately, you can also access your Argo CD instance using its command-line client. The client can be installed directly on your laptop by running `brew install argocd`, or you can include it in the dev container image.

---

### (optional) Set up Gatekeeper

Gatekeeper is widely used in production environments to constrain what changes can be applied to a Kubernetes cluster. It can therefore be handy to have your own Gatekeeper instance in your local Kube dev environment, so you can confirm that deploying your application doesn't depend on changes that aren't permitted for a production deloyment. As always, it's better to find this out while working in your own environment rather than when you first try to deploy to production or another shared environment.

You can follow the instructions at https://open-policy-agent.github.io/gatekeeper/website/docs/install/, or just run 
`kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper.yaml`

**NOTE** by default, installing Gatekeeper in its out-of-box configuration will block you from simple tasks such as creating new namespaces. You'll need to create a set of OPA policies and apply them to Gatekeeper to allow you to do nearly anything with your cluster. If Gatekeeper is getting in the way or you don't need it for your use case, you can remove it by running 
`kubectl delete -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper.yaml`

---

### (optional) Set up cert-manager

You can follow the instructions at https://cert-manager.io/docs/, which are potentially as simple as running 
`kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.9.1/cert-manager.yaml`
on your laptop.

Note that most clients' production certificate management will be based on external certificate authorities such as Hashicorp Vault or LetsEncrypt. cert-manager can be configured to issue certificates from Hashicorp or other 3rd parties, but for a local dev environment it probably makes more sense to use self-signed certificates instead to reduce the number of external dependencies. 

It should be possible to set up certificate workflows that are agnostic with respect to certificate authority, so the same workflow can be implemented in both a local dev environment as well as a production pipeline and configured to use the appropriate CA for each environment. To do this, the certificate authority could be configured via e.g. environment variable, ConfigMap or kustomize, with a different value for each environment. Implementing that level of detail is best worked out for specific use cases.

---

### (optional) Set up Istio

Instructions are at https://istio.io/latest/docs/setup/getting-started/, which are potentially as simple as running

- `brew install istioctl` (Mac only)

- `istioctl install --set profile=NAME -y` where NAME is one of `default`, `demo`, `minimal`, `external`, `empty`, `preview`

- `kubectl label namespace default istio-injection=enabled` to automatically inject Envoy sidecars to every application being deployed. Note that this will only apply to the default namespace; if you're deploying to a custom namespace, you should change `default` to the name of your workspace

---

### (optional) Set up secureCodeBox

Instructions are at https://www.securecodebox.io/docs/getting-started/installation/, which are potentially as simple as running

- `helm repo add secureCodeBox https://charts.securecodebox.io`

- `kubectl create namespace securecodebox-system`

- `helm --namespace securecodebox-system upgrade --install securecodebox-operator secureCodeBox/operator`

From here, you can follow instructions at https://www.securecodebox.io/docs/scanners to set up different types of scans

---

### (optional) Set up GCP PubSub emulator

(approach pinched from https://medium.com/google-cloud/use-pub-sub-emulator-in-minikube-67cd1f289daf)

Run 

```
$ kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gcp-emulator-deployment
  labels:
    app: pubsub
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pubsub
  template:
    metadata:
      labels:
        app: pubsub
    spec:
      containers:
        - name: pubsub-emulator
          image: gcr.io/google.com/cloudsdktool/cloud-sdk:414.0.0-emulators
          command: ["gcloud", "beta", "emulators", "pubsub", "start"]
          args: ["--project=abc", "--host-port=0.0.0.0:8085"]
---
apiVersion: v1
kind: Service
metadata:
  name: pubsub-service
spec:
  type: NodePort
  ports:
    - protocol: TCP
      port: 8008
      targetPort: 8085
      nodePort: 30001
  selector:
    app: pubsub
EOF
```

This will create a deployment named `gcp-emulator-deployment` and a service named `pubsub-service`

If you now run

`$ kubectl get service`

you should see something like

```
NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
kubernetes       ClusterIP   10.96.0.1      <none>        443/TCP          2d5h
pubsub-service   NodePort    10.96.93.173   <none>        8008:30001/TCP   9m14s
```

If you then run

`$ kubectl describe service pubsub-service`

you should see something like

```
Name:                     pubsub-service
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=pubsub
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.96.93.173
IPs:                      10.96.93.173
Port:                     <unset>  8008/TCP
TargetPort:               8085/TCP
NodePort:                 <unset>  30001/TCP
Endpoints:                10.244.0.19:8085
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

This is saying that `10.244.0.19:8085` is the port/address of the pod, and `10.96.93.173` is the internal IP address of the service.

If you now run

`$ kubectl port-forward service/pubsub-service 8035:8008 &`

your PubSub emulator instance will be externally accessible at 127.0.0.1:8035

## Suggested usage

In general, you probably want to install your code to its own dedicated namespace on your local Kubernetes cluster. This allows you to use the same node names as will be used in production, and thus remove a key point of difference between your environments.

### Security policy development & testing

Tools such as OPA, conftest, and checkov are ideal for creating executable security policies. Execution of these policies can be implemented within production CI pipelines, but it can be difficult to create a suitable test environment for these security policies.

A local Kubernetes environment makes an ideal test platform for these security policies, particularly if OPA, conftest and/or checkov are provided as part of a development container. This allows a completely containerised environment to be used for this type of testing.

Information on how to run OPA inside a container is provided at https://hub.docker.com/r/openpolicyagent/opa/

A vendor-provided, containerised conftest image is provided at https://hub.docker.com/r/openpolicyagent/conftest

A vendor-provided, containerised checkov image is provided at https://hub.docker.com/r/bridgecrew/checkov

### Testing container image signing, tagging & validation

If you're building artifacts inside your local Kubernetes environment, you might also want to consider signing, tagging and validating those images inside your environment. This is now considered best practice for securing your software supply chain, and the workflow can be built and tested entirely within your local Kube dev environment before moving it to production.

`cosign` (https://github.com/sigstore/cosign) is a very useful tool for signing/tagging/validating container images. By combining `cosign` with your own private Docker image registry (see below) it's possible to build and test a CI workflow that includes signing/tagging/validating the images you build.

Note that it's possible to sign your built container images using secrets created within Kubernetes itself. Doing so could be used to ensure that your signed containers don't leak to production later. Kubernetes-signed keys can be used by cosign by running `cosign generate-key-pair k8s://namespace/secretName`, substituting your own namespace & secretName into the command.

`cosign` itself can be run inside a container, using the image provided at https://hub.docker.com/r/bitnami/cosign/. This removes the need to install `cosign` natively on your laptop. If you go with this approach for a client engagement, it's best to build and store cosign's container image inside a controlled registry.

### API functional testing

Tools such as Karate can be run inside a container, and GitOps processes can be used to re-run Karate tests each time changes are applied to a repo.

### UI functional testing

UI automation tools such as Playwright and Puppeteer can be configured to run headless. Online patterns exist to run either tool in headless mode inside a container.

It is possible to deploy e.g. Playwright on a separate node in your local Kubernetes environment. It's also possible to create a Kubernetes BatchJob to start this node and execute tests, and link it to a GitOps workflow so the BatchJob (i.e. your UI test cases) executes whenever new UI tests are merged to a specific repo branch.

#### Testing against stubbed backends

You may wish to deploy a tool such as Hoverfly (https://hoverfly.io) to its own namespace on your Kubernetes dev environment, then use Hoverfly as a stub engine when testing your application. This approach will allow you to implement version control and GitOps workflows for your Hoverfly configurations, separate from your application code and config.

### Performance testing

Various tools such as SpeedScale, Gatling, Locust, k6 are all well-suited to running in a Kubernetes environment. Information for configuring these tools in a Kubernetes context are available in several different places.

Note that you will likely encounter problems if you try to run production-like performance workloads on a laptop Kubernetes environment. A local Kubernetes environment is best suited to writing, executing & debugging performance tests at very low volumes, then those volumes can be increased to run the same tests in a more production-sized environment.

## FAQs

### How do I recover my Kube environment after a reboot?

1. restart podman with `podman machine start`
2. restart your Kubernetes cluster with `podman restart kind-control-plane`

Your Kubernetes cluster should now be recovered, including all namespaces and tools+configs inside those namespaces

### How do I update the versions of tools?

podman and kind can be updated by running `brew upgrade && brew cleanup` on a Mac

VS Code will alert you when a new version is available, and can then be upgraded inside VS Code itself.

Tools such as `gatekeeper`, `argo-cd` and `cert-manager` that run inside the Kubernetes cluster will have their own process for deploying updates. You'll need to follow the instructions for each of these tools, which will typically involve a small number of `kubectl` commands

### How can I set up a private Docker registry inside my Kubernetes local environment?

Depending on your workflow and use case, it can be handy to be able to build, push and pull Docker images to a private registry that you control. This allows you to build and test CI workflows that include an artifact build step, without potentially corrupting a shared Docker registry that others rely on.

Follow the instructions at https://www.linuxtechi.com/setup-private-docker-registry-kubernetes/ to setup your own private Docker registry running inside your local Kubernetes cluster
