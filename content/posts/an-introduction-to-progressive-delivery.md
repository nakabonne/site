---
title: "An Introduction to Progressive Delivery"
description: "Learn about Progressive Delivery, the term which describes the way to improve delivery times for new product features."
date: 2020-06-13
tags: ["devops"]
draft: false
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
In a nutshell, it automates rollouts and rollbacks based on metrics and more.
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

TODO: こんな感じの図を作る
{{< figure src="https://images.ctfassets.net/zw48pl1isxmc/6Blb2fE7j83mIPpT8GTmRL/e1c331719b92643147a49e9b577e7e9f/progressive-delivery.png" width="100%" height="auto">}}

TODO: これを自分の言葉で

Progressive delivery provides the ability to gradually release your feature to progressively larger audiences starting with your internal teams, then beta testers, and followed by percentages of your production traffic.

If you detect errors, performance spikes, or user backlash, just roll back - all without redeploying code. Since you aren't going from 0% to 100% rollout, if you detect an issue at 1% of your traffic, you've only potentially impacted 1% of revenue or 1% of customer happiness.

Progressive delivery makes your release process more resilient. Additionally, the use of experimentation (or A/B testing) as a part of the progressive delivery process enables you to not only reduce risk but also iterate and determine the best possible experience for your end-users.

## Benefits
### Easier to test
The main benefit for lots of people would be this.
By automating rollbacks, developers can run tests in production with less risk.
It enables to easily find bugs that can't be found without running it in a production environment.
As a result, Progressive Delivery allows development teams to release features quickly with minimal risk.

### Social Autonomy
TODO: これ参考に https://redmonk.com/jgovernor/2018/08/06/towards-progressive-delivery/

## How to analyze.
Let's get a bit more specific. 上記を実現するために分析というフェーズが中核を担っていることが分かると思います。これからその説明をしていきます。

First up, let me briefly describe the deployment strategies. The industry has agreed upon high-level definitions of various kinds of deployment strategies.
While different people interpret it slightly differently, it would be almost like this:

- `Canary`: The new version gradually replaces the old version with verification
- `Blue/Green`: Switching the live traffic to the new version once it's fully tested
- `Rolling Update`: Slowly replaces the old version with the new version
- `Recreate`: Deletes the old version before bring up the new version

Although Progressive Delivery doesn't depend on specific deployment strategies, the canonical use case for Progressive Delivery in my opinion is Canary analysis;
thus let's take a look how to analyze Canary so as to dig deeper into Progressive Delivery.

### Static Rules vs Dynamic Data

Notionの内容書くhttps://www.notion.so/cyberagent/Harness-Continuous-Verification-daf93d012d6942fd86aefa69737bbccf

TODO: 以下の具体的な実現方法は別記事でいいかも

### [Flagger](https://flagger.app/)

### [Argo Rollouts](https://argoproj.github.io/argo-rollouts/)

### [Harness Continuous Verification](https://docs.harness.io/category/gurgsl2gqt-continuous-verification)


## Wrapping up

こういった課題がある
- Kubernetes 上で実現したい場合、Istio、Linkerd、Contour、Ambassador などのそこそこリッチなサービスメッシュやイングレスゲートウェイとセットで導入することになる
- 素の Envoy を横に並べてフロントプロキシとして利用しているような事例がよくある
- Progressive Delivery を始めようとすると、突然サービスメッシュがIngress Gatewayのソリューションが必要になる。選定含めおおごとになりがち
