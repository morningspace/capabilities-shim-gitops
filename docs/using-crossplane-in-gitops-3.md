<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Using Crossplane in GitOps, Part III](#using-crossplane-in-gitops-part-iii)
  - [Deploying Order](#deploying-order)
    - [Argo CD Syncwave](#argo-cd-syncwave)
    - [Dependency Resolving by Crossplane](#dependency-resolving-by-crossplane)
    - [Uninstall in Reversed Order](#uninstall-in-reversed-order)
  - [Using Hooks](#using-hooks)
    - [Argo CD Hooks](#argo-cd-hooks)
    - [Helm Hooks](#helm-hooks)
  - [Health Check](#health-check)
  - [Organizing Configuration](#organizing-configuration)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Using Crossplane in GitOps, Part III: Common Considerations

In this series of articles, I will share my recent study on using Crossplane in GitOps. I will use Argo CD as the GitOps tool to demonstrate how Crossplane can work with it to provision applications from git to target cluster. Meanwhile, I will also explore some best practices, common considerations, and lessons learned that you might experience as well when use Crossplane in GitOps.

This article particularly focuses on some common considerations that you may have when using Crossplane in GitOps such as deploying order, using hooks, health check, organizing configuration. You can find the demo project on GitHub at: https://github.com/morningspace/capabilities-shim-gitops.

## Deploying Order

### Argo CD Syncwave

It is a common case where an application may be composed of multiple modules and some modules depend on other modules. When GitOps tool synchronizes these modules to target cluster, they need to be applied in a specific order. As an example, the deployment of Crossplane needs to be finished prior to the deployment of a Crossplane configuration package. This is because the configuration package requires some Crossplane CRDs available before it can be deployed. These CRDs come from the Crossplane deployment.

In Argo CD, this can be solved by using syncwave. Syncwaves are used to order how manifests are applied or synchronized by Argo CD to the cluster. You can have one or more waves, that allows you to ensure certain resources are healthy before subsequent resources are synchronized.

Syncwave for a certain resource can be specified by using the annotation `argocd.argoproj.io/sync-wave` with a number. Resource annotated with lower number will be synchronized first. 

For example, in our case, we created Argo `Application` for Crossplane, OLM, and Sealed Secret Controller and they are all annotated with the same syncwave value. This is because they do not have dependencies with each other and can be deployed in parallel.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: crossplane-app
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "100"
  ...
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: olm-app
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "100"
  ...
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sealed-secrets-controller
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "100"
  ...
```

But for the Argo `Application` that represents the deployment of Crossplane configuration package, it needs to be applied after Crossplane is running. This is why its syncwave is a number larger the above ones. 

```yaml
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: crossplane-ext-app
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "150"
```

### Dependency Resolving by Crossplane

Using syncwave to guarantee the resource synchronization order is a very useful practice. But it also makes your GitOps solution coupled with a specific GitOps tool. For example, you may need to figure out a similar approach if you switch from Argo CD to Flux CD as Flux does not recognize the Argo CD syncwave annotations.

Crossplane, on the other hand, might help in this case. This is because it can provide a way to define the deploying order for a set of composable resources while remains neutral to GitOps tools. Although this has not been there yet, there is a design proposal in community called [Generic References](https://github.com/crossplane/crossplane/pull/2385) which is currently under discussion. This design can help us define resource dependency and do more than that.

The major goal of this design is to address the scenario where managed resource requires information that is not available when user creates the resource. For example, to create a Subnet, you need to supply a VPC ID. If you create the VPC at the same time, then the ID will be available only after the VPC creation. According to the proposal, all managed resources will have a top-level spec field that lists the generic references. They will be resolved in order. Failure of any one will cause reconciliation to fail. Below is a sample snippet for a managed resource which has a reference to another resource:

```yaml
spec:
  patches:
  - fromObject:
      apiVersion: v1
      kind: ConfigMap
      name: common-settings
      namespace: crossplane-system
      fieldPath: data.region
    toFieldPath: spec.forProvider.region
```

By defining references, we can ask Crossplane to help us resolve dependencies and patch missing fields as above if needed. Thus, we can keep deploying order without depending on Argo CD. Argo CD can simply synchronize all the resources in one go, then Crossplane will handle the deploying order for these resources properly.

Although this design has not been closed yet, a similar idea has been implemented in [the enhanced version](https://github.com/morningspace/provider-kubernetes) of provider-kubernetes. When an `Object` resource is define, the managed resource defined and handled by provider-kubernetes, it allows you to specify one or more references in `spec.references` using exactly the same syntax as it is defined in the above design.

For example, in our case, the `Object` resource for `ClusterServiceVersion` has a reference to the `Object` resource for `Subscription`, so the `ClusterServiceVersion` name can be resolved from the dependent `Object` via field path `status.atProvider.manifest.status.currentCSV` and be applied to the referencing `Object` at `spec.forProvider.manifest.metadata.name`:

```yaml
---
apiVersion: kubernetes.crossplane.io/v1alpha1
kind: Object
metadata:
  name: csv-my-logging-stack
  annotations:
    kubernetes.crossplane.io/managementType: "ObservableAndDeletable"
spec:
  references:
  - fromObject:
      apiVersion: kubernetes.crossplane.io/v1alpha1
      kind: Object
      name: sub-my-logging-stack
      fieldPath: status.atProvider.manifest.status.currentCSV
    toFieldPath: spec.forProvider.manifest.metadata.name
  forProvider:
    manifest:
      apiVersion: operators.coreos.com/v1alpha1
      kind: ClusterServiceVersion
      metadata:
        namespace: operators
  providerConfigRef:
    name: provider-config-dev
```

This approach has been adopted in the GitOps demo project: [capabilities-shim-gitops](https://github.com/morningspace/capabilities-shim-gitops). For more information on the enhanced version of provider-kubernetes, please check [this document](https://github.com/morningspace/capabilities-shim/blob/main/docs/enhanced-provider-k8s.md).

### Uninstall in Reversed Order

Although it may not be very often in a real customer environment where people want to uninstall the whole application stack completely, when you delete the resources from target cluster, usually it needs to be in a reversed order as opposed to the order used when the resources are applied.

Interestingly, Argo CD supports this case for its syncwave feature. There is [an issue](https://github.com/argoproj/argo-cd/issues/3211) reported for this as an enhancement and it has been implemented since v1.7. So, if I have two resources and specify the syncwave that resource a needs to be synchronized before resource b. When deleting these resources, it should delete resource b first, and then resource a.

On the other hand, Crossplane does not support this at the time of 	writing. It is also not covered in the [Generic References](https://github.com/crossplane/crossplane/pull/2385) design proposal. But [the enhanced version](https://github.com/morningspace/provider-kubernetes) of provider-kubernetes has implemented that. It is achieved by applying a set of finalizers to the managed resources sequentially. More on this can be found [in this section](https://github.com/morningspace/capabilities-shim/blob/main/docs/enhanced-provider-k8s.md#uninstall-order) from its design document.

## Using Hooks

To define and check the desired state of target cluster in git then have GitOps tool to synchronize it over is great. But in some cases, to run everything declaratively may not be realistic in real world. You may still have to prepare something imperatively using script before the synchronization starts or do some cleanup after it is completed.

### Argo CD Hooks

In Argo CD, this can be configured using [Resource Hooks](https://argo-cd.readthedocs.io/en/stable/user-guide/resource_hooks/). Hooks are ways to run scripts before, during, and after a synchronization happens. A hook is simply a Kubernetes resource, typically a pod or job, annotated with `argocd.argoproj.io/hook`. As an example, in our case, we defined below job as `PostSync` hook to make sure all ClusterServiceVersion resources are succeeded after synchronization is finished.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: post-install-job
  namespace: argocd
  annotations:
    argocd.argoproj.io/hook: PostSync
spec:
  template:
    metadata:
      name: post-install-job
    spec:
      containers:
        - name: check-csv
          image: quay.io/bitnami/kubectl:latest
          command:
            - /bin/sh
            - -c
            - |
              current_seconds=0
              limit_seconds=$(( $(date +%s) + 600 ))
              installing=0

              while [ ${current_seconds} -lt ${limit_seconds} ]; do
                if [ $(kubectl get csv -n "{{.Values.metadata.namespace}}" -o name | grep -c clusterserviceversion) -eq 0 ] || \
                   [ $(kubectl get csv -n "{{.Values.metadata.namespace}}" --no-headers | grep -vc Succeeded) -gt 0 ]; then
                  installing=1
                  echo "INFO: ClusterServiceVersion resources still installing."
                  sleep 1
                else
                  installing=0
                  break
                fi
                current_seconds=$(( $(date +%s) ))
              done

              if [ ${installing} -eq 0 ]; then
                echo "INFO: All ClusterServiceVersion resources are ready."
              else
                echo "ERROR: ClusterServiceVersion resources still not ready."
                exit 1
              fi
      restartPolicy: Never
      serviceAccountName: argocd-application-controller
  backoffLimit: 1
```

### Helm Hooks

Helm also supports hooks. It allows chart developers to intervene at certain points when a Helm release is deployed. Just like Argo CD, Helm uses special annotation `helm.sh/hook` to indicate whether a template, e.g.: a pod or job, will be treated as a hook.

Interestingly, Argo CD supports Helm hooks by mapping the Helm annotations onto Argo CD's own hook annotations. When Argo CD synchronizes Helm charts, a Helm hook can usually be transformed to an Argo CD hook without additional work. For example, the pre-install hook in Helm is equivalent to the PreSync hook in Argo CD, and the post-install hook in Helm is equivalent to the PostSync hook in Argo CD. For more details, please check the [Argo CD document](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/#helm-hooks) where it includes a mapping table between Helm hooks and Argo CD hooks.

However, one thing to note is that not all Helm hooks have equivalents in Argo CD. For example, there's no equivalent in Argo CD for Helm post-delete hook. That means if you define some job as post-delete hook in Helm, they will never be triggered in Argo CD. [An issue](https://github.com/argoproj/argo-cd/issues/7575) on this was opened in Argo CD community.

In Crossplane, there is no concept such as hooks. So, if you want to do something before or after a certain synchronization, you may still resort to Helm or Argo CD.

## Health Check

To check the desired state in git and ask Argo CD synchronize it is just one side. The other side is to ensure what you deployed is healthy by checking the actual state in target cluster. 

Argo CD provides built-in health assessment for several kubernetes resources. It can be further customized by writing your own health checks in Lua code. This is useful if you have a custom resource for which Argo CD does not have a built-in health check. You may find the more you rely on Argo CD for resource health check, the less you go back to check that by using kubectl from command line.

It is very common in Crossplane for its providers which have their own managed resources and do not have built-in health checks support in Argo CD. Thus, you should define your own health checks for Crossplane providers.

In our demo project, we defined custom health checks for those resource types handled by OLM and operators launched by OLM, also the checks for the managed resource type `Object` handled by provider kubernetes. All these custom health checks should be added to a ConfigMap called `argocd-cm` in the namespace where Argo CD is deployed. Again, this ConfigMap can be checked in git so that can be synchronized by Argo CD as well.

For example, below is a sample snippet for the ConfigMap which includes the custom health checks for the resource type `ClusterServiceVersion` handled by OLM and the custom resource `Elasticsearch` handled by Elasticsearch operator. Typically, you need to assess the resource healthiness by querying sub-fields under the resource status field. You can also construct a status message as below to reveal more detailed information:

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
  resource.customizations.health.operators.coreos.com_ClusterServiceVersion: |
    hs = {}
    hs.status = "Progressing"
    hs.message = ""
    if obj.status ~= nil then
      if obj.status.phase == "Succeeded" then
        hs.status = "Healthy"
      end
      hs.message = obj.status.message
    end
    return hs
  resource.customizations.health.elasticsearch.k8s.elastic.co_Elasticsearch: |
    hs = {}
    hs.status = "Progressing"
    hs.message = ""
    if obj.status ~= nil and obj.status.health ~= nil and obj.status.phase ~= nil then
      if obj.status.health == "green" and obj.status.phase == "Ready" then
        hs.status = "Healthy"
      end
      hs.message = "health = " .. obj.status.health .. " phase = " .. obj.status.phase
    end
    return hs
```

For the health check of `Object` resource handled by provider kubernetes, it is a bit more complex. 

```yaml
  resource.customizations.health.kubernetes.crossplane.io_Object: |
    hs = {}
    hs.status = "Progressing"
    hs.message = ""
    if obj.status ~= nil and obj.status.atProvider ~= nil then
      kind = obj.spec.forProvider.manifest.kind
      res = obj.status.atProvider.manifest
      if res ~= nil then
        if kind == "ClusterServiceVersion" then
          if res.status ~= nil then
            if res.status.phase == "Succeeded" then
              hs.status = "Healthy"
            end
            hs.message = res.status.message
          end
        elseif kind == "Elasticsearch" then
          if res.status ~= nil and res.status.health ~= nil and res.status.phase ~= nil then
            if res.status.health == "green" and res.status.phase == "Ready" then
              hs.status = "Healthy"
            end
            hs.message = "health = " .. res.status.health .. " phase = " .. res.status.phase
          end
        end
      end
    end
    return hs
```

## Organizing Configuration

As we all know, in GitOps, the git repository used to store desired state for target system is the single source of truth, but there is no single answer on how the desired state should be organized in git repository. Although each team may have its own consideration on this for variant reasons, there are some common suggestions as following:

- Place common configuration that can be applied to all environments in one single place. This may include the common infrastructure configuration, shared applications and their settings that are applicable to all environments. For example, in the demo project, we use `config/` folder to store all configuration required to deploy Crossplane and its provider, OLM, Sealed Secret Controller, as well as the Argo CD customization settings and RBAC settings.

- Place per-environment configuration in separate places and one place for each environment. Some environment may have its unique configuration which can be put in a place represents that specific environment. For example, there can be separate folders for dev, staging, and product environment. In our demo project, we use `environments/` folder to host all environment specific configuration, where we place the Crossplane ProviderConfig, the encrypted secret that represents the target cluster kubeconfig credentials, and so on. These are all configuration unique to each cluster.

- Use branch to track per-release configuration. It is a very common practice for developer to track code changes among different releases using git release branch, especially when you have multiple releases that need to be maintained at the same time. The same rules apply to configuration changes in GitOps. When you have multiple releases to support and the configuration keeps changing from release to release, you can create branch for each release.
Of course, it will bring additional effort to merge changes among branches and resolve merge conflicts when needed. The good thing is, all branching and merging practices that you have already been familiar with when dealing with code changes can also be applied to the configuration changes.

- Host multiple applications manifests in a monolithic repository. This is a very effective way to manage applications in a small project where you put all applications configuration manifests in a single repository. It can be stored in separate folders, one folder for each application.

- Host multiple applications manifests separately in multiple repositories. This is suitable for a large project in which you may have multiple products, each product has its own set of applications, and maintained by different team. By putting configuration manifests for these applications in separate repositories, you can leverage the sophisticated organization or repository membership and access control capabilities provided by the git infrastructure provider such as GitHub, to manage application deployment for each team differently in a fine grained manner.

- Use Helm to parameterize deployment manifests and turn into reusable templates. In some cases, you may want your application manifests to be configurable, e.g.: to allow Ops or SREs to choose which version to install, which storage to pick up, or which namespace to apply, etc. Helm as a deployment tool for Kubernetes application is widely used. It can help you extract parameters out of deployment manifest, and turn the manifest into a reusable template. Helm can also be used together with Crossplane, especially when you only use Crossplane providers to provision applications and, do not use its composition. In such case, Crossplane plays very similar role as Kubernetes controller or Operator does with its wide range of providers that extend the scope of what you can manage using GitOps.
 

- Use Kustomize to override deployment manifests as a base for a specific environment. In some cases, you may want your application manifests to be customized on a per environment basis. Instead of Helm, this can also be achieved by using Kustomize. You can put the manifests with default configuration in a folder as a base layer, then put environment specific manifest pieces in a separate folder that represents a specific environment to override the base layer. This can also be applied when you use Crossplane, especially when using composition. For example, you can define the default CompositeResourceClaim (XRC) at base layer, then override it at environment specific layer.

- Use App of Apps pattern when use Argo CD to mange a set of applications. In Argo CD, an `Application` resource is a unit that deploys a set of manifests. Since an Application is a Kubernetes resource, it can be managed by Argo CD too. Furthermore, an Application resource can manage multiple other Application resources. That is called App of Apps pattern. By using this pattern, you can deploy a set of applications in one go. Also, increasing or decreasing Application resources can be done by adding or removing manifests to git repository instead of operating Argo CD via its Web UI or command line.

Below is a sample structure for a git repository to demonstrate how folders can be organized base on the above discussion:

```console
├── config
│   ├── argocd-apps
│   ├── infra
│   ├── services
│   └── apps
└── environments
    ├── base
    └── overlays
        ├── dev
        ├── staging
        └── prod
```
