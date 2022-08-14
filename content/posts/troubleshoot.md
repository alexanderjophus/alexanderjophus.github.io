---
title: "TROUBLESHOOT (A guide on debugging kubernetes)"
date: 2022-08-14T11:02:31Z
draft: false
tags: ['troubleshoot']
---

## Intro

Disclaimer: This post is about Replicated's Troubleshoot - I work at Replicated.

Kubernetes is _hard_.
I thought I understood kubernetes; I had provisioned clusters from scratch, I had installed istio and a whole bunch of other tools, I had fixed outages in the past.
I then joined a organisation whose expertise were kubernetes, and oh boy did I have a lot to learn.

This isn't a blog post of what I've learned, I think that's a useless post, it'd be irrelevant in 3 weeks.
This _is_ a post on how to debug clusters using concepts from troubleshoot, which I'm hoping will stay relevant for a little while longer.

## The Basics

This section will be brief as we just make sure everyones up to speed on the basics of debugging kubernetes clusters.

The kubernetes docs have a great set of resources for [monitoring, logging and troubleshooting](https://kubernetes.io/docs/tasks/debug-application-cluster/).

Another great resource is this [flowchart](https://learnk8s.io/troubleshooting-deployments).

Lastly I want to highlight a few tools to help visualise the cluster, personally I'm a visual learner, and the greater insight into the cluster I have the better.
So for starters here's [k9s](https://github.com/derailed/k9s), it's a CLI visualisation tool that describes itself as a way to manage your cluster - but there's a few visualisation tools within it.
Secondly, the visualisation choice of many [lens](https://github.com/lensapp/lens) (it's on my todo list, to use on stream), which just looks phenominal.

This should serve as a baseline for everyone to get up to speed.
So let's get started with the main course!

## Introducing [Troubleshoot](https://troubleshoot.sh/)

There are 2 major sides to troubleshoot, preflight checks and support bundles.

Preflight checks are inteded to be ran before an application is installed onto a cluster, they allow the administrator to know things like is there enough CPU for my application, is the kubernetes version at least version X, do certain secrets exist, etc.

Support bundles, are subtly different, their main use case is "something broke, what can I inspect and share to software maintainers?".
They allow the cluster administrators to know the same thing as preflight checks, however they export a file that users can explore and share after the fact.

Both of these have 3 components; collectors, redactors, and analyzers.

**Collectors** allow administrators to collect (as the name implies) details around their cluster - from logs, to cluster information and everything in between.

**Redactors** (again as the name implies), allow users to redact information.
The use case here being that a collector may collect sensitive information such as database connection strings.
This step allows us to remove any sensitive information in a couple of different ways.
By default there are a few redactors run.

Lastly we have **analyzers** (also excellently named if I may add), instead of manually going through an incredibly large amount of information, the analyzers allow users to quickly highlight issues with the cluster.

## Writing your own ...

For this section we'll just spin up an empty k8s cluster, with a couple extra little things.
Feel free to deploy as much or as little as you want to your cluster.

Let's create a simple password secret with the following command

```sh
kubectl create secret -n default generic mysecret --from-literal=password=hunter2
```

Once your demo application (or real application) is ready to be troubleshot (troubleshooted?), we'll explore how to write collectors, redactors, and analyzers.

### Writing your Collector

There are many types of collectors to choose from, varying from host level information to copy files from pods.
We're going to copy a file from a pod.

```yaml
apiVersion: troubleshoot.sh/v1beta2
kind: SupportBundle
metadata:
  name: example
spec:
  collectors:
    - copy:
        selector:
          - run=busybox
        namespace: default
        containerPath: /etc/foo
        containerName: busybox
```

This will copy the path `/etc/foo` from the pod with the label `run=busybox` into the support bundle.

There are so many more [collectors](https://troubleshoot.sh/docs/collect/all/), I'd heavily recommend exploring them, trying out ones that may be useful etc.

### Writing your Redactor

Redactors are slightly different in that they have their own manifest Kind (i.e. it's not SupportBundle or Preflight)

```yaml
apiVersion: troubleshoot.sh/v1beta2
kind: Redactor
metadata:
  name: example
spec:
  redactors:
  - name: all files
    removals:
      yamlPath:
      - password
```

There are a few different ways to [redact information](https://troubleshoot.sh/docs/redact/redactors/), we've chosen yamlPath as in our collector we're collecting a yaml file.

### Writing your Analyzer

These allows us to quickly identify issues with the cluster.

```yaml
apiVersion: troubleshoot.sh/v1beta2
kind: SupportBundle
metadata:
  name: example
spec:
  analyzers:
    - yamlCompare:
        checkName: Compare YAML Example
        fileName: default/busybox/busybox/etc/foo/secrets.yaml
        path: username
        value: "Alexander"
        outcomes:
          - fail:
              when: "false"
              message: The collected data does not match the value.
          - pass:
              when: "true"
              message: The collected data matches the value
```

Following our yaml obsession, we're going to use yamlCompare, this will allow us to specify a file, path and value to compare against.
We can also specify the pass and fail conditions.

As with collectors and redactors, there are also many [analysers](https://troubleshoot.sh/docs/analyze/)

## All Together

`secrets.yaml` 

```sh
kubectl create secret -n default generic mysecret --from-file=secrets.yaml
```

```yaml
username: Alexander
password: "12345678"
```

`deployment.yaml`

```sh
kubectl apply -f deployment.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  labels:
    run: busybox
spec:
  containers:
    - command:
      - sleep
      - "3600"
      image: busybox
      name: busybox
      volumeMounts:
        - name: foo
          mountPath: /etc/foo/secrets.yaml # needed for volumeMounts
          subPath: secrets.yaml # needed otherwise it's a symlink
          readOnly: true
  volumes:
    - name: foo
      secret:
        secretName: mysecret
        optional: false
```

`support-bundle.yaml` 
```sh
kubectl support-bundle -f support-bundle.yaml
```

```yaml
apiVersion: troubleshoot.sh/v1beta2
kind: SupportBundle
metadata:
  name: example
spec:
  collectors:
    - copy:
        selector:
          - run=busybox
        namespace: default
        containerPath: /etc/foo
        containerName: busybox
  analyzers:
    - yamlCompare:
        checkName: Compare YAML Example
        fileName: default/busybox/busybox/etc/foo/secrets.yaml
        path: username
        value: "Alexander"
        outcomes:
          - fail:
              when: "false"
              message: The collected data does not match the value.
          - pass:
              when: "true"
              message: The collected data matches the value
---
apiVersion: troubleshoot.sh/v1beta2
kind: Redactor
metadata:
  name: example
spec:
  redactors:
  - name: all files
    removals:
      yamlPath:
      - password
```

With this you should see a support bundle in your terminal. (you can ignore this if you used `--interactive`).

You can then share this bundle with the maintainers of your cluster, or anyone else that's interested

## Conclusion

To recap; we're created a secret, deployed a pod, and created a support bundle.
We have redacted the users password, and automatically confirmed the users username is correct.

Obviously this example is quite contrived, however hopefully it gets the point across of how automation can easily diagnose problems with your cluster.