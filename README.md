# capabilities-shim-gitops

This repository is used to demonstrate how to effectively adopt Crossplane in GitOps by using a sample application. The sample application is based on another demo project [capabilities-shim](https://github.com/morningspace/capabilities-shim) that I created to prove the technical feasibility of composing variant existing software capabilities using Crossplane to come up with solution that meets user specific needs.

![](docs/images/crossplane-in-gitops.png)

## What it covers?

The repository includes:

* A bunch of well-organized YAML manifests: Represents variant software modules that can be synchronized and deployed by Argo CD to target cluster. This can be taken as a reference for people to evaluate what a GitOps repository can look like when using Crossplane.
* Documents in `docs` folder: Discuss best practices, common considerations, and lessons learned that you might experience as well when use Crossplane in GitOps.
* A shell script: Help you launch a GitOps demo environment in minutes including a KIND cluster with Argo CD installed.

Follow below instructions to luanch the demo environment. After it is up and running, you can explore the repository, follow the documents in `docs` folder, and experiment to get hands-on experience of using Crossplane in GitOps.

## Play with the demo environment

### Launch the demo environment

Please fork this repository to your own GitHub account, then follow below steps to launch the GitOps demo environment.

Go to project root folder and run below command to bring up the environment.

```shell
./scripts/install.sh up
```

It can help you launch a KIND cluster with an Argo CD instance installed. Besides that, it will also install all the necessary command line tools that you may need when you experiment with GitOps, e.g.: argocd CLI, kubeseal CLI.

At the end of the install process, it will print all softwares with their versions installed on your demo environment. Such as below:

```console
👏 Congratulations! The GitOps demo environment is available!
It launched a kind cluster, installed following tools and applitions:
- argocd cli v2.1.5
- kubeseal cli v0.16.0

To access Argo CD UI, open https://localhost:9443 in browser.
- username: admin
- password: ****************

For tools you want to run anywhere, create links in a directory defined in your PATH, e.g:
ln -s -f /root/Code/capabilities-shim-gitops/scripts/.cache/tools/linux_x86_64/kubectl-v1.17.11 /usr/local/bin/kubectl
ln -s -f /root/Code/capabilities-shim-gitops/scripts/.cache/tools/linux_x86_64/kind-v0.11.1 /usr/local/bin/kind
ln -s -f /root/Code/capabilities-shim-gitops/scripts/.cache/tools/linux_x86_64/argocd-v2.1.5 /usr/local/bin/argocd
ln -s -f /root/Code/capabilities-shim-gitops/scripts/.cache/tools/linux_x86_64/kubeseal-v0.16.0 /usr/local/bin/kubeseal
```

It also prints the Argo CD UI address, the username and password that you can use to login.

### Argocd cli login for cli operations
```shell
argocd login localhost:9443 --username admin
Proceed insecurely (y/n)? y
Password:
```

### Install everything else using Argo CD

Login to Argo CD from UI, then create an Argo Application using the following values:

| Field            | Value                                                    |
| ---------------- | -------------------------------------------------------- |
| Application Name | shared-apps                                              |
| Path             | config/shared                                            |
| Project          | default                                                  |
| Sync policy      | Automatic                                                |
| Self Heal        | true                                                     |
| Repository URL   | https://github.com/morningspace/capabilities-shim-gitops |
| Revision         | HEAD                                                     |
| Cluster URL      | https://kubernetes.default.svc                           |

You can also do the same thing using argocd CL from command line as below:

```shell
argocd app create shared-app --repo https://github.com/morningspace/capabilities-shim-gitops.git \
  --path config/shared \
  --dest-namespace default \
  --dest-server https://kubernetes.default.svc
```

This will install Crossplane with its providers, OLM, Sealed Secrets Controller, etc. to your demo environment.

After everything is up, run below command to generate and update the `cluster-config` secret encrypted at local by kubeseal for your demo environment, then check it in git. This is required for the Crossplane provider to access to the KIND cluster.

```shell
./scripts/install.sh cluster-config
git add environments/dev/env/cluster-config.json
git commit -m "Update cluster-config.json"
git push
```

Then create an Argo Application that represents the environment from UI using the following values:

| Field            | Value                                                    |
| ---------------- | -------------------------------------------------------- |
| Application Name | dev-env                                                  |
| Path             | environments/dev/env                                     |
| Project          | default                                                  |
| Sync policy      | Automatic                                                |
| Self Heal        | true                                                     |
| Repository URL   | https://github.com/morningspace/capabilities-shim-gitops |
| Revision         | HEAD                                                     |
| Cluster URL      | https://kubernetes.default.svc                           |


Congratulations! Now you can explore the repository, follow the documents in `docs` folder, and experiment with this environment to get hands-on experience of using Crossplane in GitOps.

### Troubleshooting

When install Argo CD, some pod may fail to install due to `ImagePullBackOff` error. This is because the image comes from Docker Hub and you do not specify image pull secret. This can be fixed by run below command:

```shell
./scripts/install.sh patch-pull-secret
```

### Clean up

Run below command to delete the demo environment completely from your local.

```shell
./scripts/install.sh down
```
