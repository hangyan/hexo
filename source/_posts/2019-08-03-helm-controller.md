---
title: What is Helm Doing Wrong and How a Helm3 Controller Can Fix It
toc: true
categories: 技术
date: 2019-08-03 11:26:30
excerpt: welcome to the controller!
tags: [Kubernetes,docker,helm]
---

Helm is big success for sure, it's nearly the standard application package format in kubernetes.You only need to provide some metadata about  your application's name, version, description….Helm can help you package up and upload to a central or custom chart repo.Just like npm,rpm,docker image, or whatever other package management system.

About a year ago, Helm3 was drafted. Since it's still on the proposal stage, most users are still using helm2, include us. The journey we spent with helm was not a very pleasant one, we struggled very hard to make it work. In the end, we decided to create a kubernetes controller based on Helm3 proposal, it works well and we open source it on github: [captain](https://github.com/alauda/captain).

This articles will describe what problem we have encountered when we are using helm2 and how we fix it in captain.



## Why is There a Tiller Server?

I remembered reading a article about a helm3 intro talk, when the speaker is showing the slide about removing tiller in helm3, the audience is so exited they applause for a very long time. Ah, awaked moment...Yes, everyone hates Tiller.

So, back to the beginning. From the helm doc:

> Tiller is the in-cluster component of Helm. It interacts directly with the Kubernetes API server to install, upgrade, query, and remove Kubernetes resources. It also stores the objects that represent releases.

Seems promising. But in the real scenario, it introduced tons of problems:

1. complicates RBAC since it's not using the RBAC of the user or pod running the helm commands to read/write kubernetes resources.
2. all releases should have a unique name within a tiller.  If you are using a global tiller(yes, a cluster can have multiple tillers...), means that all your release name must be unique across all namespaces, even when the actual release info is stored in a namespace based secret or configmap.Often you have to choose long names and it cause a lot of problems in kubernetes service discovery.
3. can cause lots of version conflicts between helm clients + tiller versions

So, before the helm3 can be production ready, users in the community have  to lives with it, enormous issues has been reported on this topic:

* [Error: could not find tiller](https://github.com/helm/helm/issues/5105)
* [Helm says tiller is installed AND could not find tiller](https://github.com/helm/helm/issues/4685)
* [Problem installing helm tiller](https://gitlab.com/gitlab-org/gitlab-ee/issues/11725)
* [Authentication problem when installing something](https://github.com/helm/helm/issues/5389)

I believe a lot of people who just want to install a chart have stuck at the first place: let tiller running normally!

And, even this is harder, but there is still some solutions to avoid it when using helm2:

* [How to avoid Tiller](https://jenkins-x.io/news/helm-without-tiller/)
* [Tillerless Helm v2](https://rimusz.net/tillerless-helm)



Since helm3 remove tiller and the core helm code is in it's alpha stage, we choose to create a kubernetes controller using helm code as a library(also a proposal in helm3). Of course the helm3 code is not production ready, and we have to do some modification to make the controller working properly, but the result is promising. The workflow around helm also changed, from a `helm install` command with various command line args to a HelmRequest CRD, this also brought some advantages:

* The controller can automatic retry install/upgrade if failed
* Since  HelmRequest CRD  is a simple yaml file, it can be saved to the source code repo
* store common configs in a configmap or secret allow multiple HelmRequest resource to ref to
* ...



## Resource Conflicts

The helm v3 proposal does't seems to  mention this at all, but i believe it's a much worse problem than Tiller. You want to install a chart, some resources already exist, it failed. you upgrade it, somehow it failed again, you delete it and reinstall it again, it failed. I believe most of the users have encountered this kind of problem. 

In fact, this problem drives us crazy, just because we have a few charts  which contains dozens of resources and a few of  crds. Constantly, it fails when we deploy these charts. This is the directly reason why we make the effort to create captain, the helm controller.

There is no need to paste github issues or stackoverflow questions on this topic, just search 'helm install failure' or 'helm upgrade failure' in google. you will find numerous results on this topic. The underlying reason is simple: the developer of helm didn't think this is problem. Why? doesn't matter now,  the consequences has prove that this is a totally wrong deceision no matter what .

Just when i'm writing this, we have encounter a problem liks this:

* [Error: UPGRADE FAILED: no resource with the name "anything_goes" found](https://github.com/helm/helm/issues/3275)

Yes, we want to upgarde a release, but it reported this error message. It's very confusing, and i'm tired of tracing these kind of  issues, let me  quote one  of the responses from this github issue:

![image-20190803142538909](/images/helm/rc.png)

Let's see how kubectl handle this problem, it has a `kubectl create` command, and also a `kubectl apply` command, it's logic is very simple:

* if the resource not exist, create
* if it exist, update it 

I think `kubectl apply` is one of the most heavily used command in kubectl. Since helm3 does not intend to fix this issue, we have fix it in captain, by adding anther resource handle layer on top of helm3. It's not perfect, but it works well.



## Hook Management

Hooks is a very powerful tool, if only you used it correctly. Helm's hook is bit of odd and confusing, I think it's happens a lot when you use helm. It's hard to guess what helm does under the hood.

For example, the `crd-install` hook. It's can be used to ensure helm install these CRDs first, so the upcoming CRs can be installed correctly (the apiserver need sometime to update it's discovery info). But, if you add crd-install hook to a CRD, the CRD is not a normal resource to helm anymore. If you delete the release, the CRD will not be deleted. Next time you install the same chart again, boom, the install will failed. Why? because it already exist. Even worse, it seems sometimes you cannot upgrade a crd with crd-install hook:

* [crd-install hook is not working on upgrade](https://github.com/helm/helm/issues/4697)

We have encountered this problem once, we don't know what to do. We have to delete the CRD, just like Thanos doing his finger snap.. a lot of related resource was gone….



Helm also provide an alternative solution: put all your CRDs in a single chart. Install it first, and then install other charts later. This actually is not helpful in many circumstances. Usually wen we hit this problem, we already have a first version of the charts's structure. If we move the crds around, we will have to do a upgrade(migration), again, either the upgrade failed, or the same resource conflicts problem occurs as described above.

In `captain`, since we choose an `apply` logic when create/update resource , non of this will happens again.



## The Default Release Name

If you install a helm chart, and does not specific a release name, helm will generate a random one for you, in the same format when docker create containers, such as `listening-rodent`,`mortal-kudu`.



Docker choose this name format, because they assign each container a uuid and users often use this uuid, so the names often doest not matter. But in helm release, this name is important, you need use this name to upgrade and delete a release, and helm generated this crap for you.



So, wen we design the HelmRequest CRD, the obvious and most simple choice is : If use supply a release name, use it, If not, use the HelmRequest's name. Since HelmRequest is namespace based, there is no need to worry about name conflicts any more.

For example, a origin resource description is :

```yaml
kind: HelmRequest
apiVersion: app.alauda.io/v1alpha1
metadata:
  name: nginx-ingress
spec:
  chart: stable/nginx-ingress
  
```

Turns to after created:

```yaml
apiVersion: app.alauda.io/v1alpha1
kind: HelmRequest
metadata:
  finalizers:
  - captain.alauda.io
  name: nginx-ingress
  namespace: default
spec:
  chart: stable/nginx-ingress
  namespace: default
  releaseName: nginx-ingress
  clusterName: ""
  installToAllClusters: false

```



## Conclusion

This list can go on and go on, different users may suffer from different issues. The helm3 proposal seems quite promising, and it brought a lot of new awesome features, and we are very looking forward to it to become production ready.

Again, weclome to check the [captain](https://github.com/alauda/captain) project out, all suggestions and PRs are welcome.

## Links

1. [Helm Glossary](https://helm.sh/docs/glossary/)
2. [Helm 3 Design Proposal](https://github.com/helm/community/blob/master/helm-v3/000-helm-v3.md)







