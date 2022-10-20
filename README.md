# Using Crossplane on EKS to enforce governance policies in ECS workloads

Workloads utilizing the Kubernetes orchestrator can take advantage of ecosystem technologies that provide granular governance through policy definition and enforcement, such as with the Open Policy Agent (OPA) project. Users of other orchestration systems, such as the Amazon Elastic Container Service (ECS), do not have analagous capabilities, presenting a challenge to the establishment of common governance policies between disparate container orchestrators within the same organizaiton.

This repository provides an examlpe of using Crossplane, a technology commonly used to provision cloud resources through the Kubernetes API, in conjunction with OPA to provide policy enforcement for ECS-based workloads.

### The sample objective

ECS Services can be configured to run atop self-managed EC2 nodes, or in an entirely serverless fashion through the use of AWS Fargate. Fargate provides numerous configurations of vCPU and memory resources ranging of 0.25 of a single vCPU up to 16 vCPUs.

In this scenario we would like to block users from provisioning tasks with 16 vCPUs, as while such tasks are useful in workloads necessitating a significant quantity of resources, they are also overkill for many workloads with lesser resource requirements and can have a negative impact to expenses.

We will setup a system where the request for an ECS Service with 16 vCPUs is generated through Crossplane running on an EKS cluster, but OPA detects and blocks the request resulting in the user being unable to provision the costly resource.

## Steps

### 1. Provision an EKS cluster with Crosssplane

The EKS Blueprints project uses Terraform to quickly scaffold out an EKS cluster aligned with best practices.

The project provides numerous examples of commmon workloads, including Crossplane. [Run the sample](https://github.com/aws-ia/terraform-aws-eks-blueprints/tree/main/examples/crossplane) to easily setup Crossplane.

The default IAM setup for the example only provides permissions to interact with S3. For our example, update the main.tf file's `crossplane_aws_provider` portion with

```
additional_irsa_policies = [
  "arn:aws:iam::aws:policy/AmazonS3FullAccess", "arn:aws:iam::aws:policy/AmazonEC2FullAccess", "arn:aws:iam::aws:policy/AmazonVPCFullAccess", "arn:aws:iam::aws:policy/AmazonECS_FullAccess"
]`
```

### 2. Setup an ECS Cluster

Now we are going to setup a variety of AWS services through Crossplane. Monitoring what is going on can be done by observing the logs from the Crossplane Provider pod running EKS. To determine this pod name and setup logs for viewing:

```shell
$ kubectl get pods --namespace crossplane-system

I1020 08:27:06.443384   79705 request.go:621] Throttling request took 1.188257852s, request: GET:https://843C388485C1A9DFC5B7E96F6E82C9E8.gr7.us-west-2.eks.amazonaws.com/apis/neptune.aws.crossplane.io/v1alpha1?timeout=32s

NAME                                             READY   STATUS    RESTARTS        AGE
aws-provider-245ce7fb587d-5c5767474f-mmzvk       1/1     Running   1 (7h24m ago)   20h
crossplane-5696d784b8-tfdzs                      1/1     Running   1 (7h24m ago)   25h
crossplane-rbac-manager-8466dfb7fc-lmzbz         1/1     Running   1 (7h24m ago)   25h
jet-aws-provider-b42c59584ad8-6b4c7685dc-7gmvt   1/1     Running   1 (7h24m ago)   25h

# Look for the aws-provider-* and follow logs

$ kubectl logs aws-provider-245ce7fb587d-5c5767474f-mmzvk --namespace crossplane-system --follow
```

1. Setup the networking stack (VPC, Subnets, NAT Gateway, Internet Gateway, Route Tables, Security Group)

```shell
kubectl apply --filename ./ecs/0_networking.yaml
```

2. Create an ECS Cluster

```shell
kubectl apply --filename ./ecs/1_ecs_cluster.yaml
```

3. Create an ECS Task Definition. For a sample application we're using a vanilla Nginx web server [from ECR Public](https://gallery.ecr.aws/nginx/nginx) with 8 vCPU and 32 GB of memory.

```shell
kubectl apply --filename ./ecs/2_ecs_task_definition.yaml
```

4. Create an ECS Service with 2 replicas frontend by an Application Load Balancer (ALB).

```shell
kubectl apply --filename ./ecs/3_ecs_service.yaml
```

Ta-da! We have a working web application! To view the application in a browser window, grab the URL from the AWS Console or query via Crossplane:

```shell
$ kubectl get loadbalancer.elbv2.aws.crossplane.io/cp-alb -o jsonpath='{.status.atProvider.dnsName}'

cp-alb-992262513.us-west-2.elb.amazonaws.com
```

### 3. Setup Open Policy Agent's Gatekeeper

1. [Install Helm](https://helm.sh/docs/intro/install/)

2. [Install Gatekeeper via Helm](https://open-policy-agent.github.io/gatekeeper/website/docs/install/#deploying-via-helm)

```shell
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts

helm install gatekeeper/gatekeeper \
  --name-template=gatekeeper \
  --namespace gatekeeper-system \
  --create-namespace
```

3. Create an OPA ConstraintTemplate to describe both the Rego that enforces the constraint and the schema of the constraint.

```yaml
# ./opa/ecsblocklarge_template.yaml

apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: ecsblocklarge
spec:
  crd:
    spec:
      names:
        kind: EcsBlockLarge
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package ecsblocklarge

        contains_16(cpu) = true {
          cpu == "16384"
        }

        violation[{"msg": msg}] {
          input.review.kind.kind == "TaskDefinition"
          cpu := input.review.object.spec.forProvider.cpu
          contains_16(cpu)
          msg := "User is not allowed to create services using 16 vCPUs. Please shrink."
        }
```

```shell
$ kubectl apply --filename ./opa/ecsblocklarge_template.yaml
```

4. Create an OPA Constraint that is used to inform Gatekeeper that we want our ConstraintTemplate to be enforced.

```yaml
# ./opa/all_ecs_smaller_than_16.yaml

apiVersion: constraints.gatekeeper.sh/v1beta1
kind: EcsBlockLarge
metadata:
  name: svc-sizer
spec:
  match:
    kinds:
      - apiGroups: ['ecs.aws.crossplane.io']
        kinds: ['TaskDefinition']
```

```shell
$ kubectl apply --filename ./opa/all_ecs_smaller_than_16.yaml
```

### 4. Deploy an ECS Task Definition

Gatekeeper is now on the lookout for `TaskDefinition` objects that are asking for 16 vCPUs, upon which it will block the attempt.

If we re-deploy our redeploy our original ECS Task Definition, which uses 8 vCPU and 32 GB of memory, we see there is nothing to change and the API accepts the request.

```shell
$ kubectl apply --filename ./ecs/2_ecs_task_definition.yaml

taskdefinition.ecs.aws.crossplane.io/cp-nginx unchanged
```

So Gatekeeper is fine with an 8 vCPU Task Definition. Let's reset with a quick delete.

```shell
$ kubectl delete --filename ./ecs/2_ecs_task_definition.yaml

taskdefinition.ecs.aws.crossplane.io "cp-nginx" deleted
```

Now we will deploy an updated Task Definition, this time using 16 vCPU which is against our policy.

```shell
$ kubectl apply --filename ./ecs/4_ecs_task_definition_16.yaml

Error from server (Forbidden): error when creating "./ecs/4_ecs_task_definition_16.yaml": admission webhook "validation.gatekeeper.sh" denied the request: [svc-sizer] User is not allowed to create services using 16 vCPUs. Please shrink.
```

Gatekeeper steps in and blocks the request, providing the error text that we defined in our ConstraintTemplate.

## Conclusion

In this sample we deployed an EKS cluster pre-configured with Crossplane through the EKS Blueprints project. We then setup an ECS environment and deployed a web application. To define and enforce policies related to such a web application we then installed the OPA Gatekeeper.

ECS does not have a native policy feature outside of IAM, which provides a rough-hewn boundary of if a user can utilize the `register-task-definition` APi route. Gatekeeper and Crossplane provide us a way to deploy an ECS service with granular policy enforcement, in this case detecting when a user attempts to register a task definition that includes the out of policy 16 vCPU resource configuration.

This pattern could be used to provide similar granular policy definition and enforcement of ECS Services, utilizing an EKS ecosystem technology within ECS operations.

## Clean Up

```shell

# OPA
kubectl delete --filename ./opa/all_ecs_smaller_than_16.yaml
kubectl delete --filename ./opa/ecsblocklarge_template.yaml

# ECS
kubectl delete --filename ./ecs/3_ecs_service.yaml
kubectl delete --filename ./ecs/1_ecs_cluster.yaml
kubectl delete --filename ./ecs/0_networking.yaml

# EKS Cluster
terraform destroy -target="module.eks_blueprints_kubernetes_addons" -auto-approve
terraform destroy -target="module.eks_blueprints" -auto-approve
terraform destroy -target="module.vpc" -auto-approve
```
