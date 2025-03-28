+++
date = '2025-03-07T15:20:13-06:00'
draft = true
title = 'The AWS Global Accelerator'
+++
## Introduction
Let me guess, you're a seasoned AWS Pro. You've deployed applications on top of EC2s or as Lambda Functions, heck maybe you've even deployed applications onto EKS. You've fronted these applications with an AWS Load Balancer and pointed your DNS to that. Pretty easy stuff. Now you've been hearing chatter about something called AWS Global Accelerator. Okay, geat. How does that fit into your current infrastructure stack and what are the benefits of it that would warrant you to begin using it?

Before we can answer this question. Let's look at a motivational example.

## Motivational Examples

### Motivational Example 1: Active-Active DR Deployment

Let's say your application is deployed on a couple of lambda functions fronted by an application load balancer. This is only deployed in a single region, `us-east-1`, but recently a new client based out of Sydney has signed up for your service. To get ahead of the game, you decide to go multi-region with your lambda functions, where you continue your infrastructure in North America, `us-east-1`, but also deploy to Asia Pacific region, `ap-southeast-2`.

How can we reduce network latency?

### Motivational Example 2: Active-Passive DR Deployment

How can I failover into my passive region as quickly as possible, avoiding DNS issues?

## Global Accelerator

### Deterministic Routing
Global Accelerator provides you with two public static IP addresses that are part of the AWS Edge Network. These IP addresses do not change. This means during a regional failover, I do not have to wait for my client's DNS cache to expire before they can begin accessing the site again in the failover region. To the client's perspective, they are still using the same two IPs for their requests. It is unknown to them that `us-east-1` may have been serving requests all this time, but suddenly `us-west-2` is now.

### AWS Global Network
Two static IPs are assigned and deployed at an AWS Edge Location. This means that we are able to move our traffic off of the public internet and onto the AWS Global Network as soon as possible. By doing this, we are able to improve network traffic performance and security.

### Security
As Global Accelerator makes use of the AWS Edge Network, we are opted into AWS Shield Standard by default. This will protect us from DDoS and other malicious attacks. All of that at no additional cost to us! Yay AWS Shield! :tada:

We are also able to allow requests from the public internet access our private resources while not have a single public endpoint in our AWS Organization! Well, technically we do, but it is part of the Global Accelerator. I'm no Security Engineer, but that sounds pretty appealing to me. It is the single point of entry, meaning a Security Engineer should not see a single public endpoint in the entire AWS Organization.

:warning: To configure Global Accelerator to forward traffic to a private Application Load Balancer, you do still need an Internet Gateway deployed in your VPC.

### Highly Available with Regional Failovers
Global Accelerator is perfect for regional failovers while remaining highly available. The Global Accelerator is configured with endpoints in the different regions. 

## Demo

## Conclusion
