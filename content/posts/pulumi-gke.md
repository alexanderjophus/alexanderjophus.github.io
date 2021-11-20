---
title: "INFRASTRUCTURE AS CODE (Pulumi + Go)"
date: 2021-11-20T17:30:21Z
draft: false
tags: ['iac']
---

## Intro

So you've been playing with the cloud for a bit, you've been fiddling with some dials, deploying some new services, and flipping some switches.
Production is running smoothly.
You get the new requirements to spin up a staging environment so you can test changes before they get into production.
Cool.
Wait, how did we set up prod again?

Introducing the concept `Infrastructure as Code`, in short this allows you to commit your infrastructure to your version control system.
Allowing you to deploy new environments really easily, automatically deploy changes, and roll back changes should you need to.

That sounds great, what options do I have?
The big player in the space right now is [Terraform](https://www.terraform.io/), however [Pulumi](https://www.pulumi.com/) has been gaining a lot of traction lately.
Today we're going to look at pulumi, though many of the concepts will stay the same.

## Getting Started

Recently I tweeted about the concept of speedrunning getting started documentation.
Pulumi does a great job of helping developers [get started](https://www.pulumi.com/docs/get-started/), no matter the users choice of cloud provider or language.

We'll be deploying to GCP using Go, so we run.

```sh
pulumi new gcp-go
```

Assuming we have our environment set up, we can then run;

```sh
pulumi up
```

This should create a bucket in GCP.
How swift was that?

## Deploying To GKE

To do this, we're going to need to import a few key components.
We're going to need the GCP project to ensure we can use the container service
We're going to need the container package to ensure we have the latest version of k8s installed and to create our cluster.
We're also going to need the kubernetes package to create a provider for us to use later.

```go
import (
    ...
	"github.com/pulumi/pulumi-gcp/sdk/v5/go/gcp/container"
	"github.com/pulumi/pulumi-gcp/sdk/v5/go/gcp/projects"
	"github.com/pulumi/pulumi-kubernetes/sdk/v3/go/kubernetes"
	"github.com/pulumi/pulumi/sdk/v3/go/pulumi"
    ...
)
```

Next, replace the NewBucket code, with the following.

```go
containerService, err := projects.NewService(ctx, "project", &projects.ServiceArgs{
    Service: pulumi.String("container.googleapis.com"),
})
if err != nil {
    return err
}

engineVersions, err := container.GetEngineVersions(ctx, &container.GetEngineVersionsArgs{})
if err != nil {
    return err
}
masterVersion := engineVersions.LatestMasterVersion

cluster, err := container.NewCluster(ctx, "demo-cluster", &container.ClusterArgs{
    InitialNodeCount: pulumi.Int(2),
    MinMasterVersion: pulumi.String(masterVersion),
    NodeVersion:      pulumi.String(masterVersion),
    NodeConfig: &container.ClusterNodeConfigArgs{
        MachineType: pulumi.String("n1-standard-1"),
        OauthScopes: pulumi.StringArray{
            pulumi.String("https://www.googleapis.com/auth/compute"),
            pulumi.String("https://www.googleapis.com/auth/devstorage.read_only"),
            pulumi.String("https://www.googleapis.com/auth/logging.write"),
            pulumi.String("https://www.googleapis.com/auth/monitoring"),
        },
    },
}, pulumi.DependsOn([]pulumi.Resource{containerService}))
if err != nil {
    return err
}

ctx.Export("kubeconfig", generateKubeconfig(cluster.Endpoint, cluster.Name, cluster.MasterAuth))

k8sProvider, err := kubernetes.NewProvider(ctx, "k8sprovider", &kubernetes.ProviderArgs{
    Kubeconfig: generateKubeconfig(cluster.Endpoint, cluster.Name, cluster.MasterAuth),
}, pulumi.DependsOn([]pulumi.Resource{cluster}))
if err != nil {
    return err
}
```

For brevity I'll link the [generateKubeconfig function](https://github.com/trelore/pulumi/blob/662eb960002abd22cc4caf1016643601a097debf/main.go#L152-L182).

Important things to note from the snippet above;
- `ctx.Export` will export a variable `kubeconfig`, which we can use outside of our pulumi script.
- `k8sProvider` ensures we have a provider to explicitly pass to future functions to ensure we're pointing at the right cluster.
- `pulumi.DependsOn` makes sure we have deployed everything in the correct order.

After all that if we run `pulumi up` again, we should have a cluster in GKE.
Sweet.

## Adding Monitoring

Next up, we want to deploy some monitoring tools, so that when we deploy any application to the cluster it has logs/metrics/traces all immediately available to use.
There are a _bunch_ of tools in this space.
For the sake of simplicity we're going to deploy the [loki-stack](https://grafana.com/docs/loki/latest/installation/helm/) helm chart, this comes bundled with logging and metric solutions, as well as a UI to visualise it all.

We're going to start by creating a new namespace for our monitoring tools - `monitoring` (creative, I know).

```go
import corev1 "github.com/pulumi/pulumi-kubernetes/sdk/v3/go/kubernetes/core/v1"
```

```go
ns, err := corev1.NewNamespace(ctx, "monitoring", &corev1.NamespaceArgs{
    Metadata: &metav1.ObjectMetaArgs{
        Name: pulumi.String("monitoring"),
    },
}, pulumi.Provider(k8sProvider))
if err != nil {
    return err
}
```

This just uses the `corev1` package to create a namespace.
We return the namespace object, as well as check for any errors. 
Note how we use the k8sProvider we defined earlier.
Now we just need to import the helm package, and deploy our helm chart.

```go
import helmv3 "github.com/pulumi/pulumi-kubernetes/sdk/v3/go/kubernetes/helm/v3"
```

```go
_, err = helmv3.NewChart(ctx, "loki-stack", helmv3.ChartArgs{
    Chart:     pulumi.String("loki-stack"),
    Version:   pulumi.String("2.5.0"),
    Namespace: ns.Metadata.Name().Elem(),
    FetchArgs: helmv3.FetchArgs{
        Repo: pulumi.String("https://grafana.github.io/helm-charts"),
    },
    Values: pulumi.Map{
        "loki": pulumi.Map{
            "enabled": pulumi.Bool(true),
        },
        "grafana": pulumi.Map{
            "enabled": pulumi.Bool(true),
        },
        "prometheus": pulumi.Map{
            "enabled": pulumi.Bool(true),
        },
    },
}, pulumi.Provider(k8sProvider), pulumi.DependsOn([]pulumi.Resource{ns}))
if err != nil {
    return err
}
```

By default loki is enabled, but I find it makes more sense to explicitly enable individual services.
We also reference the ns we used earlier to deploy our monitoring stack in the right namespace.
Lastly, we also depend on the namespace to be made before we deploy to it.

## Next Post

- We'll discuss how to use `crd2pulumi`
- How to deploy our own services to the cluster
- Walk through our monitoring stack