---
layout: post
title:  "Price Tracker: A Practical Dive into Serverless Applications"
date:   2024-01-13 13:00:00 +0100
categories: jekyll update
---

## List of contents:
1. A few words at the beginning...
2. Understanding Serverless and AWS Cloud Essentials.
3. About the app.
3. Architecture overview.
4. Hands-on.
    - Using AWS Management Console.
    - Automated, with AWS CLI.
5. Summary.

## A Few Words at the Beginning
In the realm of modern cloud computing, serverless architecture has been a buzzword, yet its practical applications often remain unexplored. This article aims to bridge this gap by demonstrating the development of a Price Tracker web application using serverless technology. Amidst the abundance of theoretical knowledge, article is focusing on a hands-on example to showcase how these concepts come to life in real-world scenarios.

## Understanding Serverless and AWS Cloud Essentials
Serverless computing is a transformative cloud computing paradigm that enables developers to build and run applications without managing servers. It is about focusing on code, not infrastructure, allowing for more agility and innovation. AWS Cloud, a leader in cloud computing, offers a comprehensive ecosystem to implement serverless architecture effectively. Their suite of services, including AWS Lambda, API Gateway, and DynamoDB, empowers developers to deploy scalable, highly available, and cost-effective applications. By leveraging AWS, the serverless concept transcends theory, offering a robust platform where it can be applied in the real world to solve complex problems and streamline operations.

## About the app
The Price Tracker app is designed for those interested in observing how product prices change over time. It automatically follows the price changes, store it in database and displays this information in visual graphs, offering a clear view of price trends. This tool is particularly useful for educational or personal projects, highlighting the practical application of serverless technology in creating web applications. Final app view has been shown below:

![image-title](/assets/images/serverless/final_view.png)

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
In this hands-on section, choose between two approaches: manual setup of Price Tracker web app via the AWS Management Console (recommended) or a script-based deployment (experimental). The manual method through the console offers an in-depth, step-by-step understanding, while the script approach, allows for a quick and automated setup. Script based approach is for developers with at least basic knowledge from AWS. Each provides a unique way to experience and interact with the application, catering to different preferences, experience and time constraints. Regardless of which option we choose, the following steps need to be performed:

1. Create policies: tracker-policy, handler-policy, gateway-policy.
2. Create roles: tracker-role, handler-role, gateway-role.
3. Create bucket (public): price-tracker-plots (name of it can be customized in app.config file).
    - **NOTE** Bucket need to be public because end user needs to download graphs from it during loading a web page. It is always good practice to remove bucket or at least limit access to it when it's no longer necessary.
4. Create Tracker Lambda - tracker-exampleItem.
5. Create Handler Lambda - tracker-handler.
6. Add tracker-handler as destination to tracker-exampleItem lambda.
7. Add EventBridge rule - tracker-exampleItem, rule: every two minutes.
8. Create gateway Lambda - tracker-gateway.
9. Create http endpoint in AWS Gateway and deploy.
10. Test a page (it can take a few minutes until graph will be generated).

### Requirements
- Docker, install from [docker.com](https://www.docker.com).
- AWS Access  AWS Management Console: [link](https://aws.amazon.com/console/) (Way 1).
- AWS CLI installed [link](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) and configured [link](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-quickstart.html) (Way 2)

#### Note
The name of an Amazon S3 bucket must be unique across all existing bucket names in Amazon S3 not just within your own AWS account or a specific AWS region. It is a big chance that default name `price-tracker-plots` cannot be use. That's the reason why this name is customizable in app.config file.  

### Way 1: AWS Management Console
1. Download the repository:
```
git clone git@github.com:czczajka/price-tracker-serverless.git
```
2. Modify bucket name in app.config file. See note section below for more details.
3. Build all necessary artifacts:
```
./scripts/build.sh
```
4. See the video and reproduce the steps.

#### Note
The name of an Amazon S3 bucket must be unique across all existing bucket names in Amazon S3 not just within your own AWS account or a specific AWS region. It is a big chance that default name `price-tracker-plots` cannot be use. That's the reason why this name is customizable in app.config file. It's important to keep name consistent in AWS environment and in app.config file. Additionally after changing `app.config` file, project should be re-builded.

<iframe width="560" height="315" src="https://www.youtube.com/embed/nhlfUf5ZV1E?si=u27NIzqo6B4GAZME" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

### Way 2: Automated, with AWS CLI (experimental)
In second method, we employ an script, which is using AWS CLI to do all the same work in automated way. the script is not yet mature and incorporates `dirty hacks`. Nice improvement will be get rid of those sleep commands. As an output from the script you should receive url, where Price Tracker app is launched.

1. Run deployment script:
```
./scripts/deploy_aws.sh
```
2. Go to the url (last line outputted from the script) and check if it is working correctly. It can take a couple of minutes before graph will be available.
3. Run cleanup script:
```
./scripts/cleanup_aws.sh
```

## Summary
The article "Price Tracker: A Practical Dive into Serverless Applications" offers a hands-on guide to building a serverless price tracker using AWS. It covers serverless computing fundamentals, AWS Cloud essentials, and a step-by-step walkthrough of creating the application, from setting up AWS services like Lambda and DynamoDB to deploying the final web app. The tutorial concludes with showcasing the web app interface, demonstrating the practical application of serverless technology in real-world scenarios.
