---
title: "TROUBLESHOOT (A guide on debugging kubernetes)"
date: 2022-04-16T11:02:31-07:00
draft: true
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

We're going to focus on a couple of different collectors to show the range of information we can collect.
Firstly, we're just going add a couple of high level collectors. 

```yaml
apiVersion: troubleshoot.sh/v1beta2
kind: SupportBundle
metadata:
  name: sample
spec:
  collectors:
    - clusterInfo: {}
    - clusterResources: {}
```

The important thing to note here is `collectors` is a list.

The first thing in that list we're collecting is [clusterInfo](https://troubleshoot.sh/docs/collect/cluster-info/), this will give us information on the version of k8s being used.

Another high level collector we have is [cluster-resources](https://troubleshoot.sh/docs/collect/cluster-resources/).
At a high level if you can do `kubectl get <resource>`, this collector will find it (this is important for another blog post).

Lastly, we're also going to add a kubernetes secret collector.

```yaml
    - secret:
        namespace: default
        name: mysecret
        includeValue: true
        key: password
```

By default the value of the secret is not collected, but we can include it by setting `includeValue` to `true`.
We're going to include it so that we can redact it later (this is mostly for demonstration purposes).

There are so many more [collectors](https://troubleshoot.sh/docs/collect/collectors/), I'd heavily recommend exploring them, trying out ones that may be useful etc.

### Writing your Redactor

Redactors are incredibly useful tools to remove sensitive information from your collected data.
You can remove passwords from secrets, ip addresses from config maps, and PII from logs.
Or any combination of that and more!

Taking ip addresses as the example, we can remove specific ip addresses, or use regex to remove all ip addresses.

If we run the support bundle we have above, you'll notice our password is exposed to the world. Oh no!
This is a security issue, so we'll need to redact it.
  
```yaml
apiVersion: troubleshoot.sh/v1beta2
kind: SupportBundle
metadata:
  name: sample
spec:
spec:
  collectors:
    {{ as above}}
  redactors:
    - name: Redact password
      removals:
        regex:
        - selector: 'password'
          redactor: '("value": ").*(")'
```

### Writing your Analyzer

## All Together

`secrets.yaml` kubectl create secret -n default generic mysecret --from-file=secrets.yaml

```yaml
username: Alexander
password: "12345678"
```

`deployment.yaml` kubectl apply -f deployment.yaml

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
          mountPath: "/etc/foo"
          readOnly: true
  volumes:
    - name: foo
      secret:
        secretName: mysecret
        optional: false
```

`support-bundle.yaml` kubectl support-bundle -f support-bundle.yaml

```yaml
apiVersion: troubleshoot.sh/v1beta2
kind: SupportBundle
metadata:
  name: sample
spec:
  collectors:
    - copy:
        selector:
          - run=busybox
        namespace: default
        containerPath: /etc/foo
        containerName: busybox
  redactors:
  - name: all files
    removals:
      yamlPath:
      - "password"
  analyzers:
    - yamlCompare:
        checkName: Compare YAML Example
        fileName: /etc/foo/secrets.yaml
        path: "username"
        value: "Alexander"
        outcomes:
          - fail:
              when: "false"
              message: The collected data does not match the value.
          - pass:
              when: "true"
              message: The collected data matches the value.
```
