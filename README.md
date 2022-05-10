# Run a Cost Optimized kOps Kubernetes cluster with Amazon EC2 Spot Instances

## Introduction

[Amazon EC2 Spot Instances](https://aws.amazon.com/ec2/spot/) offer spare compute capacity available in the AWS Cloud at steep discounts compared to On-Demand prices. EC2 can interrupt Spot Instances with two minutes of notification when EC2 needs the capacity back. You can use Spot Instances for various fault-tolerant and flexible applications. Some examples are analytics, containerized workloads, high-performance computing (HPC), stateless web servers, rendering, CI/CD, and other test and development workloads.

One of the best practices to successfully adopt Spot Instances is to implement Spot Instance diversification as part of your configuration. Spot Instance diversification helps to procure capacity from multiple Spot Instance pools, both for scaling up and for replacing Spot Instances that may receive a Spot Instance termination notification. A Spot Instance pool is a set of unused EC2 instances with the same Instance type, operating system and Availability Zone (for example, m5.large on Red Hat Enterprise Linux in us-east-1a).

In this tutorial you will learn how to add Spot Instances to your kOps Kubernetes clusters, while adhering to Spot Instance best practices. This will allow you to run applications without compromising performance or availability. Kubernetes Operations (kOps) is an open source project that provides a cohesive set of tools for provisioning, operating, and deleting Kubernetes clusters in the cloud. As part of the tutorial, you will deploy a kOps Kubernetes deployment and autoscale it on your Spot Instance worker nodes by using Kubernetes Cluster-Autoscaler and Karpenter.

## What You Will Learn

- How to set up and use the kOps CLI to create a Kubernetes cluster with On-Demand nodes
- How to add Instance Groups with Spot Instances to your cluster, automatically leveraging best practices
- How to deploy the AWS Node Termination Handler
- How to deploy the Kubernetes Cluster Autoscaler
- How to deploy a sample application, test that it is running on Spot Instances and that it properly scales
- How to clean up your resources

<details>
  <summary>Step 1: Set up AWS CLI, kOps, and kubectl</summary>
  <br/>
  In this step you will install all the dependencies that you will need during the tutorial.
  <br/>
    
    1. Install version 2 of the AWS CLI by running the following commands — if you’re using Linux — or follow the instructions in the [AWS CLI installation guide](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) for different operating systems.

    ```bash
    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
    unzip awscliv2.zip
    sudo ./aws/install
    ```

</details>
