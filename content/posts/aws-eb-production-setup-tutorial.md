---
title: "Get Your Service Production-Ready on AWS Free Tier — A Step-By-Step Guide"
date: 2021-04-25T18:03:35+02:00
draft: true
---

Recently, I was setting up a simple backend service for my favorite party game [Ryggio](http://rygg.io) on AWS.
To my surprise, a comprehensive[^1] guide was missing—which motivated my attempt to create one.

## The Setup

We will be using the following tools to deploy our app on [AWS Free Tier](https://aws.amazon.com/free/?all-free-tier.sort-by=item.additionalFields.SortRank&all-free-tier.sort-order=asc&awsf.Free%20Tier%20Types=*all&awsf.Free%20Tier%20Categories=*all):

* Docker to package our application
* AWS Elastic Beanstalk with a **single instance deployment** on Amazon Linux 2
* Let's Encrypt for generating a certificate
* Amazon S3 for storing the certificate data
* Namecheap for configuring our custom domain (of course, any provider will do)

## Step 1: Dockerize Your Project

To follow this guide, it's easiest if your project comes with a `Dockerfile`. This is unfortunately out of scope for this guide. If you just want to follow the tutorial for fun, please check out my [example project](https://github.com/ppati000/aws-eb-docker-example)! The repository contains all the files you need to get started.

## Step 2: Set Up AWS Elastic Beanstalk

Getting our app up and running is fairly easy. Here's what you will need to do:

1. Create a zip file containing your entire project. Make sure the `Dockerfile` is on the top directory level (i.e. don't zip the project folder, but the files inside it).
2. Go to [Elastic Beanstalk](https://eu-central-1.console.aws.amazon.com/elasticbeanstalk/home?region=eu-central-1#/welcome) and click on "Create Application".
3. Enter a project name and select Docker as platform. As platform branch, make sure to choose **Docker running on 64bit Amazon Linux 2**. Note that if you choose Amazon Linux 1, some stuff in this guide may not work.
4. Select "Upload your code" and upload your previously created zip file. Then add any text as version label, e.g. "Version 1".
5. Click on "Configure more options" on the bottom of the page. Then, select **Single instance (Free Tier eligible)**. This is the most appropriate option for a low-traffic application. And don't worry, you will be able to change this later.
6. That's it! Click on "Create application" on the bottom of the page. Then wait a few minutes for everything to be set up—you'll be redirected to your app's status page automatically.

This is approximately what you'll be seeing after completing Step 2:

![The Amazon Elastic Beanstalk status page is showing us that everything is up and running.](/posts/aws-eb-production-setup-tutorial/statusPage.png)

Well, that was easy! So what's missing now?

* Our app is still running on some cryptic URL like `http://ebtestapplication-env.eba-yb3jbrc5.eu-central-1.elasticbeanstalk.com`. We'll want to use our own domain instead.
* We will need to set up HTTPS / TLS encryption so that people can access our service securely. Nowadays, this is a [must](https://www.cloudflare.com/es-es/learning/ssl/why-use-https/) for any application used by anyone other than yourself.

The following steps will cover just that.

## Step 3: Set Up a Custom Domain

This step is done using your domain provider's admin page.
This guide uses Namecheap, but any other provider will have a similar interface.

In Namecheap, select the domain name you would like to use and navigate to *Advanced DNS*. Add a new record with the following properties:

* Type: `CNAME`
* Host: `@` if you want to use the top level domain, or the name of your subdomain, e.g. `api` if you want to use `api.your-domain.org`. For this example we'll use `aws-eb-tutorial`, so our server will be available at `aws-eb-tutorial.patrick.engineering`.
* Target: Enter your app's Elastic Beanstalk domain, e.g. `ebtestapplication-env.eba-yb3jbrc5.eu-central-1.elasticbeanstalk.com`.

Here's what it will look like when saved:

![DNS settings showing type, host and target in Namecheap.](/posts/aws-eb-production-setup-tutorial/dnsSettings.png)

Nice! After a few minutes[^2], the example app is reachable at `http://aws-eb-tutorial.patrick.engineering`!

## Step 4: Configure TLS Encryption

[^1]: A good, yet not entirely complete guide which this post is partly based on is available [here](https://medium.com/@hzburki/configure-ssl-certificate-elastic-beanstalk-single-instance-a2846211851b).
[^2]: Well, that depends. If the app isn't reachable yet even though it runs fine using your Elastic Beanstalk URL, grind yourself some coffee and check back later.
