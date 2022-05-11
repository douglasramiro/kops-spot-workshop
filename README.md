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

5. In addition to kOps and kubectl, install [yq](https://github.com/mikefarah/yq/), a portable command-line YAML processor. You can follow yq [installation instructions](https://mikefarah.gitbook.io/yq/) for your system. On Cloud9 and Linux, you can install yq with the command on the right. The command requires that Go tools are installed in your environment. You can run  `go version` to check if Go is already installed in your environment; if it is not, [install go tools](https://golang.org/doc/install#install) before proceeding with this step.

    ```bash
    GO111MODULE=on go get github.com/mikefarah/yq ; export PATH=$PATH:~/go/bin
    ```
</details>

<br/>

<details>
  <summary>Step 2: Set up kOps Cluster environment and state store</summary>
<br/>

In this step you will configure some of the environment variables that will be used to set up your environment, and create and configure the S3 bucket that kOps will use as [states store](https://kops.sigs.k8s.io/state/).


1. Export environment variables according to the following requirements:
    - The name of your cluster will be **“spot-kops-cluster”**. To reduce the dependencies on other services, in this tutorial you will create your cluster using [Gossip DNS](https://kops.sigs.k8s.io/gossip/), hence the cluster domain will be **k8s.local** and the fully qualified name of the cluster **spot-kops-cluster.k8s.local**.
    - You will also create an S3 bucket where kOps configuration and the cluster's state will be stored. You will use [uuidgen](https://man7.org/linux/man-pages/man1/uuidgen.1.html) to generate a [unique S3 bucket name](https://docs.aws.amazon.com/AmazonS3/latest/dev/UsingBucket.html).
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
    --region ${AWS_REGION} 
    ```

4. Once the bucket has been created, you can apply one of kOps best practices by enabling S3 Versioning on the bucket. S3 is acting as the state store, and by enabling versioning on the bucket you will be able to recover your cluster back to a previous state and configuration.

    ```bash
    aws s3api put-bucket-versioning \
    --bucket ${KOPS_STATE_PREFIX} \
    --region ${AWS_REGION} \
    --versioning-configuration Status=Enabled
    ```
</details>

<br/>

<details>
  <summary>Step 3: Cluster creation and On-Demand node configuration</summary>
<br/>

In this step you will create the cluster control plane and a kOps InstanceGroup with OnDemand instances. You will also add some labels to the group, so that you can place pods accordingly later on.

1. It is now time to create the cluster. You will build a [Highly Available (HA)](https://kops.sigs.k8s.io/operations/high_availability/) cluster using m5.large instances for the [kubernetes masters](https://kubernetes.io/docs/concepts/) spread across three Availability Zones. Additionally you will create an InstanceGroup with two t3.large OnDemand worker nodes, that you will use to demonstrate how you can configure your applications to run on Spot or OnDemand Instances, depending on the type of workflow.

    ```bash
    kops create cluster \
    --name ${NAME} \
    --state ${KOPS_STATE_STORE} \
    --cloud aws \
    --master-size m5.large \
    --master-count 3 \
    --master-zones ${AWS_REGION_AZS} \
    --zones ${AWS_REGION_AZS} \
    --node-size t3.large \
    --node-count 2 \
    --dns private 
    ```

2. Great! The output of the previous command displays all the resources that will be created. You can check that the cluster configuration has been written to the kOps state S3 bucket. The following command should showcase the cluster state, and yield and an output similar to the following one:

    ```bash
    aws s3 ls --recursive ${KOPS_STATE_STORE}
    ```

    ```bash
    2022-05-11 00:08:21          0 spot-kops-cluster.k8s.local/clusteraddons/default
    2022-05-11 00:08:21       1723 spot-kops-cluster.k8s.local/config
    2022-05-11 00:08:21        454 spot-kops-cluster.k8s.local/instancegroup/master-us-east-1a
    2022-05-11 00:08:21        454 spot-kops-cluster.k8s.local/instancegroup/master-us-east-1b
    2022-05-11 00:08:21        454 spot-kops-cluster.k8s.local/instancegroup/master-us-east-1c
    2022-05-11 00:08:21        450 spot-kops-cluster.k8s.local/instancegroup/nodes-us-east-1a
    2022-05-11 00:08:21        450 spot-kops-cluster.k8s.local/instancegroup/nodes-us-east-1b
    2022-05-11 00:08:21        450 spot-kops-cluster.k8s.local/instancegroup/nodes-us-east-1c
    ```

3. As for the two nodes in the InstanceGroup that you created, you should label those as OnDemand nodes by adding a lifecycle label. kOps created an instance group per AZ for your nodes, so you will apply the changes to each of them. To merge the new configuration attributes to the cluster nodes, you will use yq.

    ```bash
    for availability_zone in $(echo ${AWS_REGION_AZS} | sed 's/,/ /g')
    do
    NODEGROUP_NAME=nodes-${availability_zone}
    echo "Updating configuration for group ${NODEGROUP_NAME}"
    cat << EOF > ./nodes-extra-labels.yaml
    spec:
      nodeLabels:
        kops.k8s.io/lifecycle: OnDemand
    EOF
    kops get instancegroups --name ${NAME} ${NODEGROUP_NAME} -o yaml > ./${NODEGROUP_NAME}.yaml
    yq merge --overwrite --inplace ./${NODEGROUP_NAME}.yaml ./nodes-extra-labels.yaml
    aws s3 cp ${NODEGROUP_NAME}.yaml ${KOPS_STATE_STORE}/${NAME}/instancegroup/${NODEGROUP_NAME}
    done
    ```

4. You can validate the result of your changes by running the following command, and verifying that the labels have been added to the spec.nodeLabels section. The output of this command should be: 
    - Instancegroup nodes-us-east-1a contains label kops.k8s.io/lifecycle: OnDemand
    - Instancegroup nodes-us-east-1b contains label kops.k8s.io/lifecycle: OnDemand
    - Instancegroup nodes-us-east-1c contains label kops.k8s.io/lifecycle: OnDemand

    ```bash
    for availability_zone in $(echo ${AWS_REGION_AZS} | sed 's/,/ /g')
    do
    NODEGROUP_NAME=nodes-${availability_zone}
    kops get ig --name ${NAME} ${NODEGROUP_NAME} -o yaml | grep "lifecycle: OnDemand" > /dev/null
    if [ $? -eq 0 ]
    then
        echo "Instancegroup ${NODEGROUP_NAME} contains label kops.k8s.io/lifecycle: OnDemand"
    else
        echo "Instancegroup ${NODEGROUP_NAME} DOES NOT contains label kops.k8s.io/lifecycle: OnDemand"
    fi
    done
    ```

5. Aside from validating that the lifecycle label is set up, you can inspect one of the nodegroup's configuration. Run the following command to view it.

    ```bash
    kops get ig --name ${NAME} nodes-$(echo ${AWS_REGION_AZS}|cut -d, -f 1) -o yaml
    ```

</details>

<br/>

<details>
  <summary>Step 4: Adding Spot workers with kOps toolbox instance-selector</summary>
<br/>

Until January of 2021, to adhere to Spot best practices using kOps, users were required to select a group of Spot instances to diversify manually. They then had to configure a [MixedInstancePolicy InstanceGroup](https://kops.sigs.k8s.io/instance_groups/#mixedinstancepolicy-aws-only) in order to apply diversification within the instance group. In that date, we introduced a new tool: **kOps toolbox instance-selector**. This tool is distributed as part of the standard kOps distribution, and it simplifies the creation of kOps Instance Groups, by creating groups that fully adhere to Spot Instances best practices.

In order to tap into multiple Spot capacity pools, you will create two Instance Groups, each containing multiple instance types. Diversifying into more capacity pools increases the chances of achieving the desired scale, and maintaining it if some of the capacity pools get interrupted (when EC2 needs the capacity back). Each Instance Group ([EC2 Auto Scaling group](https://aws.amazon.com/ec2/autoscaling/)) will launch instances using Spot pools that are optimally chosen based on the available Spot capacity.


1. The following  command creates an Instance Group, which will be called **spot-group-base-4vcpus-16gb**. To create the group, use kOps toolbox instance-selector, which saves the effort of manually configuring the new group for diversification. In this case, use the `--instance-type-base` with m5.xlarge as your base instance, made up of pools from the latest generations. You can get more information about which parameters `kops toolbox instance-selector` uses by running **kops toolbox instance-selsector –-help**

    ```bash
    kops toolbox instance-selector "spot-group-base-4vcpus-16gb" \
    --usage-class spot --cluster-autoscaler \
    --base-instance-type "m5.xlarge" --burst-support=false \
    --deny-list '^?[1-3].*\..*' --gpus 0 \
    --node-count-max 5 --node-count-min 1 \
    --name ${NAME} 
    ```

2. Now let’s create the second Instance Group. This time, you will create the group **spot-group-base-2vcpus-8gb**, using a different approach as in the previous step. Instead of defining a base instance type, now you will specify the amount of memory and vcpus:

    ```bash
    kops toolbox instance-selector "spot-group-base-2vcpus-8gb-2" \
    --usage-class spot --cluster-autoscaler \
    --burst-support=false \
    --vcpus=2  --memory=8GB   \
    --gpus 0 --node-count-max 5 \
    --node-count-min 1 --name ${NAME}
    ```

3. Before proceeding with the final instantiation of the cluster, let’s validate and review the newly created Instance Group's configuration. Run the following command to display the configuration of the **spot-group-base-2vcpus-8gb** Instance Group.

    ```bash
    kops get ig spot-group-base-2vcpus-8gb --name $NAME -o yaml
    ```

4. Your cluster is now configured with all the resources depicted in the architecture diagram below. 

    ![Architecture Diagram](/arch.png)
    <br/>

    However, you have only configured the cluster up to this point. To actually instantiate it, you must execute the following command:

    ```bash
    kops update cluster --state=${KOPS_STATE_STORE} --name=${NAME} --yes --admin 
    ```

   > :warning: If your environment previously had a kubeconfig file, you may need to run `kops export kubeconfig ${NAME} --admin` to store the configuration and change the config. ]

5. The command in the previous step will start requesting for all the cluster resources, and end up with an output similar to the following one. This may take around five minutes.

    ```bash
    Cluster is starting.  It should be ready in a few minutes.

    Suggestions:
    * validate cluster: kops validate cluster --wait 10m
    * list nodes: kubectl get nodes --show-labels
    * ssh to the master: ssh -i ~/.ssh/id_rsa ubuntu@api.spot-kops-cluster.k8s.local
    * the ubuntu user is specific to Ubuntu. If not using Ubuntu please use the appropriate user based on your OS.
    * read about installing addons at: https://kops.sigs.k8s.io/addons.
    ```

6. You can run the **kOps validate cluster** command to evaluate the state of the cluster a few times per minute, capturing the progress of its creation.

    ```bash
    kops validate cluster --wait 10m
    ```

The command will raise errors until the cluster is created.

7. Once the cluster is in a healthy state, you can list nodes to check that the cluster and all its associated resources are up and running.

    ```bash
    kubectl get nodes --show-labels
    ```

</details>

<br/>

<details>
  <summary>Step 5: Deploying the aws-node-termination-handler</summary>
<br/>

When an interruption happens, EC2 sends a [Spot interruption notification](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-interruptions.html) to the instance, giving the application two minutes to gracefully handle that interruption, and minimize the impact to its availability or performance. Also, recently the new [Instance Rebalance Recommendation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/rebalance-recommendations.html) signal was made available, which notifies you when a Spot Instance is at elevated risk of interruption; it can arrive sooner that the Spot interruption notice, giving you extra time to proactively manage the Spot Instance, by rebalancing to new or existing Spot Instances that are not at risk. In order to gracefully handle either scenario on Kubernetes, we will deploy the  [aws-node-termination-handler](https://github.com/aws/aws-node-termination-handler) in this section. 

Let's proceed to installing the [aws-node-termination-handler](https://github.com/aws/aws-node-termination-handler) in Queue Processor mode, with the help of kOps. The Handler will continuously poll an Amazon SQS queue, which receives events emitted by Amazon EventBridge that can lead to the termination of the nodes in your cluster (Spot Interruption/Rebalance events, maintenance events, Auto-Scaling Group lifycle hooks and [more](https://github.com/aws/aws-node-termination-handler#queue-processor)). This enables the Handler to [cordon](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#cordon) and [drain](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#drain) the node - also issuing a SIGTERM to the Pods and containers running on it, in order to achieve a graceful application termination. 

1. kOps facilitates the deployment of the aws-node-termination-handler, allowing you to add its configuration as an addon to the kOps cluster spec. This addon also takes care of deploying all the necessary AWS infrastructure for you: SQS Queue, EventBridge rules, and the necessary Auto-Scaling group Lifecycle hooks. Deploy the aws-node-termination-handler addon with the following command:


    ```bash
    kops get cluster --name ${NAME} -o yaml > ~/environment/cluster_config.yaml 
    cat << EOF > ./node_termination_handler_addon.yaml
    spec:
      nodeTerminationHandler:
        cpuRequest: 200m
        enabled: true
        enableSQSTerminationDraining: true
        managedASGTag: "aws-node-termination-handler/managed"
    EOF
    yq merge --overwrite --inplace ~/environment/cluster_config.yaml ~/environment/node_termination_handler_addon.yaml
    aws s3 cp ~/environment/cluster_config.yaml ${KOPS_STATE_STORE}/${NAME}/config
    kops update cluster --state=${KOPS_STATE_STORE} --name=${NAME} --yes --admin
    ```

2. To check that the aws-node-termination-handler has been deployed successfully, execute the following command.

    ```bash
    kubectl get deployment aws-node-termination-handler -n kube-system -o wide
    ```
</details>
<br/>

<details>
  <summary>Step 6: Deploy the Kubernetes Cluster Autoscaler</summary>
<br/>


[Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler) is a Kubernetes [controller](https://kubernetes.io/docs/concepts/architecture/controller/) that dynamically adjusts the size of the cluster. If there are pods that can't be scheduled in the cluster due to insufficient resources, Cluster Autoscaler will issue a scale-out action. When there are nodes in the cluster that have been under-utilized for a period of time, Cluster Autoscaler will scale-in the cluster. Internally Cluster Autoscaler evaluates a set of **instance groups** to scale up the cluster. When Cluster Autoscaler runs on AWS, **instance groups** are implemented using Auto Scaling Groups. To calculate the number of nodes to scale-out/in when required, Cluster Autoscaler assumes all the instances in an instance group are homogenous (i.e. have the same number of vCPUs and memory size).

Metrics Server is a scalable, efficient source of container resource metrics for Kubernetes built-in autoscaling pipelines. These metrics will drive the scaling behavior of the deployments.

1. Before installing Cluster Autoscaler, install the metric server using a kOps addon:

    ```bash
    kops get cluster --name ${NAME} -o yaml > ~/environment/cluster_config.yaml 
    cat << EOF > ./metric_server_addon.yaml
    spec:
      certManager:
        enabled: true
      metricsServer:
        enabled: true
    EOF
    yq merge --overwrite --inplace ~/environment/cluster_config.yaml ~/environment/metric_server_addon.yaml
    aws s3 cp ~/environment/cluster_config.yaml ${KOPS_STATE_STORE}/${NAME}/config
    kops update cluster --state=${KOPS_STATE_STORE} --name=${NAME} --yes --admin
    ```

2. Wait until the metric server is up and running:

    ```bash
    kubectl get deploy metrics-server -n kube-system
    ```

3. kOps facilitates the deployment of the Cluster Autoscaler, allowing you to add its configuration as an addon to the kOps cluster spec. Deploy the Cluster Autoscaler addon with the following command:

    ```bash
    kops get cluster --name ${NAME} -o yaml > ~/environment/cluster_config.yaml 
    cat << EOF > ./cluster_autoscaler_addon.yaml
    spec:
      clusterAutoscaler:
        awsUseStaticInstanceList: false
        balanceSimilarNodeGroups: false
        cpuRequest: 100m
        enabled: true
        expander: random
        image: us.gcr.io/k8s-artifacts-prod/autoscaling/cluster-autoscaler:v1.23.0
        memoryRequest: 300Mi
        newPodScaleUpDelay: 0s
        scaleDownDelayAfterAdd: 10m0s
        skipNodesWithLocalStorage: true
        skipNodesWithSystemPods: true
    EOF
    yq merge --overwrite --inplace ~/environment/cluster_config.yaml ~/environment/cluster_autoscaler_addon.yaml
    aws s3 cp ~/environment/cluster_config.yaml ${KOPS_STATE_STORE}/${NAME}/config
    kops update cluster --state=${KOPS_STATE_STORE} --name=${NAME} --yes --admin
    ```

![Cluster Autoscaler Arch](/clusterautoscaler.png)

4. You can also check the logs and steps taken by Cluster Autoscaler with the following command. This command will display Cluster Autoscaler logs.

    ```bash
    kubectl logs -f deployment/cluster-autoscaler -n kube-system --tail=10
    ```
</details>

<br/>
<details>
  <summary>Step 7: Deploy a Sample Application</summary>
<br/>

Finally let's deploy a test application and scale our cluster. To scale our application, we will use a Deployment and a Horizontal Pod Autoscaler.

1. Deploy an application and expose as a service on TCP port 80. The application is a custom-built image based on the php-apache image. The index.php page performs calculations to generate CPU load. More information can be found [here](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/#run-expose-php-apache-server)

    ```bash
    kubectl create deployment php-apache --image=us.gcr.io/k8s-artifacts-prod/hpa-example
    kubectl set resources deploy php-apache --requests=cpu=2000
    kubectl expose deploy php-apache --port 80
    kubectl get pod -l app=php-apache
    ```

2. Create an Hpa resource. This HPA scales up when CPU exceeds 50% of the allocated container resource.

    ```bash
    kubectl autoscale deployment php-apache  \
    --cpu-percent=50 \
    --min=1  \
    --max=30 
    ```

3. View the HPA using kubectl. You probably will see `<unknown>/50%` for 1-2 minutes and then you should be able to see `0%/50%`.

    ```bash
    kubectl get hpa
    ```

4. Open a new terminal window and create a new container:

    ```bash
    kubectl run -i --tty load-generator-1 --image=busybox /bin/sh
    ```
5. Execute a while loop to generate load in the application:

    ```bash
    while true; do wget -q -O - http://php-apache; done
    ```

6. In the previous terminal, watch the HPA with the following command:

    ```bash
    kubectl get hpa -w
    ```

You should see your environment scaling. 

</details>

<br/>

<details>
  <summary>Step 8: Clean Up</summary>
<br/>

1. Remove the kOps cluster:

    ```bash
    kops delete cluster --name ${NAME} --yes
    ```

2. In the console, remove the S3 bucket. Read ["Deleting a single object" section](https://docs.aws.amazon.com/AmazonS3/latest/user-guide/delete-objects.html) of the AWS Documentation to find out how to delete a bucket from the console.

</details>


