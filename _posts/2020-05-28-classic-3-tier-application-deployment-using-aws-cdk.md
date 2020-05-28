---
layout: post
title:  "Deploying a Classic 3-tiers Application using aws-cdk"
date:   2020-05-28 00:00:00 +0800
categories: devops aws
comments: true
published: true
---

### Background

I have been looking for ways to adapt infrastructure-as-code with my team. But the initial complexity is a big deterant. Remember we not only have to output the initial configuration, but to maintain it as well.

`aws-cdk` is released on [2019-07-11](https://aws.amazon.com/about-aws/whats-new/2019/07/the-aws-cloud-development-kit-aws-cdk-is-now-generally-available1/).
It is simpler than writing a CloudFromation template from scratch.
Perhaps it is a good entry point for teams that want to adapt infrastructure-as-code.


### A Classic 3-Tiers Application

With load-balancer tier, stateless application logic tier, and database tier.

| Tier | Componenet | AWS Service | Subnet |
|------|------------|-------------|--------|
| 1 | Load-balancer | AWS ELB | Public |
| 2 | Application Logic | AWS ECS Fargate | Private |
| 3 | Database | AWS RDS Aurora | Isolated |


Following the [security practice of separating subnets for different tiers](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Subnets.html#vpc-subnet-basics), the application will be deployed into:  
1. a public subnet(with two-way Internet access), 
2. a private subnet(with out-going Internet access only), and 
3. a isolated subnet(no Internet access either way).


We are also using environment variable to pass database credentials as it is easier to reuse existing docker image.

Here is the `aws-cdk` stack that I managed to get working:  
<script src="https://gist.github.com/billykong/a778613eb0ecf1ab59ff82b76bbf985b.js"></script>

### Deploy
If you want to deploy it and poke around, you can checkout the [GitHub repository here](https://github.com/billykong/aws-cdk-fargate-rds-stack). The deployment instruction is written in `README.md`.  
  
Note that we should install the **same version** of `aws-cdk` and other `@aws-cdk/*` dependencies. It seems even minor version difference may be incompatible. I used `v1.38.0`.


#### Some Rooms for Improvements
1. Use separate route tables for each subnet.
2. Database security group should allow traffic from the private subnet only.
3. Calling AWS Secret Manager API from application code for database credential is probably more secure, but it will require some custom code. If you expect to reuse the same Docker image in, say, Kubernetes, it may cause problems.
4. I couldn't quite get the `DatabaseCluster` construct to work. So I used the CloudFormation verions `CfnDBCluster`. If you managed to use `DatabaseCluster`, please feel free to leave a comment.


### References
- [AWS CDK API Reference](https://docs.aws.amazon.com/cdk/api/latest/docs/aws-construct-library.html)
- [How to create an Aurora serverless RDS instance on AWS with CDK](https://dev.to/cjjenkinson/how-to-create-an-aurora-serverless-rds-instance-on-aws-with-cdk-5bb0)
- [bind-almir/cdk-aurora-serverless](https://github.com/bind-almir/cdk-aurora-serverless/blob/master/lib/cdk-aurora-serverless-stack.ts)
- [AWS VPC Subnet Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Subnets.html#vpc-subnet-basics)
- [Source Code of aws-cdk-fargate-rds-stack](https://github.com/billykong/aws-cdk-fargate-rds-stack)
- [Source Code for express-database-checker](https://github.com/billykong/express-database-checker)
- [aws-cdk is now generally available](https://aws.amazon.com/about-aws/whats-new/2019/07/the-aws-cloud-development-kit-aws-cdk-is-now-generally-available1/)