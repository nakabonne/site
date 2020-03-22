---
title: Why was gRPC born?
description: >-
  The term gRPC became very famous, and articles such as “Introducing gRPC in
  house” and “How to build gRPC server” can now overflow.
date: '2018-10-04'
tags: ["Microservices"]
---

The term gRPC became very famous, and articles such as “Introducing gRPC in house” and “How to build gRPC server” can now overflow.

{{< figure src="https://cdn-images-1.medium.com/max/800/1*e2RMFaYNVAsjcjFWHmWvYQ.png" width="100%" height="auto">}}

By doing this, the hurdle for introducing gRPC has come down, and it has become possible to create a production ready gRPC server relatively easily.

But what was the origin of gRPC originated in the first place?

### Background

In the [gRPC Blog](https://grpc.io/blog/principles), it is described as follows.

> _Google has been using a single general-purpose RPC infrastructure called Stubby to connect the large number of microservices running within and across our data centers for over a decade. Our internal systems have long embraced the microservice architecture gaining popularity today. Having a uniform, cross-platform RPC infrastructure has allowed for the rollout of fleet-wide improvements in efficiency, security, reliability and behavioral analysis critical to supporting the incredible growth seen in that period._

> _Stubby has many great features — however, it’s not based on any standard and is too tightly coupled to our internal infrastructure to be considered suitable for public release. With the advent of SPDY, HTTP/2, and QUIC, many of these same features have appeared in public standards, together with other features that Stubby does not provide. It became clear that it was time to rework Stubby to take advantage of this standardization, and to extend its applicability to mobile, IoT, and Cloud use-cases._

Also on the [Google Cloud Platform Blog](https://cloudplatform.googleblog.com/2016/08/gRPC-a-true-Internet-scale-RPC-framework-is-now-1-and-ready-for-production-deployments.html), it says:

> _Building highly scalable, loosely coupled systems has always been tough. With the proliferation of mobile and IoT devices, burgeoning data volumes and increasing customer expectations, it’s critical to be able to develop and run systems efficiently and reliably at internet scale._

> _In these kinds of environments, developers often work with multiple languages, frameworks, technologies, as well as multiple first- and third-party services. This makes it hard to to define and enforce service contracts and to have consistency across cross-cutting features such as authentication and authorization, health checking, load balancing, logging and monitoring and tracing, all the while maintaining efficiency of teams and underlying resources. It becomes especially challenging in today’s cloud-native world, where new services need to be added very quickly and the expectation from each service is to be agile, elastic, resilient, highly available and composable._

> _For the past 15 years, Google has solved these problems internally with Stubby, an RPC framework that consists of a core RPC layer that can handle internet-scale of tens of billions of requests per second (yes, billions!). Now, this technology is available for anyone as part of the open-source project called_ [_gRPC_](http://www.grpc.io/)_. It’s intended to provide the same scalability, performance and functionality that we enjoy at Google to the community at large._

To summarize the above

*   It is difficult to create a scalable and loosely coupled system, but in modern times where mobile etc. are popular, we have to create it
*   Google used a cross-platform RPC framework to handle it
*   But it depended heavily on Google’s internal infrastructure
*   Since HTTP / 2 etc appeared, it became possible to create more than the equivalent with the standard without relying on Google infrastructure
*   Let’s make it public based on standards standard!

### Reference

If you want to deepen your understanding of RPC, you’d better start reading from the link below.

*   [https://grpc.io/blog](https://grpc.io/blog/)
*   [https://github.com/grpc/grpc/blob/master/CONCEPTS.md](https://github.com/grpc/grpc/blob/master/CONCEPTS.md)
