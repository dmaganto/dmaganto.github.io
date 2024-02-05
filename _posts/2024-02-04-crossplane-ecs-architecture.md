---
title: "Using Crossplane to provision ECS Architecture"
date: 2024-02-05T21:00:00+01:00
categories:
  - Blog
tags:
  - Crossplane
  - AWS
  - ECS
  - Architecture
---

This is an example about how to create ECS infrastructure using Crossplane. It creates a Cluster, Service and TaskDefinition resources for simplicity. The goal is to show how to use Crossplane to provision infrastructure and deploy workloads using helm charts on top of Crossplane compositions.

## Architecture

![ECS Architecture](/assets/diagrams/ecs_arch.png)

## Crossplane

Crossplane is a tool for provisioning and managing infrastructure on cloud providers. It uses Kubernetes Custom Resource Definitions (CRDs) to define the infrastructure resources and their configuration.

![ECS Architecture](/assets/diagrams/crossplane_arch.png)

## ECS Cluster
{% highlight yaml %}
apiVersion: microservices.dmaganto.infra/v1alpha1
kind: ECSCluster
metadata:
  name: {% raw %}{{ .Values.tenant }}{% endraw %}
spec:
  compositionSelector:
    matchLabels:
      dmaganto.infra/provider: upbound
      dmaganto.infra/cluster-type: tenant
  resourceConfig:
    providerConfigName: default
    tenant: {% raw %}{{ .Values.tenant }}{% endraw %}
    name: {% raw %}{{ .Values.tenant }}{% endraw %}
    deletionPolicy: Delete
    region: {% raw %}{{ .Values.region }}{% endraw %}
  parameters:
    clusterName: {% raw %}{{ .Values.clusterName }}{% endraw %}
{% endhighlight %}

## ECS Service
{% highlight yaml %}
apiVersion: microservices.dmaganto.infra/v1alpha1
kind: ECSService
metadata:
  name: {% raw %}{{ .Values.serviceName }}{% endraw %}
spec:
  compositionSelector:
    matchLabels:
      dmaganto.infra/provider: upbound
      dmaganto.infra/service-type: tenant
  resourceConfig:
    providerConfigName: default
    tenant: {% raw %}{{ .Values.tenant }}{% endraw %}
    name: {% raw %}{{ .Values.serviceName }}{% endraw %}
    deletionPolicy: Delete
    region: {% raw %}{{ .Values.region }}{% endraw %}
  parameters:
    clusterName: {% raw %}{{ .Values.clusterName }}{% endraw %}
    family: {% raw %}{{ .Values.serviceName }}{% endraw %}
    task_id: "{% raw %}{{ .Values.containerDefinitions | toString | sha256sum | trunc 10 }}{% endraw %}"
    desiredCount: {% raw %}{{ .Values.replicas }}{% endraw %}
    securityGroups: ["sg-xxxxxxxxxx"]
    subnets: ["subnet-xxxxxxxxxx","subnet-xxxxxxxxxx"]
{% endhighlight %}

## ECS TaskDefinition
{% highlight yaml %}
apiVersion: microservices.dmaganto.infra/v1alpha1
kind: ECSTaskdef
metadata:
  name: {% raw %}{{ .Values.serviceName }}-{{ .Values.containerDefinitions | toString | sha256sum | trunc 10 }}{% endraw %}
spec:
  compositionSelector:
    matchLabels:
      dmaganto.infra/provider: upbound
      dmaganto.infra/taskdef-type: tenant
  resourceConfig:
    providerConfigName: default
    tenant: {% raw %}{{ .Values.tenant }}{% endraw %}
    name: {% raw %}{{ .Values.serviceName }}{% endraw %}
    deletionPolicy: Delete
    region: {% raw %}{{ .Values.region }}{% endraw %}
  parameters:
    family: {% raw %}{{ .Values.serviceName }}{% endraw %}
    task_id: "{% raw %}{{ .Values.containerDefinitions | toString | sha256sum | trunc 10 }}{% endraw %}"
    containerDefinitions: |- 
      [  
{% raw %}{{ .Values.containerDefinitions | toPrettyJson | indent 8}}{% endraw %}
      ]
{% endhighlight %}

As you can notice in the task definition object we need to do the trick using this expression:

{% highlight yaml %}
{% raw %}{{.Values.containerDefinitions | toString | sha256sum | trunc 10 }}{% endraw %}
{% endhighlight %}

This expression will generate a hash of the container definition object and use the first 10 characters of the hash to generate the task definition name.

Once you apply those object with ArgoCD (to follow the GitOps WoW) or using directly Helm will create the new TaskDefinition with the new hash and remove the old one, and the service using the same definition could obtain the id from `task_id`and deploy it. 

You can find the full code in the [repository](https://github.com/dmaganto/crossplane-ecs)