<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Using Crossplane in GitOps, Part I](#using-crossplane-in-gitops-part-i)
  - [Why Crossplane?](#why-crossplane)
  - [Bootstrap: Deploy Crossplane](#bootstrap-deploy-crossplane)
    - [Setup ProviderConfig](#setup-providerconfig)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Using Crossplane in GitOps, Part I: Bootstrap

In this series of articles, I will share my recent study on using Crossplane in GitOps. I will use Argo CD as the GitOps tool to demonstrate how Crossplane can work with it to provision applications from git to target cluster. Meanwhile, I will also explore some best practices, common considerations, and lessons learned that you might experience as well when use Crossplane in GitOps.

This article particularly focuses on why Crossplane fits into GitOps and how to bootstrap Crossplane using GitOps. You can find the demo project on GitHub at: https://github.com/morningspace/capabilities-shim-gitops.

## Why Crossplane?

GitOps is essentially a way to host a git repository that contains declarative descriptions of target system as desired state and an automated process, usually driven by a GitOps tool such as Argo CD or Flux CD, to make the target system match the desired state in the repository, so that when you deploy a new application or update an existing one, you just need to update the repository, then the automated process handles everything else.

Crossplane, on the other side, is a Kubernetes add-on that enables you to assemble infrastructure from multiple vendors and applications on top of that to expose them as high level self-service APIs for people to consume.

There are a few reasons that I can see how Crossplane can fit into GitOps realm quit well:

* Since Crossplane is designed to assemble infrastructure from multiple vendors, it makes it a lot easier to practice GitOps for application deployment across different vendors, typically public cloud vendors such as Amazon AWS, Google Cloud, Azure, IBM Cloud, etc. in a consistent manner.

* With the help of its powerful composition engine, Crossplane allows people to compose different modules from infrastructure, service, to application as needed in a declarative way, where we can check these declarative descriptions into git for GitOps tools to pick up easily.

* Crossplane allows people to extend its capabilities using Provider that can interact with different backends. There is a large amount of providers available in community and it is still actively evolving. By using variant providers, we can turn many different backends into something that are Kubernetes friendly, so that the desired state can be described using Kubernetes custom resource, then check into git and driven by GitOps tools.

> A good example such as Terraform provider allows people to integrate existing Terraform automation assets into Crossplane and modeled as Kubernetes custom resource. See: https://github.com/crossplane-contrib/provider-terraform

Turn everything into Kubernetes API no matter what kind of backend it is so that can be handled consistently using Kubernetes native way. As a result, those Kubernetes native objects are what we store in git which drive the GitOps flow. Using Crossplane opens the door to **gitopsifying** everything for us!

## Bootstrap: Deploy Crossplane

When using Crossplane, usually you will have Crossplane with a set of providers and optionally configuration packages installed on your cluster. Since Crossplane works as a control plane to drive infrastructure and application provisioning and composing, it can be co-located with Argo CD on the same cluster as a GitOps control plane.

Interestingly, if you have already installed Argo CD, you can take Argo CD as a top level "installer" which can help to install everything else including Crossplane itself. The only thing you need to do is to create an Argo `Application` that points to the helm repository where hosts the Crossplane helm charts. Check below YAML manifest into git, you will see Argo CD drives the Crossplane installation automatically.

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: crossplane-system 
spec:
  finalizers:
  - kubernetes 
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: crossplane-app
  namespace: argocd
spec:
  destination:
    namespace: crossplane-system
    server: https://kubernetes.default.svc
  project: default
  source:
    repoURL: https://charts.crossplane.io/stable
    chart: crossplane
    targetRevision: 1.4.1
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

For provider installing, it can also be handled as such if the provider has been packaged and published so that it can be described using Crossplane manifest.

Here I'm using [capabilities-shim](https://github.com/morningspace/capabilities-shim) as a demo project to demonstrate the Crossplane use in GitOps. There is a Crossplane configuration package which includes some pre-defined Composition and CompositeResourceDefinition manifests. It also depends on [an enhanced version](https://github.com/morningspace/provider-kubernetes) of [provider-kubernetes](https://github.com/crossplane-contrib/provider-kubernetes). In order to install both the configuration package and the provider, just need to check a `Configuration` resource in git as below:

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: capabilities-shim
spec:
  ignoreCrossplaneConstraints: false
  package: quay.io/moyingbj/capabilities-shim:v0.0.1
  packagePullPolicy: IfNotPresent
  revisionActivationPolicy: Automatic
  revisionHistoryLimit: 0
  skipDependencyResolution: false
```

Crossplane will then extract the package, install all resources included, along with the dependent provider all automatically.

### Setup ProviderConfig

For most Crossplane providers, you need to setup ProviderConfig before it can connect to the remote backends. In our case, provider-kubernetes needs below `ProviderConfig` in order to connect to the target cluster to provision the actual applications. 

```yaml
apiVersion: kubernetes.crossplane.io/v1alpha1
kind: ProviderConfig
metadata:
  name: provider-config-dev
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: dev
      name: cluster-config
      key: kubeconfig
```

This can also be checked into git. However, there are couple of things need to note.

The above `ProviderConfig` requires a secret referenced by `spec.credentials.secretRef` to be created beforehand. The secret includes kubeconfig that maps local or remote clusters for the provider to connect to. This needs to be checked in git too if you want Argo CD to help you create it rather than manually create by yourself.

You should never commit raw secrets in git. Before the secret can be stored in git, it needs to be encrypted at first. This can be done by sealed secrets introduced by the [Bitnami Sealed Secrets Controller](https://engineering.bitnami.com/articles/sealed-secrets.html). I will skip the detailed introduction about sealed secret here but you can google and learn how it works. Again, Sealed Secrets Controller can be installed using Argo CD by checking below Argo `Application` into git:

```yaml
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sealed-secrets-controller
  namespace: argocd
spec:
  destination:
    namespace: argocd
    server: https://kubernetes.default.svc
  project: default
  source:
    repoURL: https://bitnami-labs.github.io/sealed-secrets
    targetRevision: 1.16.1
    chart: sealed-secrets
    helm:
      values: |-
        # https://github.com/argoproj/argo-cd/issues/5991
        commandArgs:
          - "--update-status"
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Then you need to generate a secret including the kubeconfig information for your target cluster, and use the `kubeseal` command line tool to encrypt it into a sealed secret. For example:

```console
kubectl create secret generic cluster-config --from-literal=kubeconfig="`cat path/to/your/kubeconfig`" --dry-run -o yaml > cluster-config.yaml
kubeseal -n dev --controller-namespace argocd < cluster-config.yaml > cluster-config.json
```

After the encrypted secret is generated, check in git and let Argo CD synchronize it to target cluster.

> There are alternative approaches to handle secrets in GitOps, e.g. store secrets in external storage such as HashiCorp Vault, then store the secret key in git. It is not covered in this article.

Now that you have launched the environment with the prerequisites ready including Crossplane, its providers, and all the necessary configuration, you can start to check application manifests in git to trigger the application provisioning driven by GitOps. In next article, I will explore several ways that I experimented when using Crossplane and Argo CD to provision applications.

*(To be continued)*
