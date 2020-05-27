---
layout: post
title:  "Deploying a Classic 3-tiers Application using aws-cdk"
date:   2020-05-25 00:00:00 +0800
categories: devops aws
comments: true
published: false
---

### Background

I have been looking for ways to adapt infrastructure-as-code with my team. But the initial complexity is a big deterant. Remember we not only have to output the initial configuration but also to maintain it.

`aws-cdk` is released on 2019-07-11.
It is simpler than writing a CloudFromation template from scratch.
Perhaps it is a good entry point for teams which want to adapt infrastructure-as-code.

### Tryout - classic 3-tier application
With load-balancer tier, stateless application logic tier, and database tier.

Separated subnets: public, private and isolated subnet for better security.

Using environment variable to pass database credentials as it is easier to reuse existing docker image.

### Room for Improvements
1. use separate route tables for each subnet.
2. database security group should allow traffic from the private subnet only



### References
- [aws-cdk is now generally available](https://aws.amazon.com/about-aws/whats-new/2019/07/the-aws-cloud-development-kit-aws-cdk-is-now-generally-available1/)