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
<br/>
<br/>

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

2. kOps requires that you have AWS credentials configured in your environment. The `aws configure` command is the fastest way to set up your AWS CLI installation for general use. Run the command and follow the prompts. You can use Administrator IAM policy, but if you want to limit the permissions required by kOps, the minimum required IAM privileges you will need are:

- AmazonEC2FullAccess
- AmazonRoute53FullAccess
- AmazonS3FullAccess
- IAMFullAccess
- AmazonVPCFullAccess
- Events:
  - DeleteRule
  - ListRules
  - ListTargetsByRule
  - ListTagsForResource
  - PutEvents
  - PutRule
  - PutTargets
  - RemoveTargets
  - TagResource
- SQS:
  - CreateQueue
  - DeleteQueue
  - GetQueueAtttributes   
  - ListQueues
  - ListQueueTags 

3. [Install kOps](https://kops.sigs.k8s.io/getting_started/install/) in your environment. You can also follow this guide to install kOps for other architectures and platforms. At the time of writing, the latest version of kOps is v1.23.1

    ```bash
    export KOPS_VERSION=v1.23.1
    curl -LO https://github.com/kubernetes/kops/releases/download/${KOPS_VERSION}/kops-linux-amd64
    chmod +x kops-linux-amd64
    sudo mv kops-linux-amd64 /usr/local/bin/kops
    kops version
    ```

4. Install Kubectl. You can also follow [this guide](https://kubernetes.io/docs/tasks/tools/install-kubectl/) for other architectures and platforms. You should use the same major kubectl version as the kOps version selected.

    ```bash
    export KUBECTL_VERSION=v1.23.6
    sudo curl --silent --location -o /usr/local/bin/kubectl https://storage.googleapis.com/kubernetes-release/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl
    sudo chmod +x /usr/local/bin/kubectl
    kubectl version
    ```

5. In addition to kOps and kubectl, install [yq](https://github.com/mikefarah/yq/), a portable command-line YAML processor. You can follow yq [installation instructions](https://mikefarah.gitbook.io/yq/) for your system. On Cloud9 and Linux, we can install yq with the command on the right. The command requires that Go tools are installed in your environment. You can run  `go version` to check if Go is already installed in your environment; if it is not, [install go tools](https://golang.org/doc/install#install) before proceeding with this step.

    ```bash
    GO111MODULE=on go get github.com/mikefarah/yq ; export PATH=$PATH:~/go/bin
    ```
</details>

<br/>
<br/>

<details>
  <summary>Step 2: Set up kOps Cluster environment and state store</summary>
<br/>

In this step we will configure some of the environment variables that will be used to set up our environment, and create and configure the S3 bucket that kOps will use as [states store](https://kops.sigs.k8s.io/state/).

<br/>

1. Export environment variables according to the following requirements:
- The name of our cluster will be **“spot-kops-cluster”**. To reduce the dependencies on other services, in this tutorial we will create our cluster using [Gossip DNS](https://kops.sigs.k8s.io/gossip/), hence the cluster domain will be **k8s.local** and the fully qualified name of the cluster **spot-kops-cluster.k8s.local**.
- You will also create an S3 bucket where kOps configuration and the cluster's state will be stored. We will use [uuidgen](https://man7.org/linux/man-pages/man1/uuidgen.1.html) to generate a [unique S3 bucket name](https://docs.aws.amazon.com/AmazonS3/latest/dev/UsingBucket.html).
- In the above command, you will set the environment variables that will be used across the rest of the session.

    ```bash
    export NAME=spot-kops-cluster.k8s.local
    export KOPS_STATE_PREFIX=spot-kops-$(uuidgen)
    export KOPS_STATE_STORE=s3://${KOPS_STATE_PREFIX}
    ```

2. Additionally you will set a few other environment variables that define the region and availability zones where your cluster will be deployed. In this tutorial, the region will be “us-east-1”, you can change this and point it to the region where you would prefer running your cluster.

    ```bash
    export AWS_REGION=us-east-1
    export AWS_REGION_AZS=$(aws ec2 describe-availability-zones \
    --region ${AWS_REGION} \
    --query 'AvailabilityZones[0:3].ZoneName' \
    --output text | \
    sed 's/\t/,/g')
    ```

3. Now that you have the name of your cluster and S3 State Store bucket defined, let's create the S3 bucket.

    ```bash
    aws s3api create-bucket \
    --bucket ${KOPS_STATE_PREFIX} \
    --region ${AWS_REGION} \
    ```

4. Once the bucket has been created, you can apply one of kOps best practices by enabling S3 Versioning on the bucket. S3 is acting as the state store, and by enabling versioning on the bucket you will be able to recover your cluster back to a previous state and configuration.

    ```bash
    aws s3api put-bucket-versioning \
    --bucket ${KOPS_STATE_PREFIX} \
    --region ${AWS_REGION} \
    --versioning-configuration Status=Enabled
    ```
</details>
