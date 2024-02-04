---
layout: post
title:  "Price Tracker: A Practical Dive into Serverless Applications"
date:   2024-01-13 13:00:00 +0100
categories: jekyll update
---

## List of contents:
1. A Few Words at the Beginning...
2. Understanding Serverless and AWS Cloud Essentials.
3. About the app.
3. Architecture overview.
4. Hands-on.
    - Using AWS Management Console.
    - Using the provided script.
5. Summury.

## A Few Words at the Beginning
In the realm of modern cloud computing, serverless architecture has been a buzzword, yet its practical applications often remain unexplored. This article aims to bridge this gap by demonstrating the development of a Price Tracker web application using serverless technology. Amidst the abundance of theoretical knowledge, article is focusing on a hands-on example to showcase how these concepts come to life in real-world scenarios. It's not just about understanding serverless architecture; it's about witnessing its potential in solving actual problems and its application in everyday digital solutions.

## Understanding Serverless and AWS Cloud Essentials
Serverless computing is a transformative cloud computing paradigm that enables developers to build and run applications without managing servers. It's about focusing on code, not infrastructure, allowing for more agility and innovation. AWS Cloud, a leader in cloud computing, offers a comprehensive ecosystem to implement serverless architecture effectively. Their suite of services, including AWS Lambda, API Gateway, and DynamoDB, empowers developers to deploy scalable, highly available, and cost-effective applications. By leveraging AWS, the serverless concept transcends theory, offering a robust platform where it can be applied in the real world to solve complex problems and streamline operations.

## About the app
The Price Tracker app is designed for those interested in observing how product prices change over time. It automatically updates and displays this information in visual graphs, offering a clear view of price trends. This tool is particularly useful for educational or personal projects, showcasing the practical application of serverless technology in creating web applications.

## Architecture overview
The architecture of Price Tracker application, built on AWS, exemplifies a modern serverless design that leverages the power and flexibility of cloud resources. Below is a visual representation of the architecture, followed by a detailed explanation.
![image-title](/assets/images/serverless/architecture_overview.svg)

The system operates through a series of coordinated actions between AWS services to track and manage data:
- **Tracking Lambda:** This function is at the heart of the tracking operation. Its job is to gather and return relevant data about an item, including its value and a timestamp.
- **EventBridge Rule:** AWS EventBridge is configured with a rule to trigger the Tracking Lambda function at regular intervals. This setup ensures that the item is tracked periodically, making the data collection process consistent and automated.
- **Handler Lambda:** Once the Tracking Lambda function gathers the data, the Handler Lambda function takes over. It processes this data, storing it in a DynamoDB table for persistence. In addition to storage, this function is also responsible for generating visual plots of the tracked data, which it then uploads to an AWS S3 bucket.
- **Gateway Lambda and API Gateway:** The Gateway Lambda function interfaces with the AWS API Gateway, which acts as the front-end for user interactions. When a user makes a request through the API Gateway, the Gateway Lambda function responds by fetching and displaying the relevant data and visualizations from the S3 bucket.
- **S3 bucket (public):** The S3 bucket is publicly accessible. This is essential as the API Gateway uses links from this bucket in HTML to display graphs on the webpage. The public setting allows all users to access and download these HTML-formatted graphs.

## Hands-on
In this hands-on section, choose between two approaches: manual setup of Price Tracker web app via the AWS Management Console or a swift script-based deployment. The manual method through the console offers an in-depth, step-by-step understanding, while the script approach, allows for a quick and automated setup. Each provides a unique way to experience and interact with the application, catering to different preferences, experience and time constraints.

### Requirements
- Docker, install from [docker.com](https://www.docker.com).
- AWS Accesses, you need to have access to AWS Management Console: [link](https://aws.amazon.com/console/). Additionally if you would like to use script to ceate an app (Way 2) you will need:
    - AWS CLI Installation, install the AWS CLI [link](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) and setup credentials to acccess AWS programatically [link](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-quickstart.html)

### Steps to reproduce
1. Create policies: tracker-policy, handler-policy, gateway-policy.
2. Create roles: tracker-role, handler-role, gateway-role.
3. Create bucket (public): price-tracker-plots (name of it can be customized in app.config file).
    - **NOTE** bucket need to be public because end user need to download graphs from it. It is always good practice to remove bucket or at least limit access to it when it's not longer necessary.
4. Create Tracker Lambda - tracker-exampleItem.
5. Create Handler Lambda - tracker-handler.
6. Add tracker-handler as destination to tracker-exampleItem lambda.
7. Add EventBridge rule - tracker-exampleItem, rule: every two minutes.
8. Create gateway Lambda - tracker-gateway.
9. Create http endpoint in AWS Gateway and deploy.
10. Final tests.

### Common steps (applicable for Way 1 and Way 2)
1. Download the repository.
2. Modify bucket name in app.config file. See note section below for more details.
3. Build lambdas:
{% highlight ruby %}
./scripts/build.sh
{% endhighlight %}

#### Note
The name of an Amazon S3 bucket must be unique across all existing bucket names in Amazon S3 not just within your own AWS account or a specific AWS region. It is a big chance that default name `price-tracker-plots` cannot be use. That's the reason why this name is customizable in app.config file.  

### Way 1: AWS Management Console
See the video and reproduce the steps.
<iframe width="560" height="315" src="https://www.youtube.com/embed/nhlfUf5ZV1E?si=u27NIzqo6B4GAZME" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>


### Way 2: Automated
1. Run deployment script:
{% highlight ruby %}
./scripts/deploy.sh
{% endhighlight %}
2. Run http endpoint and check if it is working correctly. It can take a couple of minutes before new data will be available (tracker Lambda is started every two minuets)
3. Run cleanup script:
{% highlight ruby %}
./scripts/deploy.sh
{% endhighlight %}

### App view
This chapter presents the web app's final interface, showcasing the outcome of our work with AWS services. Below is an image of the app displaying an item graph, demonstrating the functionality developed throughout the tutorial.

![image-title](/assets/images/serverless/final_view.png)

## Summury
The article "Price Tracker: A Practical Dive into Serverless Applications" offers a hands-on guide to building a serverless price tracker using AWS. It covers serverless computing fundamentals, AWS Cloud essentials, and a step-by-step walkthrough of creating the application, from setting up AWS services like Lambda and DynamoDB to deploying the final web app. The tutorial concludes with showcasing the web app interface, demonstrating the practical application of serverless technology in real-world scenarios.

## End Notes (to remove)
possible other title:
"Demystifying Serverless: A Real-World Price Tracker Example"