# Shared Internet Ingress on AWS
## Introduction
First up, let me say that I am highly opinionated on this topic. I have had dozens (if not hundreds) of conversations with customers about this over the years.

My strong opinion is that customers should not have a shared ingress point. The reasons for wanting them are quite common:
1. "This is how we did it on-premises" - a totally valid claim but in the on-premises world there were generally very few (one or two) network links into the customer network from the internet. It made sense to have shared ingress for everyone b ecause there was (after all) a shared network link where the traffic was delivered. In AWS this is no longer true - from the perspective of customer VPCs the Internet is "just there" and there is more bandwidth available than is practical for a customer to consume.
2. "I want to inspect everything" which tends to come from security and compliance teams. This is a valid requirement but feeding all traffic through a shared firewall (again, as was done on-premises) or firewall cluster means that the firewall(s) become a big central point of failure. If one application is having a good day (i.e. higher levels of traffic than usual) then every other application relying on those firewalls may be having a bad day. If one application is having a bad day (i.e. under DDoS) then everyone is having a bad day.
3. "Distributed ingress scares me" because the level of cloud maturity and the use of automation witin the customer's team is not at a level where they can effectively administer, monitor and control multiple ingress points. This is a completely valid point and it is why this repo exists.

 The other reason I am not a fan of shared ingress is linked to (1) and sometimes (2) - because firewalls are not always in the mix here - and that is that bringing all traffic into a single point introduces a large administrative blast radius. If a user has a bad day and accidentally makes the wrong change they can affect a lot of applications even if that change is not malicious.
 
 And in the AWS world, all accounts come with limits of some sort. If there are dozens, hundreds or thousands of applications relying on a single ingress point then the likelihood of running into some sort of API limit on the AWS side is high. Splitting the applications up is a wise move.

This repo is a middle ground - it delivers a design compromise where there is some level of shared ingress and centralised control; but it distributes the application traffic in a way that is scalable both in terms of AWS accounts and the ability to give responsibility for each application to different teams.

I still believe that distributed ingress is more scalable and a better solution overall - customers often tell me that they can see the benefits but they cannot make the leap to get there in the short term.

I can't take credit for the original idea - it comes from [an AWS blog post](https://aws.amazon.com/blogs/networking-and-content-delivery/how-to-securely-publish-internet-applications-at-scale-using-application-load-balancer-and-aws-privatelink/) by Tom Adamski so do go and check that out.
## Solution Components
- [Amazon Virtual Private Cloud](https://aws.amazon.com/vpc/) (VPC)
- [Application Load Balancer](https://aws.amazon.com/elasticloadbalancing/application-load-balancer/) (ALB)
- [Network Load Balancer](https://aws.amazon.com/elasticloadbalancing/network-load-balancer/) (NLB)
- [AWS Web Application Firewall](https://aws.amazon.com/waf/) (WAF)
- [Amazon CloudFront](https://aws.amazon.com/cloudfront/)
- [AWS PrivateLink](https://aws.amazon.com/privatelink/)
## Network Diagram
![Network diagram](network-diagram.png)

This diagram only shows a single Availability Zone to make it simpler to look at. However, I strongly encourage you to operate in multiple AZs for the obvious reason of redundancy. This will increase costs because PrivateLink charges for attachments on a per-AZ basis.
## Traffic Flow
![Traffic flow](traffic-flow.png)
## CloudFormation Templates
There are four CloudFormation templates - deploy them in the following order:
1. [Ingress VPC](cfn-ingress-vpc.yaml)
2. [Application VPC](cfn-application-vpc.yaml)
3. [Application configuration for the ingress VPC](cfn-ingress-application.yaml)
4. [Workload deployment for the application VPC](cfn-application-workload.yaml)

(3) and (4) can be swapped around - the order is unimportant. Note that there is a manual step after deploying (3) - more details below.

None of the templates allow you to choose the IP address range being used; while I could do that and get CloudFormation to divvy up the address space it adds complexity to the template that isn't needed from a demonstration perspective. As it is, the IP ranges chosen are more than is required - if you are short on IP address space then consider using only the minimum range that you require.
### Ingress VPC Template
This template creates the ingress VPC and Internet Gateway along with two public subnets and two private subnets with appropriate route tables.

Deploy this template only once - it is used by all applications.
### Application VPC Template
This template creates the application VPC, a NLB for distributing traffic to the workload and creates the PrivateLink endpoint service that will be shared with the ingress VPC.

For now, the NLB listener is HTTP only because creating certificates and verifying them in a demo is a little bit difficult. In the real world you have a choice - do you run your application via HTTP only (because it's easier) when the traffic is staying with AWS? Perhaps. But it is best practice to perform encryption in transit at all times so it would be far better to use HTTPS for the PrivateLink and NLB parts of this solution.

You will need to provide two parameters when deploying this template:

First is the name of the application. This can be more-or-less anything but it must be compatible with being used in a tag. This means (for example) no spaces. You **must** use the same application name when deploying the workload CloudFormation stack because this stack exports a few values that the workload stack needs to find. Those exports must be unique so the application name has to be unique as well.

Second is principal that is able to request connections to the PrivateLink Endpoint Service that is created by this template. Remember that this is designed to be deployed cross-account so you need to give the account where the ingress VPC is deployed permissions to request connections. This parameter will most likely be the same for every single application VPC that you create because there will only be a single ingress VPC in a single AWS account. The easiest way to do this is to give the entire account (where the ingress VPC is hosted) access by using a principal of `arn:aws:iam::ACCOUNTNUMBER:root` however this may be too permissive for you. If so, read [the PrivateLink documentation](https://docs.aws.amazon.com/vpc/latest/privatelink/configure-endpoint-service.html#add-remove-permissions) to see how you can further restrict access.

Finally, you will deploy this template once per application. You can have each application in a separate account; or you have multiple applications in a single account - it's completely up to you. For more information see the FAQ below.
### Application Ingress Configuration Template
This templates adds resources to the ingress VPC. It creates the ALB for the application; creates and assigns a security group to the ALB; creates a (very simple!) WAF ACL and assigns it to the ALB; sets up a CloudFront distribution with an origin pointing to the ALB; secures the connection to the ALB by using a prefix group and a custom header [as per the documentation](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/restrict-access-to-load-balancer.html); and finally requests a connection to the PrivateLink Endpoint Service that was created in the previous step..

For now, the ALB listener is HTTP only because creating certificates and verifying them in a demo is a little bit difficult. But in the real world, the connection from CloudFront to the ALB would be via HTTPS.

Once again there are three parameters required when deploying this template:

First (again) is the name of the application. This does not have to match the name given in the previous step but it will make sense for you to do so - purely so you can easily identify which components are assigned to which application. But otherwise the name can be completely different - it isn't used to link the two stacks together in any way.

Second is the value of the custom header that will be used to ensure that only your CloudFront distribution can access the ALB. Ideally this is a randomly generated string that is not used anywhere else.

Third is the name of the PrivateLink Endpoint Service that was created by the previous stack. This is given to you as an output by CloudFormation so copy that from the application account and paste it into this field.

At this point there is a manual step. You will need to go back to the account where the application VPC is deployed and accept the PrvateLink Endpoint connection request that has just been made for this application. This is a little inconvenient but it stops connections being maded without actions on both sides (the ingress account and the workload account). If you don't want this to happen, you can automatically accept connection requests by editing the application VPC template and changing the line `AcceptanceRequired: True` to `AcceptanceRequired: False`. There's no huge risk here because the permissions defined as to who can request connections will restrict those connections to the ingress account.

One thing this template does **not** do is create a WAF rule for CloudFront. This is definitely recommended - I haven't done it here because it must be deployed in `us-east-1` which means another template with additional instructions. This document is long enough already without doing that. But using WAF on CloudFront and WAF on ALB gives you the opportunity to filter traffic at two points and even apply different rules if necessary. In a production environment I strongly recommend that you do it.

Deploy this template once per application. It is deployed in the same account and region as the Ingress VPC Template.
### Application Workload Template
This template is separate because in a production environment you are probably going to have a network or infrastructure team deploy the VPC template and then you'll want to control deployment of the application separately. Keeping them separate also means you can completely change the workload without having to tear down and recreate the inter-VPC and inter-account PrivateLink connections.

The template deploys an auto-scaling group of two web servers running on t3.nano instances spread across two availability zones. Instead of EC2 you might use containers with [Amazon Elastic Container Service](https://aws.amazon.com/ecs/), [Amazon Elastic Kubernetes Service](https://aws.amazon.com/eks/) or [AWS Fargate](https://aws.amazon.com/fargate/). But for a demonstration, this will do.

The only parameter you need to provide is the application name. It is critical that this is the same as the application name provided in step two when the application VPC was deployed. This template looks for resources exported from that stack with specific names.

The web servers are not particularly intelligent - they provide a single static web page that shows the CloudFormation stack name and the hostname of the EC2 instance that is running. Enough to prove that everything is working correctly.

Deploy this template once per application in the same account as the application VPC template.
### Testing
An output from the applicatio ingress configuration template is a CloudFront distribution. Put that into your browser (or use a tool like `curl`) and you will see the static web page being served. Refresh the page and at some point you'll see slightly different output - the hostname will change indicating that the requests are being load-balanced between the two instances.

To see if the WAF rules are working, you can use `curl` to send the `block` parameter: `curl https://xxxxxxxxxxx.cloudfront.net/?block=blockme` and that request will be denied. You can also do that in your browser. The reason that the `Amplify` managed cache profile was chosen for CloudFront was so that requests with different query parameters (such as `block=blockme`) are not cached the same way as requests without those parameters.

For troubleshooting, check the target groups in the ingress VPC and in the application VPC and ensure that the targets are marked as `Healthy`. If your WAF rules are not triggering, enable logging and look in CloudWatch Logs.
### Notes
The CloudFormation templates deploy resources into two Availability Zones. When operating in multiple accounts (and therefore creating subnets in different VPCs in those accounts), make sure that you are using the same Availability Zones by ccross-referencing the Availability Zone identifiers. You can see these by looking at the subnet details in the AWS console or by using the AWS CLI and running `aws ec2 describe-availability-zones`.

While the WAF components for the ALB are shown in the diagram as being "beside" the ALBs, in reality they are deployed "on" each ALB.

This template only deploys a HTTP endpoint but the design fully supports (and I strongly recommend) that you use HTTPS because encryption is a good (and should be mandatory!) thing. You can incorporate HTTPS into the CloudFormation template - I have not done so in this case because it requires you to confirm ownership of domains which is not practical as a demonstration.

HTTPS connections require valid certificates. In a production environment you would be using your own domain name and so you should use [AWS Certificate Manager](https://aws.amazon.com/certificate-manager/) to deploy certificates for CloudFront and ALB because it is free and it will automatically rotate your certificates for you.

Given that there will be multiple WAF instances deployed (one for each ALB and one for CloudFront) you should also consider using [AWS Firewall Manager](https://aws.amazon.com/firewall-manager/) to centrally manage the rules.
## Frequently Asked Questions
### Why is PrivateLink being used?
PrivateLink is providing two benefits:
1. It allows for the application VPCs to have IP addresses that overlap with each other and/or the ingress VPC. In most environments this will not be the case and I'd advise that you shouldn't do this in any case as it will generally make your life harder if you do. I'm also opinionated about this and you should [generally avoid deliberately creating networks with IP overlap if you can](https://aws.amazon.com/blogs/networking-and-content-delivery/connecting-networks-with-overlapping-ip-ranges/).
2. Access to applications can only be initiated from the ingress VPC to the application VPCs. The application VPCs cannot use the ingress VPC as an egress path. To put it another way, the applications cannot initiate traffic to the internet throught the ingress VPC.

Of these two things, (2) is probably the most relevant although PrivateLink is not the only way to deliver traffic to the applications. You might use Transit Gateway...
### Can I use Transit Gateway instead of PrivateLink?
Yes. You can link all the VPCs together using Transit Gateway. On the application ALB in the ingress VPC you would use the private IP addresses of the NLB in the application VPC as IP targets. Note that if you intend on having the IP address ranges of the application VPCs overlap you must use PrivateLink. In most customer environments this is not the case.

Using Transit Gateway instead of PrivateLink means that you could use the ingress VPC as an egress VPC as well. This might be desirable. Or not.
### How do applications reach the internet?
When application instances initiate connections out of their VPC the traffic will use the routes specified in that VPC. Normally, the VPC will be connected to a Transit Gateway so that it can communicate with other VPCs in your environment. Presumably, that Transit Gateway will have some way of allowing VPCs that are attached to it to access the internet.

[Here's a blog post](https://aws.amazon.com/blogs/networking-and-content-delivery/creating-a-single-internet-exit-point-from-multiple-vpcs-using-aws-transit-gateway/) describing how to set up a single internet exit point using Transit Gateway.
### Does this work across multiple accounts?
Yes. The intention is that the network or security team will control the ingress VPC; but the application teams each have their own separate VPCs. Those VPCs can (and should be!) in different accounts. Using different accounts greatly simplifies permissions; eestablishes firm boundaries around each application (in terms of security and billing); reduces administrative blast radius; and allows each application to operate independently in terms of AWS account limits.
### Is using multiple accounts mandatory?
No, this solution works equally well in a single account but it makes more sense to use multiple accounts to maintain administrative separation as per the question/answer above. If you were running in a single account you might as well run all the resources in a single VPC which would be a lot less expensive. But then it is avoiding the point of this exercise which is to have separation of administrative duties.
### Can I inspect the ingress traffic with my firewall?
Yes, you can. But you should think carefully before doing this. It can lead to the situation where the firewalls are a single shared point of failure for all of the applications.

If you do wish to inspect all traffic, you can use [Gateway Load Balancer](https://aws.amazon.com/elasticloadbalancing/gateway-load-balancer/) to send traffic to the firewall of your choice or use [AWS Network Firewall](https://aws.amazon.com/network-firewall/). You can see how to do this in the [launch blog post](https://aws.amazon.com/blogs/networking-and-content-delivery/introducing-aws-gateway-load-balancer-supported-architecture-patterns/) for Gateway Load Balancer - the patterns for Network Firewall are the same.

You can choose to inspect the traffic betore it is delivered to the ALBs (using an ingress route table); or you can inspect in beteen the ALBs and the PrivateLink endpoints (by adding an inspection subnet and routing traffic to the Gateway Load Balancer/Network Firewall endpoint in that subnet).

If you must do this (and I'm not encouraging this design modification) I would recommend inspecting between the ALBs and the PrivateLink endpoints as you can use ALB and WAF (as well as CloudFront and WAF) to get rid of unwanted traffic in less expensive and scalable way, leaving only "interesting" traffic for the firewalls to inspect. The less traffic to inspect, the less expensive it will be to inspect that traffic.