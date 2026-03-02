---
title: "Fabric8 Kubernetes Client v7.6.0: Kubernetes 1.35, Vert.x 5, and More"
date: "2026-03-02"
category: "Open Source"
description: "A behind-the-scenes look at the Fabric8 Kubernetes Client v7.6.0 release - Kubernetes v1.35 support, Vert.x 5 HTTP client, OkHttp 5, API compatibility reporting, and why you should start contributing."
---

![Fabric8 Kubernetes Client](./fabric8-k8s-client.jpg)

Today I am excited to share what's new in **Fabric8 Kubernetes Client v7.6.0** - a release I shipped as a Senior Software Engineer at Red Hat.

I want to walk you through what changed, why it matters, how to use the library in your own projects, and - most importantly - why you should consider getting involved in making it even better.

---

## What is Fabric8 Kubernetes Client?

For those unfamiliar, the [Fabric8 Kubernetes Client](https://github.com/fabric8io/kubernetes-client) is the most widely-used Java client library for interacting with Kubernetes and OpenShift clusters. It provides a fluent, type-safe API that lets Java developers create, read, update, delete, and watch Kubernetes resources - Pods, Services, Deployments, Custom Resources, and more - without wrestling with raw HTTP calls or hand-rolled JSON serialization.

The project has a long and impactful history in the broader Java community. It is a foundational building block for frameworks like Quarkus, the Java Operator SDK(JOSDK), strimzi-kafka-operator, many Apache projects and countless enterprise applications running on Kubernetes or Red Hat OpenShift. When you write a Kubernetes operator in Java, or integrating your app with Kubernetes or OpenShift or build a controller that manages workloads on a cluster, there is a good chance Fabric8 is doing the heavy lifting underneath.

---

## My First Solo Release

For v7.6.0, I owned the release process end-to-end for the first time - from changelog review and CI validation to Sonatype publishing and Maven Central propagation.

I want to thank my colleague and mentor **[Marc Nuri](https://github.com/manusa)**, one of the core maintainers, whose patient guidance and mentorship made this possible. Thank you, Marc.

---

## What's New in v7.6.0

Let me walk you through the most significant changes in this release.

### Kubernetes v1.35 (Timbernetes) Support

One of the headline additions in v7.6.0 is support for **Kubernetes v1.35**, codenamed *Timbernetes*. This keeps Fabric8 aligned with the latest Kubernetes API surface, ensuring developers working on clusters running v1.35 have access to the newest resource types, API groups, and field-level changes without waiting on the library to catch up.

Staying current with upstream Kubernetes is one of the ongoing commitments of the project. Every new Kubernetes release brings API additions, deprecations, and sometimes breaking changes in the API spec itself - and it is the job of the client library to absorb those changes so application developers don't have to think about them.

### Vert.x 5 HTTP Client

This is arguably the most technically significant addition in v7.6.0. The library now includes a **Vert.x 5 HTTP client implementation** ([#7174](https://github.com/fabric8io/kubernetes-client/issues/7174)).

For context: Fabric8 Kubernetes Client is designed to be HTTP-client-agnostic. It ships with pluggable HTTP backends - OkHttp and Vert.x are the two primary ones - and you can even bring your own. This architecture means teams building reactive or high-throughput applications can select the client that fits their existing stack rather than having a library force a transport choice on them.

The Vert.x 5 implementation comes with meaningful improvements over the Vert.x 4 integration:

- **Improved async handling** that aligns with Vert.x 5's programming model and event loop semantics
- **WebSocket separation** - WebSocket support is now handled through a dedicated API path, matching how Vert.x 5 formally separates HTTP and WebSocket concerns at the framework level
- **Runtime conflict detection** - a check that prevents silent class-loading issues when both Vert.x 4 and Vert.x 5 are present on the classpath, which can happen in complex dependency trees

Additionally, the HTTP client factory priority ordering was corrected in this release: `VertxHttpClientFactory` is now set to priority `-1` (correctly establishing it as the default when Vert.x is on the classpath), and `OkHttpClientFactory` is restored to priority `0`. This resolves a regression where the factory selection order was inverted.

For teams already on Vert.x 5, this upgrade path is now first-class.

### OkHttp Upgraded to 5.3.2

The OkHttp dependency has been upgraded from **4.12.0 to 5.3.2** ([#7422](https://github.com/fabric8io/kubernetes-client/issues/7422)). OkHttp 5 is a significant release from Square - it stabilizes the Kotlin-based API, cleans up the public surface, and eliminates a range of deprecated patterns.

This upgrade required auditing and removing all internal API usages from OkHttp that had accumulated in the integration layer over time, and updating deprecated call patterns throughout the OkHttp backend implementation.

While the upgrade maintains binary compatibility at the API level, subtle behavioral differences can surface in specific configurations. It is worth testing your application against v7.6.0 before rolling it out to production, particularly if you use custom interceptors or advanced OkHttp configuration.

### Byte-Code Level API Compatibility Reporting via Revapi

Starting with v7.6.0, we have integrated **Revapi** ([#7402](https://github.com/fabric8io/kubernetes-client/issues/7402)) to generate byte-code level semantic versioning API compatibility reports as part of the build and release process.

This is a tooling improvement that primarily benefits library consumers even though it is not a user-facing feature. Revapi statically analyzes compiled bytecode across consecutive releases and flags any breaking changes - removed methods, changed signatures, narrowed visibility - that would affect downstream code at the binary level.

---

## Improvements

Beyond the headline additions, v7.6.0 includes several meaningful improvements that improve the day-to-day experience of using and extending the library.

### Javadoc Cross-Linking

We've added **Javadoc cross-linking** ([#1105](https://github.com/fabric8io/kubernetes-client/issues/1105)) across all Fabric8 modules and their external dependencies. This might sound like a minor housekeeping item, but for developers exploring the API through generated Javadocs or IDE documentation popups, having clickable links between modules and their dependencies significantly reduces the friction of understanding how components connect and which types come from where.

### Editable Interface Instead of Reflection for Resource Builders

Custom resources using the visitor pattern now require implementing the `io.fabric8.kubernetes.api.builder.Editable` interface ([#5756](https://github.com/fabric8io/kubernetes-client/issues/5756)), replacing the previous approach that relied on reflection for builder instantiation.

Reflection-based resource building is inherently fragile - it depends on implementation details that aren't part of any public contract - and it carries a runtime performance cost. Moving to an explicit interface makes the dependency clear at compile time, surfaces issues early rather than at runtime, and makes the library significantly more compatible with ahead-of-time compilation environments like GraalVM native image.

**Note for upgraders:** If you have custom resources that use visitors, implementing the `Editable` interface is the primary migration step for v7.6.0. This is the one breaking change in this release that requires a code change on the consumer side.

### additionalConfig Honored for Vert.x HTTP Clients

The `additionalConfig` callback is now properly invoked when building Vert.x HTTP clients ([#7252](https://github.com/fabric8io/kubernetes-client/issues/7252)). This callback allows users to customize the underlying Vert.x HTTP client options at build time - setting connection pool sizes, timeout policies, SSL options, and more. The capability existed in the API contract but wasn't being honored in the Vert.x backend. It now works as documented.

---

## Bug Fixes

Every release carries its share of bug fixes, and v7.6.0 addresses several issues that had been affecting users.

**Cluster TLS server name configuration** ([#5292](https://github.com/fabric8io/kubernetes-client/issues/5292)): When using `cluster()` configuration to build a client, the `tlsServerName` field was being silently ignored, causing TLS hostname validation to fail or fall back to default behavior. This has been corrected.

**Generic type erasure in java-generator** ([#7415](https://github.com/fabric8io/kubernetes-client/issues/7415)): A bug in the java-generator caused incorrect type handling for enum arrays that carried default values. In affected cases, the generated Java code would either fail to compile or produce incorrect runtime behavior when the enum array field was accessed. The fix correctly preserves the enum array type through the generator pipeline.

**Configurable timeout in `createOrReplace()`** ([#7446](https://github.com/fabric8io/kubernetes-client/issues/7446)): The `BaseOperation.createOrReplace()` method now correctly respects the client's configured timeout settings. Previously, this operation used an internal hardcoded value regardless of how the client was configured, which could cause timeouts to trigger unexpectedly in environments with strict latency budgets or when operating against slower API servers.

---

## Dependency Updates

The **SnakeYAML Engine** library has been upgraded from 2.10 to **3.0.1** ([#7374](https://github.com/fabric8io/kubernetes-client/issues/7374)). SnakeYAML Engine is used throughout the client for YAML parsing - loading kubeconfig files, deserializing Kubernetes resource manifests, and handling YAML-formatted API responses. This update brings in upstream bug fixes and improvements and keeps the dependency chain current.

---

## Using Fabric8 in Your Projects

Now that you know what's in v7.6.0, let's talk about how to start using Fabric8 Kubernetes Client in your own Java applications.

### Adding the Dependency

For **Maven**, add the following to your `pom.xml`:

```xml
<dependency>
    <groupId>io.fabric8</groupId>
    <artifactId>kubernetes-client</artifactId>
    <version>7.6.0</version>
</dependency>
```

For **Gradle**:

```groovy
implementation 'io.fabric8:kubernetes-client:7.6.0'
```

If you're working with **OpenShift** and need access to OpenShift-specific resources like Routes, BuildConfigs, or ImageStreams, use the OpenShift client artifact:

```xml
<dependency>
    <groupId>io.fabric8</groupId>
    <artifactId>openshift-client</artifactId>
    <version>7.6.0</version>
</dependency>
```

### Creating a Client

The simplest way to create a client:

```java
try (KubernetesClient client = new KubernetesClientBuilder().build()) {
    // your operations here
}
```

The builder automatically discovers configuration from your environment in priority order: system properties, environment variables, kubeconfig file, and then in-cluster service account credentials. If you are running inside a Kubernetes Pod, it just works with no additional setup.

For OpenShift-specific resources, you can adapt the standard client:

```java
OpenShiftClient osClient = client.adapt(OpenShiftClient.class);
```

### Working with Resources

The client API is fluent and consistent across resource types:

```java
// List Pods in a namespace
PodList pods = client.pods().inNamespace("default").list();

// Fetch a specific Service
Service svc = client.services()
    .inNamespace("my-namespace")
    .withName("my-service")
    .get();

// Delete a resource
client.pods().inNamespace("default").withName("my-pod").delete();
```

Beyond basic CRUD operations, Fabric8 supports **watches** for real-time event streaming, **informers** for efficient in-memory caching of resources, **server-side apply** for declarative management, and fully typed **Custom Resource** support via the java-generator. For a complete walkthrough of all capabilities, the [official usage documentation](https://github.com/fabric8io/kubernetes-client?tab=readme-ov-file#usage) is the best place to start.

---

## Contributing to Fabric8

One of the things I value most about working on Fabric8 Kubernetes Client is that it is a genuinely welcoming project. The community is not gatekept - the maintainers actively want contributors at every level of experience, and they make that clear both in words and in practice.

As the contributing guide puts it: *"Contribution can be very small... We even love to receive a typo fix PR."*

This is not just marketing language. The review culture on this project is constructive and educational. Reviewers will explain *why* something needs to change, not just *what* to change. As someone who grew significantly as an engineer through this project, I can say that the feedback loop you get from contributing here is genuinely valuable - you are working on a library that runs in production at scale, and that forces a level of rigor and thoughtfulness that accelerates growth.

If you are a Java developer working with Kubernetes and want to get more involved in open source, here is how to get started:

1. **Browse open issues** at [github.com/fabric8io/kubernetes-client/issues](https://github.com/fabric8io/kubernetes-client/issues). Look for issues labeled `good-first-issue` or `help-wanted`. Assign yourself to one so the team knows someone is working on it.

2. **Fork and clone** the repository, then build with `mvn clean install`. The build will run the full test suite and confirm your environment is set up correctly.

3. **Write your fix or feature** following the existing code style. Add unit tests for your changes.

4. **Format your code** with `mvn spotless:apply` before pushing - the CI will enforce this.

5. **Open a pull request** linking it to the issue. A message like "Fix #7446" in the PR description automatically links the PR to the issue.

The full contributing guide lives at [CONTRIBUTING.md](https://github.com/fabric8io/kubernetes-client/blob/main/CONTRIBUTING.md) and covers integration testing, license header requirements, and how to run specific subsets of tests.

The kinds of contributions that are valuable go well beyond code: documentation improvements, clarifying confusing Javadoc, adding missing test coverage, reproducing and confirming bugs that others have reported, and helping review open pull requests are all meaningful contributions. Start where you are comfortable, and build from there.

---

## Wrapping Up

Fabric8 Kubernetes Client v7.6.0 is a solid release - Kubernetes v1.35 alignment, a first-class Vert.x 5 HTTP backend, a jump to OkHttp 5, byte-code level API compatibility reporting with Revapi, improved Javadoc cross-linking, and a handful of targeted bug fixes across the board.

For me personally, managing this release end-to-end was a milestone, made possible by the foundation laid by the team and [Marc Nuri](https://github.com/manusa)'s mentorship.

If you are building Java applications that talk to Kubernetes or OpenShift, Fabric8 Kubernetes Client deserves a place in your stack. And if you are looking for an open-source project that is impactful, actively maintained, and genuinely welcoming to new contributors - the door is open.

The [repository is here](https://github.com/fabric8io/kubernetes-client), the issues are waiting, and the community is ready to help you get started.
