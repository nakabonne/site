---
title: "An Introduction to Progressive Delivery"
description: "Learn about Progressive Delivery, the term which describes the way to improve deliveries for new product features."
date: 2020-06-14
tags: ["devops"]
draft: false
images: ["/img/BlastRadius.jpg"]
---

As reported by DORA's [The Accelerate State of DevOps Report](https://services.google.com/fh/files/misc/state-of-devops-2019.pdf), elite performers deploy code to production a bunch of times a day,

The more you deploy a lot, the more likely you are to win the market,
while it's clear that most incidents are caused by code changes.
Frequent deployments come with significant risk.

James Governor at RedMonk who invented the term Progressive Delivery says:

>The elite performers are shipping a lot more code and with higher quality.
> -- "[An Introduction to Progressive Delivery](https://www.infoq.com/presentations/progressive-delivery/)", James Governor at Qcon 2019

That is, what we want to end up with is a way to deploy a lot without breaking anything.
I'd say Progressive Delivery is the closest solution to what's being looking for, that's why I will now walk you through what it is.

## What is Progressive Delivery?
In a nutshell, it is a set of gradual deployments, analysis and automated rollbacks (as well as rollouts).
This is obviously not enough, so let me take it a bit further.

This is an excerpt from [Progressive Delivery in Kubernetes: Blue-Green and Canary Deployments](https://www.cloudbees.com/blog/progressive-delivery-kubernetes-blue-green-and-canary-deployments) written by Carlos Sanchez:

>Progressive delivery is the next step after continuous delivery, where new versions are deployed to a subset of users and are evaluated in terms of correctness and performance before rolling them to the totality of the users and rolled back if not matching some key metrics.

I guess that explains it pretty well. That's not only an overview, but also shows that it is Continuous Delivery++.

Here's another short and coherent explanation:

>Progressive Delivery is Continuous Delivery with fine-granted control over the blast radius.
>  
>Building blocks:
>- User segmentaition
>- Traffic management
>- Observability
>- Automation
>
> -- "[Flagger: Istio Progressive Delivery Operator](https://www.slideshare.net/weaveworks/flagger-istio-progressive-delivery-operator)", Stefan Prodan

I really like the expression "blast radius", which is sometimes used to describe the range of failures.

{{< figure src="/img/BlastRadius.jpg" width="100%" height="auto">}}

## Benefits
### Testing in production
The main benefit for lots of people would be this.
Automated rollbacks makes it possible to perform tests in production with less risk.
That lowers tha bar for finding bugs that have to be executed in a production environment to be found.

We often cannot completely simulate live, real-world software in a test and dev environment. 
Because of that, it's been mentioned in [a variety of places](https://medium.com/@copyconstruct/testing-in-production-the-safe-way-18ca102d0ef1) that production testing has become a necessary part of continuous delivery.

Accordingly, you can see Progressive Delivery allows us to release features quickly with minimal risk.

### Beyond alpha thinking
We can now go forward and backward in our releases using automated analysis processes.
That makes it much easier to release different versions for different regions and user segments.

That means Progressive delivery has the potential to take us beyond simple alpha, beta and GA thinking.


## Wrapping up
While I didn't describe the specific process, we are currently facing a few challenges. The main challenges I consider are:

- In many instances, it requires Service Mesh which is crucial for traffic management
- What you can analyze the dynamic deployment by defining static rules is limited

The continuous delivery area is still a kind of chaos.
I believe it's big step in the right direction to verbalize what many people have in mind.
I hope to write about the elements that make it up such as analyses in the near future.

### Links:
- [Towards Progressive Delivery - RedMonk](https://redmonk.com/jgovernor/2018/08/06/towards-progressive-delivery/)
- [Progressive Delivery - InfoQ](https://www.infoq.com/presentations/progressive-delivery/)
- [Flagger: Istio Progressive Delivery Operator - SlideShare](https://www.slideshare.net/weaveworks/flagger-istio-progressive-delivery-operator)
- [Progressive Delivery in Kubernetes: Blue-Green and Canary Deployments](https://www.cloudbees.com/blog/progressive-delivery-kubernetes-blue-green-and-canary-deployments)
- [What is Progressive Delivery? - Split Glossary](https://www.split.io/glossary/progressive-delivery/)
- [Progressive Delivery - Optimizely](https://www.optimizely.com/optimization-glossary/progressive-delivery/)
