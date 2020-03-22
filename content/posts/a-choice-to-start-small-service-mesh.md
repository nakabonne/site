---
title: "A choice to start small Service Mesh"
description: "Introduces the means to start small Service Mesh referring to sxds"
date: 2018-11-24
tags: ["Golang", "Microservices"]
comments: false
---

Now, the architecture called Service Mesh is attracting attention in the Microservice industry.  
There are various products for realization, but I made a small control-plane called sxds.  
In this article, I will introduce the means to start small Service Mesh referring to sxds.

[**nakabonne/sxds**  
_simple-xds that provides configuration and route table to data-plane for service discovery - nakabonne/sxds_github.com](https://github.com/nakabonne/sxds "https://github.com/nakabonne/sxds")[](https://github.com/nakabonne/sxds)

### Target reader

*   Those considering introducing Service Mesh
*   Those who want to deepen their understanding of Service Mesh
*   Those being Interested in Microservices

### What is Service Mesh

Briefly, it’s one of the architectures to solve various problems caused by Microservices.  
Since the definition of Service Mesh is introduced in various articles, it’s omitted in the present article.

*   [What’s a service mesh?](https://www.redhat.com/ko/topics/microservices/what-is-a-service-mesh)
*   [Istio / What is Istio?](https://istio.io/docs/concepts/what-is-istio/#what-is-a-service-mesh)
*   [What Is a Service Mesh? — NGINX](https://www.nginx.com/blog/what-is-a-service-mesh/)
*   [Lyft’s Envoy: From Monolith to Service Mesh — Matt Klein | Microservices.com](https://www.microservices.com/talks/lyfts-envoy-monolith-service-mesh-matt-klein/)
*   [What’s a service mesh? And why do I need one? — blog.buoyant](https://blog.buoyant.io/2017/04/25/whats-a-service-mesh-and-why-do-i-need-one/)

### Means of realizing Service Mesh

The tools that realize control-plane and data-plane are as follows.  
I will explain only what is considered representative at the time of writing here.

#### control-plane

*   [Istio](https://istio.io/)
*   [Synapse](https://inch-ci.org/github/airbnb/synapse)
*   [Nelson](https://getnelson.io/)
*   [AWS App Mesh](https://aws.amazon.com/jp/app-mesh/) (Available only as public preview at the time of writing)

Istio is famous as the control-plane at the time of writing.  
Istio encloses a data-plane. Therefore, Service Mesh can be realized by this one.  
It’s a very good tool, but of course it’s not a silver bullet.  
Istio exerts its effect by collecting cluster information etc from the orchestration tool.  
In other words, it can not be effective in environments not using orchestration tools.  
Also, Istio offers many functions, but if you use only a few functions you can not get enough benefit.

#### data-plane

*   [Envoy](https://www.envoyproxy.io/)
*   [Linkerd](https://linkerd.io/)
*   [NGINX](https://www.nginx.com/)
*   [HAProxy](https://www.haproxy.com/)
*   [Traefik](https://traefik.io/)

I think that Envoy which is also adopted as Istio’s default data-plane can be considered as a representative.  
Envoy is an L7 proxy. The feature is built on the premise of mediation of inbound and outbound traffic, so distribution tracing and dynamic service discovery can be realized at the proxy level.

### Control-plane self-making

As I mentioned, there are lots of great tools. However, the only function I initially wanted in my project was to dynamically update the data-plane configuration for service discovery.  
Also, since all services were running on a container management service called AWS’s ECS, we have not used orchestration tools such as k8s.  
So I adopted Envoy for data-plane and created control-plane [sxds](https://github.com/nakabonne/sxds) which controls it.  
I will explain the method from the next section in detail.

#### Interface

The interface for communicating with the Envoy is published as [data-plane-api](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api), and we will create an API that satisfies this.  
While Envoy is the only proxy conforming to this API definition at the time of writing, the author Matt Klein is speaking on a [blog](https://blog.envoyproxy.io/the-universal-data-plane-api-d15cec7a) to make data-plane-api generic.  
Therefore, if this API definition becomes generic in the future, the control-plane created by you will be generic as well.  
Also, the API that satisfies data-plane-api is called xds.

#### Implementation by Go

Envoy defines a proprietary protocol, [xds protocol](https://github.com/envoyproxy/data-plane-api/blob/master/XDS_PROTOCOL.md).  
According to this, xds must be implemented in gRPC bidi stream or long polling REST.  
It’s necessary to implement an implementation that complies with xds protocol, but only Go and Java have official libraries containing this implementation.

*   [envoyproxy/go-control-plane](https://github.com/envoyproxy/go-control-plane)
*   [envoyproxy/java-control-plane](https://github.com/envoyproxy/java-control-plane)

In sxds I selected Go, and imported go-control-plane and implemented xds server.

*   [xds/server.go · nakabonne/sxds · GitHub](https://github.com/nakabonne/sxds/blob/master/server/xds/server.go)

Yes, this is the end of xds implementation. However, since go-control-plane needs to cache data-plane settings in memory, we also implemented a REST server for caching as follows.

*   [cacher/server.go · nakabonne/sxds · GitHub](https://github.com/nakabonne/sxds/blob/master/server/cacher/server.go)
*   [cacher/cacher.go · nakabonne/sxds · GitHub](https://github.com/nakabonne/sxds/blob/master/server/cacher/cacher.go)

The whole architecture looks like this.

![](https://cdn-images-1.medium.com/max/800/1*eFBElcoENe93GfH8K4ERLA.png)

#### Static resources management

You need to manage the static resources needed to respond to Envoy.  
Since sxds provides four services, [LDS](https://www.envoyproxy.io/docs/envoy/latest/configuration/listeners/lds)/[CDS](https://www.envoyproxy.io/docs/envoy/latest/configuration/cluster_manager/cds)/[RDS](https://www.envoyproxy.io/docs/envoy/latest/configuration/http_conn_man/rds)/[EDS](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/eds.proto#envoy-api-file-envoy-api-v2-eds-proto), you need to define the following resources necessary for these services.

*   [listeners](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/lds.proto#envoy-api-msg-listener)
*   [clusters](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/cds.proto#cluster)
*   [routes](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/rds.proto#routeconfiguration)
*   [endpoints](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/eds.proto#envoy-api-msg-clusterloadassignment)

You also need to specify the version of the resources.  
In order to prevent unnecessary packet exchange, the rule that no response is returned when there is no change in version is defined in xds protocol.  
You need to define them with json and send to the server for caching.

*   [resources/sidecar.json · nakabonne/sxds · GitHub](https://github.com/nakabonne/sxds/blob/master/sample/resource/sidecar.json)

#### Envoy settings

As a static resources, define information on the cluster that runs sxds.  
As a dynamic resources, specify the cluster defined earlier.

*   [envoy/envoy.yml · nakabonne/sxds · GitHub](https://github.com/nakabonne/sxds/blob/master/sample/envoy/envoy.yml)

#### Execution

**Execution of Envoy**

sxds caches resources for each node type to reduce memory consumption.  
Therefore, it’s necessary to set the node type at startup, and set this by setting the node ID naming rule to {node\_type}-something.  
Also, since its own cluster information is required for discovery request to control-plane, please specify the cluster name as well.

$ envoy --service-node sidecar-app1 --service-cluster app1

**Execution of sxds**

$ sxds  
2018-12-01T10:00:00.518+0900    INFO    xds/server.go:56    xDS server is listening {"conf": {"Port":8081}}  
2018-12-01T10:00:00.518+0900    INFO    cacher/server.go:58 cacher server is listening  {"conf": {"Port":8082}}

**Send resources**

Send the json file created above to the cacher server.

$ curl -XPUT http://{IP\_ADDRESS}:8082/resources/sidecar -d @sidecar.json

Then you should be able to confirm that the setting is reflected in Envoy.

### Conclusion

I introduced sxds, but of course this is not perfect.  
It becomes hard to handle when xds is made redundant, and a mechanism to persist resources json is also necessary.

**What I want to tell you is that using the official library of envoyproxy makes it possible to create a control-plane at low cost, and that means of self-making only the necessary functions is also a sufficiently effective means.  
When migrating an existing service to Microservice, it’s not necessarily a modern environment.  
In such a situation, having a means to be able to easily self-make according to the environment is a great weapon.**

I would be pleased if you have the choice to start small Service Mesh in your head now.
