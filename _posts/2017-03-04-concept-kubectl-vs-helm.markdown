---
layout: post
title:  "Past experience on managing kubernetes cluster with kubectl and helm"
date:   2017-03-04 17:19:27 +0700
---

At some point of my past project, 
I've faced with two options when managing add-on on my kubernetes cluster: 
should I use `helm`, or should I stick with `kubectl` applying manual manifest?

Both methods lead to the same result, with different approach.

With `kubectl`, you need to have the manifest of what you are trying to create,
in which in later part you will need to apply that manifest to your cluster 
by issuing  `kubectl -f manifest.yaml`.

With `Helm`, you have `Helm Chart`, which is basically a collection of multiple manifest combined into a single package,
hosted in the repository that later can be retrieved by your cluster. 
To install a package, we simply run `helm install <package/chrat name>`.

Now this is highly opinionated, but considering:
- Several manifest can be merged into a single manifest file with triple dash `---` annotation
- Modern project will be kept in a VCS repository
- Inside the repo, the project source code will live inside a folder named `src` or `app`
- The root of the repo will usually contain things that are not directly related to the source code, such as documentation or manual
- All things related to deployments will also be kept inside the repo for future re-usability

Then my whole experiences so far will lead me to use `kubectl` instead of `helm`, 
unless I have something from helm that I didn't understand yet.

With manual `kubectl`, the developer have the freedom to research, modify, and keep a working manifest for certain package.

For example, if I want to deploy NGINX Ingress Controller and Certificate Manager for my cluster, 
the steps would go like these:

First, I want to find the official documentation of each, and find a link to the yaml manifest.
1. NGINX Ingress Controller v1.8.2: [manifest link](https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml) 
2. Certificate Manager v1.13.0: [manifest](https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml)

Upon opening these links, we can see that it is basically just a plain text formatted as yaml, and we call these manifests.

Now since we know `kubectl` accepts link, 
we have options for applying manifest online rather than offline (i.e. running locally).

For a certain straightforward application, you may want to do this the "online" way by issuing 
`kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml`.

But for a specific reason, which lead to me writing this article, is that maybe you want to modify something
with how the package is designed/structured before actually applying it on your cluster.

With each of the package being varied in their customization option, 
you can customize the manifest by downloading it first, save it locally as a yaml file, and modify it according to your need.

<br>
<br>

Let's see a real case: deploying NGINX Ingress Controller on Digital Ocean won't work straight out-of-the-box
by only applying the manifest from the official documentation.
Instead, some custom annotations are needed inside this particular Service resource:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx-controller
  annotations:
    service.beta.kubernetes.io/do-loadbalancer-enable-proxy-protocol: "true"
    service.beta.kubernetes.io/do-loadbalancer-hostname: "ingress.domain.com"
    service.beta.kubernetes.io/do-loadbalancer-tls-passthrough: "true"
```

The first custom annotation tells the Ingress Controller to act as a proxy
(as it is basically a Load Balancer resource and by default it will not act as a proxy).

The second annotation tells the Ingress Controller to conduct pod-to-pod communication via the ingress's public 
domain at "ingress.domain.com" instead of using private cluster address. 
This will fix nasty issue inside Digital Ocean's cluster where a pod 
can't communicate with each other using internal ingress address. (Of course, we need to assign the subdomain to A record fot his to be working).

The final annotation tells Ingress Controller to act as a TLS pass through.

With such requirements in mind, you may find it difficult to use online manifest or chart. 
I think helm has chart overriding feature, but it is way more organized to download the manifest locally. 
Simply modify it until it's working, and keep it.
Plus we got the benefit of version controlling by committing it inside our git repo. 
You ship the repo somewhere, the manifest will also be included.

This pattern is applicable for every add-on package you need to install on your cluster.
At first, you may need to learn how to directly configure the yaml. 
But with time, you will have a bunch of manifest files that can act as your baseline/template for your next project, 
and are highly reusable.

I would probably only need more experience on this subject, so until that time come I'll still stick with `kubectl` and plain manifest.
