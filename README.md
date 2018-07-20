# Managing Helm releases the GitOps way

**What is GitOps?**

GitOps is a way to do Continuous Delivery, it works by using Git as a source of truth for declarative infrastructure and workloads. 
For Kubernetes this means using `git push` instead of `kubectl create/apply` or `helm install/upgrade`.

In a traditional CICD pipeline, CD is an implementation extension powered by the 
continuous integration tooling to promote build artifacts to production. 
In the GitOps pipeline model, any change to production must be committed in source control 
(preferable via a pull request) prior to being applied on the cluster. 
This way rollback and audit logs are provided by Git. 
If the entire production state is under version control and described in a single Git repository, when disaster strikes, 
the whole infrastructure can be quickly restored from that repository.

To better understand the benefits of this approach to CD and what are the differences between GitOps and 
Infrastructure-as-Code tools, head to Weaveworks website and read [GitOps - What you need to know](https://www.weave.works/technologies/gitops/) article.

In order to apply the GitOps pipeline model to Kubernetes you need three things: 

* a Git repository with your workloads definitions in YAML format, Helm charts and any other Kubernetes custom resource that defines your cluster desired state (I will refer to this as the *config* repository)
* a container registry where your CI system pushes immutable images (no *latest* tags, use *semantic versioning* or git *commit sha*)
* an operator that runs in your cluster and does a two-way synchronization:
    * watches the registry for new image releases and based on deployment policies updates the workload definitions with the new image tag and commits the changes to the config repository 
    * watches for changes in the config repository and applies them to your cluster

![gitops](https://github.com/stefanprodan/openfaas-flux/blob/master/docs/screens/flux-helm-gitops.png)

### Install Weave Flux

First step in automating Helm releases with [Weave Flux](https://github.com/weaveworks/flux) is to create a Git repository with your charts source code.
You can fork the [gitops-helm](https://github.com/stefanprodan/gitops-helm) project and use it as a template for your cluster config.

Add the Weave Flux chart repo:

```bash
helm repo add weaveworks https://weaveworks.github.io/flux
```

Install Weave Flux and its Helm Operator by specifying your fork URL 
(replace `stefanprodan` with your GitHub username): 

```bash
helm install --name flux \
--set rbac.create=true \
--set helmOperator.create=true \
--set git.url=ssh://git@github.com/stefanprodan/gitops-helm \
--set git.chartsPath=charts \
--namespace flux \
weaveworks/flux
```

The Flux Helm operator provides an extension to Weave Flux that automates Helm Chart releases for it. 
A Chart release is described through a Kubernetes custom resource named FluxHelmRelease. 
The Flux daemon synchronizes these resources from git to the cluster, 
and the Flux Helm operator makes sure Helm charts are released as specified in the resources.

Note that Flux Helm Operator works with Kubernetes 1.9 or newer. 
If you are running Tiller with TLS you'll need to configure the operator to use the client certificate, more details [here](https://github.com/weaveworks/flux/blob/master/chart/flux/README.md#installing-weave-flux-helm-operator-and-helm-with-tls-enabled).  

At startup Flux generates a SSH key and logs the public key. 
Find the SSH public key with:

```bash
kubectl -n flux logs deployment/flux | grep identity.pub | cut -d '"' -f2
```

In order to sync your cluster state with Git you need to copy the public key and 
create a **deploy key** with **write access** on your GitHub repository.

Open GitHub, navigate to your fork, go to _Setting > Deploy keys_ click on _Add deploy key_, check 
_Allow write access_, paste the Flux public key and click _Add key_.

### GitOps pipeline example

The config repo has the following structure:

```
├── charts
│   └── podinfo
│       ├── Chart.yaml
│       ├── README.md
│       ├── templates
│       └── values.yaml
├── hack
│   ├── Dockerfile.ci
│   └── ci-mock.sh
├── namespaces
│   ├── dev.yaml
│   └── stg.yaml
└── releases
    ├── dev
    │   └── podinfo.yaml
    └── stg
        └── podinfo.yaml
```

I will be using [podinfo](https://github.com/stefanprodan/k8s-podinfo) to demonstrate a full CI/CD pipeline including promoting releases between environments.  

Inside the *hack* dir you can find a script that simulates the CI process. 
The *ci-mock.sh* script does the following:
* pulls the podinfo source code from GitHub
* generates a random string that will server as the Git commit short SHA
* builds a Docker image with the format: `yourname/podinfo:branch-sha`
* pushes the image to Docker Hub

Let's create an image corresponding to the `dev` branch (replace `stefanprodan` with your Docker Hub username):

```
$ cd hack && ./ci-mock.sh -r stefanprodan/podinfo -b dev -v 1.0.0-alpha1

Sending build context to Docker daemon  4.096kB
Step 1/15 : FROM golang:1.10 as builder
....
Step 12/15 : COPY --from=builder /go/src/github.com/stefanprodan/k8s-podinfo/podinfo .
....
Step 15/15 : CMD ["./podinfo"]
....
Successfully built 71bee4549fb2
Successfully tagged stefanprodan/podinfo:dev-kb9lm91e
The push refers to repository [docker.io/stefanprodan/podinfo]
36ced78d2ca2: Pushed 
```

Inside the *charts* directory there is Helm chart for podinfo. 
Using this chart I want to create a release with the image I've just published to Docker Hub, but instead of editing the 
`values.yaml` from the chart source I will create a FluxHelmRelease definition: 

```yaml
apiVersion: helm.integrations.flux.weave.works/v1alpha2
kind: FluxHelmRelease
metadata:
  name: podinfo-dev
  namespace: dev
  labels:
    chart: podinfo
  annotations:
    flux.weave.works/automated: "true"
    flux.weave.works/tag.chart-image: glob:dev-*
spec:
  chartGitPath: podinfo
  releaseName: podinfo-dev
  values:
    image: stefanprodan/podinfo:dev-kb9lm91e
    replicaCount: 1
    hpa:
      enabled: false
``` 

Flux Helm release fields:

* `metadata.name` is mandatory and needs to follow Kubernetes naming conventions
* `metadata.namespace` is optional and determines where the release is created
* `metadata.labels.chart` is mandatory and should match the directory containing the chart
* `spec.releaseName` is optional and if not provided the release name will be $namespace-$name
* `spec.chartGitPath` is the directory containing the chart, given relative to the charts path
* `spec.values` are user customizations of default parameter values from the chart itself

With the `flux.weave.works` annotations I instruct Flux to automate this release.
When a new tag with the prefix `dev` is pushed to Docker Hub, Flux will update the image field in the yaml file, 
will commit and push the change to Git and finally will apply the change on the cluster. 
When the `podinfo-dev` FluxHelmRelease object changes inside the cluster, 
Kubernetes API will notify the Flux Helm Operator and the operator will perform a Helm release upgrade. 

Now let's assume that I want to promote the code from the `dev` branch into a more stable environment for others to test it. 
I would create a release candidate by merging the podinfo code from `dev` into the `stg` branch. 
The CI would kick in and publish a new image:

```bash
$ cd hack && ./ci-mock.sh -r stefanprodan/podinfo -b stg -v 1.0.0-beta1

Successfully tagged stefanprodan/podinfo:stg-9ij63o4c
The push refers to repository [docker.io/stefanprodan/podinfo]
8f21c3669055: Pushed 
```

Assuming the staging environment has some sort of automated load testing in place, 
I want to have a different configuration to my dev release: 

```yaml
apiVersion: helm.integrations.flux.weave.works/v1alpha2
kind: FluxHelmRelease
metadata:
  name: podinfo-stg
  namespace: stg
  labels:
    chart: podinfo
  annotations:
    flux.weave.works/automated: "true"
    flux.weave.works/tag.chart-image: glob:stg-*
spec:
  chartGitPath: podinfo
  releaseName: podinfo-stg
  values:
    image: stefanprodan/podinfo:stg-9ij63o4c
    replicaCount: 2
    hpa:
      enabled: true
      maxReplicas: 10
      cpu: 50
      memory: 128Mi
```


### Getting Help

If you have any questions about this Weave Flux Helm Demo:

- Checkout the weaveworks [helm integration guide](https://github.com/weaveworks/flux/blob/master/site/helm/helm-integration.md)
- Invite yourself to the <a href="https://weaveworks.github.io/community-slack/" target="_blank">Weave community</a> slack.
- Ask a question on the [#flux](https://weave-community.slack.com/messages/flux/) slack channel.
- Join the <a href="https://www.meetup.com/pro/Weave/"> Weave User Group </a> and get invited to online talks, hands-on training and meetups in your area.
- Send an email to <a href="mailto:weave-users@weave.works">weave-users@weave.works</a>
- <a href="https://github.com/stefanprodan/weave-flux-helm-demo/issues/new">File an issue.</a>

Your feedback is always welcome!
