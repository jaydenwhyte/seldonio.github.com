---
layout: default
title: Getting Started
---

# Getting Started

Seldon has created a virtual machine with all services pre-wired for you to test with your service data and movie recommender demo.

 * Ways to get the VM :
   * [Vagrant Virtual Machine installation](vm.html)
   * [AWS AMI](vm-aws.html)
 * [Movie Recommender Demo](movie-recommender-demo.html)
 * Creating Recommenders with your own data
   * [Loading your data into Seldon](data.html)
   * [Creating Content Recommendation Models](content-recommendation-models.html)


To hear about further releases [sign up for the beta program](http://www.seldon.io/open-source)


## Can I use Seldon in my project?
Yes. Seldon is licensed under the permissive Apache Licence, Version 2.0. So please feel free to use Seldon in either commercial or non-commercial projects. And no, you do not have to make your project open source.

## Is it production ready?
Seldon VM is closed alpha release for testing and demonstration purposes only. Do not use it in production. Seldon will shortly release the project source code and seperate Docker containers for the separate components which have been tested in high traffic production environments.

Seldon has been running a SaaS service at scale (over a billion API calls per month) with extremely high availability and low latency. If you want to run Seldon Cloud in production, please contact [hello@seldon.io](mailto:hello@seldon.io)
