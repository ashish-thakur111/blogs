---
title: "Building Production-Grade Kubernetes Operators in Java — My Talk at FOSS United Pune"
date: "2026-06-22"
category: "Open Source"
description: "A walkthrough of my talk at FOSS United Pune on building production-grade Kubernetes operators using the Java Operator SDK (JOSDK) and Fabric8 Kubernetes Client — covering the operator pattern, the Java stack, dependent resources, and a live demo."
---

![FOSS United Pune Conference](./cover.jpg)

On June 20th, I took the stage at the **FOSS United Pune conference** to talk about building production-grade Kubernetes operators in Java. The room was packed with developers from across the open source community, and the energy was exactly what you would hope for at a FOSS event — curious, technical, and ready to dig into the details.

You can view the full presentation here: [Building Production-Grade Kubernetes Operators with Java](https://presentations.ashthakur.in/presentations/2026-foss-united-pune-josdk/)

This post walks through the key ideas from the talk and adds some context that didn't fit into the 35-minute slot.

---

## The Problem: Day 2 Operations

I opened the talk with a question most Kubernetes users have lived through: what happens after `kubectl apply`?

Day 0 is planning — architecture, tooling, capacity. Day 1 is deployment — write the YAML, apply it, watch it come up. These are the parts everyone enjoys.

Day 2 is where it gets real. Zero-downtime upgrades, scaling decisions, backup and restore, failover, configuration drift. These operations require domain-specific knowledge about your application — knowledge that Kubernetes itself does not have. A database cluster has a specific bootstrap sequence. A message broker has particular rules for when replicas join or leave. Kubernetes cannot know these things out of the box.

The operator pattern is how you encode that operational knowledge into software that runs inside the cluster, watching and reacting continuously. Humans cannot be on-call 24/7 for every resource — operators can.

---

## What is a Kubernetes Operator?

An operator is a software extension that manages resources on behalf of Kubernetes. It follows a three-step cycle:

1. **Declare** — the user specifies what they want via a Custom Resource
2. **Watch** — a controller detects the desired state change
3. **Reconcile** — the controller continuously drives current state toward desired state

This is fundamentally declarative. The user says *what* they want, the operator figures out *how*. Something breaks? The operator notices and fixes it. Configuration drifts? The operator corrects it. The loop never stops.

An important distinction I covered in the talk: every operator is a controller, but not every controller is an operator. A controller follows the observe-compare-act loop on built-in Kubernetes resources — no CRDs needed. An operator extends this with Custom Resource Definitions to manage an application's full lifecycle with domain knowledge.

---

## Why Java?

The dominant operator framework has been Go's Operator SDK, and for good reason — most Kubernetes ecosystem tooling is written in Go. But that is not the reality for a lot of teams.

Java has massive enterprise adoption. If your organization runs microservices on Spring Boot or Quarkus, your developers already know Java. Asking them to context-switch to Go for infrastructure tooling is a real cost — in learning curve, in tooling, in CI/CD pipelines.

The Java stack for building operators looks like this:

```
┌─────────────────────────────────────┐
│         Your Operator               │
│   Business logic & domain knowledge │
├─────────────────────────────────────┤
│     Java Operator SDK (JOSDK)       │
│  Reconciliation, events, retries,   │
│  dependent resources                │
├─────────────────────────────────────┤
│   Fabric8 Kubernetes Client         │
│  Fluent API, CRUD, watch, CRDs,     │
│  OpenShift support                  │
├─────────────────────────────────────┤
│       Kubernetes API                │
│       REST API & etcd               │
└─────────────────────────────────────┘
```

[Fabric8 Kubernetes Client](https://github.com/fabric8io/kubernetes-client) is the foundation — the industry-standard Java client for Kubernetes and OpenShift with over 1 million monthly downloads, 3.7k stars, and 11 years of history. It powers Quarkus, Spring Cloud Kubernetes, Apache Flink, and JOSDK itself.

[Java Operator SDK (JOSDK)](https://github.com/operator-framework/java-operator-sdk) sits on top of Fabric8 and adds everything you need for a production operator — the reconciliation loop, event handling, retry logic, dependent resource management. It is a CNCF project, works with both Quarkus and Spring Boot, and is used in production by Keycloak, Apache Flink, Apache Spark, Strimzi, Debezium, and Glasskube.

---

## The Real-World Comparison

One of the slides that got the most reaction was the side-by-side comparison of deploying a microservice with and without an operator.

**Without an operator**, you manage everything yourself:

- Write a Deployment YAML
- Write a Service YAML
- Write a ConfigMap YAML
- Create a Secret with database credentials
- Deploy a PostgreSQL StatefulSet
- Configure Ingress and HPA

That is 6+ YAML files, all manual, all error-prone. Forget one label selector and things break silently.

**With an operator**, you declare your intent:

```yaml
apiVersion: example.io/v1
kind: MicroService
metadata:
  name: petclinic
spec:
  image: spring-petclinic:3.5.6
  replicas: 2
  database: { databaseName: petclinicdb }
  exposed: true
```

One resource, about 8 lines of spec. The operator auto-creates and manages all of the underlying Deployments, Services, ConfigMaps, Secrets, and Ingress resources behind the scenes. The important thing is that those resources still exist — the operator manages them, they do not disappear. The user just does not have to deal with them directly anymore.

---

## Building It: CRD, Reconciler, and Dependent Resources

The core of the talk was showing how JOSDK makes building this operator straightforward.

### Custom Resource Definition

Your CRD is a Java class — three annotations, extend `CustomResource`, implement `Namespaced`. The Fabric8 CRD Generator produces the YAML schema automatically:

```java
@Group("example.io") @Version("v1") @ShortNames("ms")
public class MicroService extends
    CustomResource<MicroServiceSpec, MicroServiceStatus>
    implements Namespaced { }
```

### The Reconciler

The reconciler is where JOSDK shines. The `@Workflow` annotation declares the dependency graph — which dependent resources exist and in what order they should be created:

```java
@Workflow(dependents = {
    @Dependent(type = ConfigMapDependentResource.class),
    @Dependent(type = DeploymentDependentResource.class,
               dependsOn = "ConfigMap"),
    @Dependent(type = ServiceDependentResource.class,
               dependsOn = "Deployment"),
    @Dependent(type = IngressDependentResource.class,
               reconcilePrecondition = ExposedCondition.class)
})
@ControllerConfiguration
public class MicroServiceReconciler
    implements Reconciler<MicroService> {

    public UpdateControl<MicroService> reconcile(
            MicroService ms, Context<MicroService> ctx) {
        int ready = ctx.getSecondaryResource(Deployment.class)
            .map(d -> d.getStatus().getReadyReplicas())
            .orElse(0);
        return UpdateControl.patchStatus(
            buildStatusPatch(ms, ready));
    }
}
```

The `reconcile()` method itself is tiny — it reads the Deployment's ready replicas from the cache and patches the status. All the heavy lifting is handled by dependent resources.

### Dependent Resources

Each dependent resource is a class that knows how to build its desired state from the primary `MicroService` spec. You implement one method — `desired()` — and the framework handles the rest:

```java
public class DeploymentDependentResource
    extends CRUDKubernetesDependentResource
             <Deployment, MicroService> {

    protected Deployment desired(MicroService ms,
                        Context<MicroService> ctx) {
        return new DeploymentBuilder()
            .withNewMetadata()
                .withName(ms.getMetadata().getName())
                .withNamespace(ms.getMetadata().getNamespace())
            .endMetadata()
            .withNewSpec()
                .withReplicas(ms.getSpec().getReplicas())
                .withNewSelector()
                  .withMatchLabels(Map.of("app",
                    ms.getMetadata().getName()))
                .endSelector()
            .endSpec().build();
    }
}
```

JOSDK compares the desired state with what actually exists in the cluster — creates if missing, updates if different, leaves alone if matching. Owner references are set automatically, so deleting the custom resource garbage-collects everything.

---

## Patterns That Matter in Production

Four patterns I emphasized during the talk:

- **Idempotency** — your reconcile method will be called many times. It must produce the same result regardless of how many times it runs.
- **Always reconcile all resources** — do not just react to the event that triggered you. Check the actual state of everything you manage.
- **Stateless logic** — use the Kubernetes Status sub-resource to track progress, not in-memory state. If your operator restarts, it should pick up where it left off.
- **Leverage caching** — use informers and event sources to avoid hammering the API server.

For error handling, JOSDK provides `updateErrorStatus` to write errors to the status sub-resource so users can see what went wrong. And cleanup is typically one line — `DeleteControl.defaultDelete()` — because owner references handle garbage collection automatically.

---

## The Demo

The highlight was the live demo using the [microservice-operator](https://github.com/ash-thakur-rh/microservice-operator). Applied a single `MicroService` CR to a live cluster and watched the operator provision the full stack in real time — Deployment, Service, ConfigMap, and when the database spec was added, a PostgreSQL StatefulSet with auto-generated credentials.

The moment that landed best: deleting the custom resource and watching Kubernetes garbage-collect every dependent resource automatically. No manual cleanup, no orphaned resources. That is the operator pattern working as designed.

---

## FOSS United Pune

[FOSS United](https://fossunited.org/) continues to be one of the best communities for technical talks in India. The Pune chapter brings together people from genuinely diverse backgrounds — not just web and AI, but infrastructure, systems programming, and open source tooling. The questions after the talk were sharp — leader election, rate limiting, server-side apply support in JOSDK — the kind of questions that tell you people are thinking about actually using this.

If you are in Pune and working on something worth sharing, I would encourage you to propose a talk. The community is welcoming and the feedback is real.

---

## Resources

- **Slides**: [Building Production-Grade Kubernetes Operators with Java](https://presentations.ashthakur.in/presentations/2026-foss-united-pune-josdk/)
- **JOSDK**: [javaoperatorsdk.io](https://javaoperatorsdk.io) | [GitHub](https://github.com/operator-framework/java-operator-sdk)
- **Fabric8 Kubernetes Client**: [GitHub](https://github.com/fabric8io/kubernetes-client)
- **Demo project**: [microservice-operator](https://github.com/ash-thakur-rh/microservice-operator)

If you have follow-up questions, reach out on [Twitter/X](https://twitter.com/ashish___thakur) or [LinkedIn](https://linkedin.com/in/ashish-thakur111).
