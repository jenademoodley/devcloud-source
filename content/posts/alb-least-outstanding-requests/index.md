+++
date = 2020-07-24
draft = false
title = 'Understanding Least Outstanding Requests ALB Load Balancing Algorithm'
url = 'understanding-alb-least-outstanding-request'
description = 'Understanding Least Outstanding Requests ALB Load Balancing Algorithm'
weight = 10
showAuthor = false
authors = ["jenademoodley"]
showHero = false
categories = ['AWS']
tags = ['ELB', 'ALB']
+++

If you are using an Application Load Balancer, you may have come across the "Least Outstanding Requests" (LOR) load balancing algorithm. A load balancing algorithm dictates how requests are sent to targets. With the default round robin configuration, all requests will be split among the targets equally, regardless of the state of the target. This can cause issues if the requests are not equal (i.e. spiky workloads) as this may result in overloading of requests to one of the targets, and under-utilization of other targets. However, the least outstanding requests algorithm aims to solve this by sending requests to targets with the lowest number of outstanding (or existing) connections. Straight from [AWS' blog post](https://aws.amazon.com/about-aws/whats-new/2019/11/application-load-balancer-now-supports-least-outstanding-requests-algorithm-for-load-balancing-requests/):

> With this algorithm, as the new request comes in, the load balancer will send it to the target with least number of outstanding requests. Targets processing long-standing requests or having lower processing capabilities are not burdened with more requests and the load is evenly spread across targets. This also helps the new targets to effectively take load off of overloaded targets.

## What is a "Least Outstanding Request"

Simply put, outstanding requests refers to the number of requests that each target is currently processing. In order for this algorithm to work effectively, the ALB needs to keep track of the number of existing requests per target, and update this list whenever a request is completed, or a new request is made. The ALB will route an incoming request to the target with the lowest number of these requests. This also means that a target could be at the bottom of this list for one request, and at the top of the list for the next request, hence the process has a very fluid state.

## Some of my targets receive more requests than others

Based on the nature of this algorithm, this can be expected. The ALB will route requests to targets with the least number of outstanding requests at that point in time. It will not consider how many requests have already been sent to that target as these requests would have already been fulfilled.

Consider a scenario in which there are no requests served to any of your targets, or if there is an equal number of long lasting connections on all targets. If a new request comes in, that request can be served by any of the targets, and an arbitrary target (let's call this target A) is chosen. If the request is completed, and a new request comes in, the scenario is exactly the same, hence target A can be chosen again. It is not uncommon for there to be an imbalance of requests in such scenarios. This should also not cause any issues with resource utilization as each target would handle it's fair share of requests at any given time.

From an analysis view point, we tend to look at the number of requests processed by each target over the course of a period of time (such as over an hour, day, week, month, etc), which often causes confusion. When dealing with the least outstanding requests algorithm, we need to take into account that it does not consider the number of requests over a period of time, but the number of requests that each target is processing at that exact moment in time when the request is received.

## When should I use the least outstanding requests algorithm

The least outstanding requests algorithm should be used in either or both of the below scenarios:

- Not all of the requests are equal (i.e. requests vary in complexity).
- Each of the targets very in processing capability (for example, each EC2 instance used as a target is a different instance type).

{{< figure
    src="img/lor.webp"
    alt="Least Outstanding Requests Diagram"
    caption="[Image by cloudonaut.io](https://cloudonaut.io/reinvent-recap-2019-aws-announcements/)"
    >}}

If all of your targets are the same, and you expect a similar form and complexity of all requests, you may not see any benefit to using the least outstanding requests algorithm, and you should consider switching back to the round robin load balancing algorithm. 