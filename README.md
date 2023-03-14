# EKS cluster with EBS and EFS and ALB

<!-- TOC -->

- [EKS cluster with EBS and EFS and ALB](#eks-cluster-with-ebs-and-efs-and-alb)
  - [Env vars](#env-vars)
  - [Install CLIs](#install-clis)
    - [AWS CLI](#aws-cli)
    - [EKS CLI](#eks-cli)
  - [Install EKS](#install-eks)
  - [Access to cluster](#access-to-cluster)
    - [For AWS account members](#for-aws-account-members)
    - [For non-members of the AWS account](#for-non-members-of-the-aws-account)
  - [Add EBS CSI driver to the Cluster](#add-ebs-csi-driver-to-the-cluster)
    - [Create an IAM policy and role for EBS](#create-an-iam-policy-and-role-for-ebs)
    - [Creating the Amazon EBS CSI driver IAM role for service accounts](#creating-the-amazon-ebs-csi-driver-iam-role-for-service-accounts)
  - [Add EFS to the Cluster](#add-efs-to-the-cluster)
    - [Enable IAM roles](#enable-iam-roles)
    - [Create an IAM policy and role for EFS](#create-an-iam-policy-and-role-for-efs)
    - [Install the EFS driver](#install-the-efs-driver)
    - [Create an Amazon EFS file system](#create-an-amazon-efs-file-system)
  - [Adding the AWS Load Balancer Controller add-on](#adding-the-aws-load-balancer-controller-add-on)
    - [Validating ALB](#validating-alb)
  - [Create Route53 zone for ADS](#create-route53-zone-for-ads)
  - [Delete the EKS cluster](#delete-the-eks-cluster)

<!-- /TOC -->

Executed from a container image, so it assumes you have a container runtime on your machine, such as Docker or Podman:

I used image `icr.io/continuous-delivery/pipeline/pipeline-base-ubi:3.15` but any other Linux-based distro would work.

## Env vars

```sh
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
export AWS_REGION=

cluster_name=#...
cluster_region=${AWS_REGION}
dns_domain=#...
```

---

## Install CLIs

The installation needs the following CLIs:

- aws
- eksctl
- curl
- git
- kubectl

### AWS CLI

Extracted from <https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html>

```sh
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
./aws/install
```

### EKS CLI

<https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html>

```sh
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
mv /tmp/eksctl /usr/local/bin
eksctl version
```

## Install EKS

Alternative, unexplored, possibility: <https://github.com/aws-samples/amazon-eks-refarch-cloudformation>

Right now, created both VPC and EKS through AWS console:

```sh
eksctl create cluster \
    --region "${cluster_region:?}" \
    --name "${cluster_name:?}" \
    --with-oidc \
    --full-ecr-access \
    --nodegroup-name=workers \
    --node-ami-family AmazonLinux2 \
    --node-type m5.2xlarge \
    --nodes 3

# For autoscaling

#    --nodes-min 2 \
#    --nodes-max 4
```

## Access to cluster

### For AWS account members

```sh
aws eks update-kubeconfig \
    --region "${cluster_region:?}" \
    --name "${cluster_name:?}" \
&& kubectl get namespaces
```

Example output

```txt
NAME              STATUS   AGE
default           Active   20m
kube-node-lease   Active   20m
kube-public       Active   20m
kube-system       Active   20m
```

### For non-members of the AWS account

This process generates a new `kubeconfig` file to be handed to another user.

That other user will need then to export the KUBECONFIG variable to point at this file.

To generate the new configuration file:

```sh
user=cluster-admin
export KUBECONFIG="${HOME}/.kube/config"
aws eks update-kubeconfig \
    --region "${cluster_region:?}" \
    --name "${cluster_name:?}" \
&& kubectl get namespaces

cluster_arn=$(aws eks describe-cluster \
    --region ${cluster_region:?} \
    --name ${cluster_name:?} \
    --query 'cluster.arn' \
    --output text)
cluster_endpoint=$(aws eks describe-cluster \
    --region ${cluster_region:?} \
    --name ${cluster_name:?} \
    --query 'cluster.endpoint' \
    --output text)

kubectl get serviceaccount "${user:?}" 2> /dev/null \
|| kubectl create serviceaccount "${user}"

kubectl get clusterrolebinding "${user}-binding" 2> /dev/null \
|| kubectl create clusterrolebinding "${user}-binding" \
  --clusterrole "${user}" \
  --serviceaccount default:cluster-admin

kube_token=$(kubectl create token "${user}" --duration=100h)

export KUBECONFIG="${HOME}/kubeconfig-${cluster_name:?}"
rm -rf "${KUBECONFIG:?}"

kubectl config set-cluster "${cluster_arn}" \
    --server "${cluster_endpoint}" \
    --insecure-skip-tls-verify=true

kubectl config set-context "${cluster_arn}" \
    --cluster  "${cluster_arn}" \
    --user cluster-admin

kubectl config use-context "${cluster_arn}"

kubectl config set-credentials cluster-admin --token="${kube_token}"
```

Now share the file in $KUBECONFIG with the user. Keep in mind that this file contains an administrative bearer token to the cluster, so handle it with the same care as you would handle the sharing of any other credential or password.

The user, after making a local copy of the file, should be able to export their local `KUBECONFIG` environment variable to match the full pathname of the file and use the `kubectl` CLI to interact with the cluster.

## Add EBS CSI driver to the Cluster

References:

- <https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html>
- <https://docs.aws.amazon.com/eks/latest/userguide/storage-classes.html>
- <https://docs.aws.amazon.com/cli/latest/reference/eks/create-addon.html>

### Create an IAM policy and role for EBS

Extracted from: <https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html>

The cluster needs an OIDC provider. Note that when the cluster is created with the `--with-oidc`, the block below is redundant.

```sh
oidc_id=$(aws eks describe-cluster \
        --region "${cluster_region}" \
        --name "${cluster_name}" \
        --query "cluster.identity.oidc.issuer" \
        --output text | cut -d '/' -f 5) \
oidc_provider=$(aws iam list-open-id-connect-providers \
    | grep $oidc_id \
    | cut -d "/" -f4)
if [ -z ${oidc_provider} ]; then
    eksctl utils associate-iam-oidc-provider \
        --region "${cluster_region}" \
        --cluster "${cluster_name}" \
        --approve
    oidc_provider=$(aws iam list-open-id-connect-providers \
        --region "${cluster_region}" \
        | grep $oidc_id | cut -d "/" -f4)
    echo "OIDC Provider ${oidc_provider}"
fi
```

### Creating the Amazon EBS CSI driver IAM role for service accounts

Extracted from <https://docs.aws.amazon.com/eks/latest/userguide/csi-iam-role.html>.

```sh
role_name=AmazonEKS_EBS_CSI_DriverRole
eksctl create iamserviceaccount \
    --region "${cluster_region}" \
    --cluster "${cluster_name}" \
    --name ebs-csi-controller-sa \
    --namespace kube-system \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
    --approve \
    --role-only \
    --role-name "${role_name}" \
    --override-existing-serviceaccounts

role_arn=$(aws iam get-role \
    --region "${cluster_region}" \
    --role-name "${role_name}" \
    --query 'Role.Arn' \
    --output text)

aws eks create-addon \
    --region "${cluster_region}" \
    --cluster-name "${cluster_name}" \
    --addon-name aws-ebs-csi-driver \
    --addon-version v1.16.1-eksbuild.1 \
    --service-account-role-arn "${role_arn}" \
&& aws eks wait addon-active \
    --region "${cluster_region}" \
    --cluster-name "${cluster_name}" \
    --addon-name aws-ebs-csi-driver \
&& aws eks describe-addon \
    --region "${cluster_region}" \
    --cluster-name "${cluster_name}" \
    --addon-name aws-ebs-csi-driver 
```

Deploy a sample application and verify that the CSI driver is working

```sh
cd /tmp
rm -rf aws-ebs-csi-driver

git clone https://github.com/kubernetes-sigs/aws-ebs-csi-driver.git
cd aws-ebs-csi-driver/examples/kubernetes/dynamic-provisioning/

kubectl apply -f manifests/
kubectl describe storageclass ebs-sc
kubectl get pv
```

## Add EFS to the Cluster

Extracted from <https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html>

### Enable IAM roles

Extracted from <https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html>.

The cluster needs an OIDC provider. Note that when the cluster is created with the `--with-oidc`, the block below is redundant.

```sh
oidc_id=$(aws eks describe-cluster \
    --region "${cluster_region}" \
    --name "${cluster_name}" \
    --query "cluster.identity.oidc.issuer" \
    --output text | cut -d '/' -f 5)
oidc_provider=$(aws iam list-open-id-connect-providers \
    --region "${cluster_region}" \
    | grep $oidc_id | cut -d "/" -f4)
if [ -z ${oidc_provider} ]; then
    eksctl utils associate-iam-oidc-provider \
        --region "${cluster_region}" \
       --cluster "${cluster_name}" \
       --approve
    oidc_provider=$(aws iam list-open-id-connect-providers \
        --region "${cluster_region}" \
        | grep $oidc_id | cut -d "/" -f4)
    echo "OIDC Provider ${oidc_provider}"
fi
```

### Create an IAM policy and role for EFS

```sh
cd /tmp

curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/docs/iam-policy-example.json

efs_policy_name=AmazonEKS_EFS_CSI_Driver_Policy
policy_arn=$(aws iam list-policies \
    --scope Local \
    --query "Policies[?PolicyName=='${efs_policy_name}'].Arn" \
    --output text)

if [ -z "${efs_policy_name}" ]; then
    aws iam create-policy \
        --policy-name "${efs_policy_name:?}" \
        --policy-document file://iam-policy-example.json
    policy_arn=$(aws iam list-policies \
        --scope Local \
        --query "Policies[?PolicyName=='${efs_policy_name}'].Arn" \
        --output text)
fi

eksctl create iamserviceaccount \
    --region "${cluster_region}" \
    --cluster "${cluster_name}" \
    --namespace kube-system \
    --name efs-csi-controller-sa \
    --attach-policy-arn "${policy_arn:?}" \
    --approve
```

### Install the EFS driver

Need to get this for the specific cloud region from <https://docs.aws.amazon.com/eks/latest/userguide/add-ons-images.html>.

```sh
# this is the id for the us-east-2 region
image_repository_id=602401143452

helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
helm repo update
helm upgrade -i aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
    --namespace kube-system \
    --set image.repository=${image_repository_id:?}.dkr.ecr.${cluster_region:?}.amazonaws.com/eks/aws-efs-csi-driver \
    --set controller.serviceAccount.create=false \
    --set controller.serviceAccount.name=efs-csi-controller-sa
kubectl get pod -n kube-system -l "app.kubernetes.io/name=aws-efs-csi-driver,app.kubernetes.io/instance=aws-efs-csi-driver"
```

### Create an Amazon EFS file system

```sh
vpc_id=$(aws eks describe-cluster \
    --region "${cluster_region:?}" \
    --name "${cluster_name:?}" \
    --query "cluster.resourcesVpcConfig.vpcId" \
    --output text)
cidr_range=$(aws ec2 describe-vpcs \
    --region "${cluster_region}" \
    --vpc-ids "${vpc_id:?}" \
    --query "Vpcs[].CidrBlock" \
    --output text)
security_group_id=$(aws ec2 create-security-group \
    --region "${cluster_region}" \
    --vpc-id "${vpc_id:?}" \
    --group-name MyEfsSecurityGroup \
    --description "My EFS security group" \
    --output text)
aws ec2 authorize-security-group-ingress \
    --region "${cluster_region}" \
    --group-id "${security_group_id:?}" \
    --protocol tcp \
    --port 2049 \
    --cidr "${cidr_range:?}" \
local filesystem_name="aws-fs-${cluster_name}" \
file_system_id=$(aws efs create-file-system \
    --region "${cluster_region}" \
    --creation-token "${filesystem_name}" \
    --tags Key=Name,Value="${filesystem_name}"
    --performance-mode generalPurpose \
    --query 'FileSystemId' \
    --output text)

aws ec2 describe-subnets \
    --region "${cluster_region}" \
    --filters "Name=vpc-id,Values=${vpc_id}" \
    --query 'Subnets[*].SubnetId' \
    --output text | tr -s "\\t" "\\n" \
| xargs -I {} aws efs create-mount-target \
    --region "${cluster_region}" \
    --file-system-id "${file_system_id:?}" \
    --subnet-id {} \
    --security-groups "${security_group_id}"

curl -sL https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/examples/kubernetes/dynamic_provisioning/specs/storageclass.yaml \
| sed "s/fileSystemId:.*/fileSystemId: ${file_system_id:?}/" \
| kubectl apply -f -
```

## Adding the AWS Load Balancer Controller add-on

Extracted from
<https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html>

```sh
alb_policy_name=AWSLoadBalancerControllerIAMPolicy
alb_policy_arn=$(aws iam list-policies \
    --scope Local \
    --query "Policies[?PolicyName=='${alb_policy_name}'].Arn" \
    --output text)

if [ -z "${alb_policy_arn}" ]; then
    curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.7/docs/install/iam_policy.json
    aws iam create-policy \
        --policy-name "${alb_policy_name}" \
        --policy-document file://iam_policy.json
    alb_policy_arn=$(aws iam list-policies \
        --scope Local \
        --query "Policies[?PolicyName=='${alb_policy_name:?}'].Arn" \
        --output text)
fi

eksctl create iamserviceaccount \
    --region "${cluster_region:?}" \
    --cluster "${cluster_name:?}" \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --role-name AmazonEKSLoadBalancerControllerRole \
    --attach-policy-arn="${alb_policy_arn:?}" \
    --approve

helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
    -n kube-system \
    --set clusterName="${cluster_name:?}" \
    --set serviceAccount.create=false \
    --set serviceAccount.name=aws-load-balancer-controller 
    
kubectl get deployment -n kube-system aws-load-balancer-controller
```

### Validating ALB

Extracted from <https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html>.

```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.7/docs/examples/2048/2048_full.yaml

sleep 60

kubectl get ingress/ingress-2048 -n game-2048
```

## Create Route53 zone for ADS

```sh
zone_id=$(aws route53 list-hosted-zones \
    --query "HostedZones[?Name==\`${dns_domain:?}.\`].Id" \
    --output text \
| cut -d "/" -f 3)

if [ -z "${zone_id}" ]; then
    caller_reference="$(date +%s)"
    aws route53 create-hosted-zone \
        --name "${dns_domain:?}" \
        --caller-reference "${caller_reference}" \
        --hosted-zone-config Comment="Public hosted zone for ADS installation" 

    zone_id=$(aws route53 list-hosted-zones \
        --query "HostedZones[?Name==\`${dns_domain}.\`].Id" \
        --output text \
    | cut -d "/" -f 3)
fi

```

## Delete the EKS cluster

< under construction, needs to remove EFS resources and EBS from AWS account, but still not set on the list/cmds >

```sh
aws efs describe-mount-targets \
    --region "${cluster_region}" \
    --query 'MountTargets[*].MountTargetId' \
    --file-system-id ${file_system_id} \
    --output text | tr -s "\\t" "\\n" \
| xargs -I {} aws efs delete-mount-target \
    --region "${cluster_region}" \
    --mount-target-id {}

aws efs delete-file-system \
    --region "${cluster_region}" \
    --file-system-id ${file_system_id}

eksctl delete cluster \
    --region "${cluster_region:?}" \
    --name "${cluster_name:?}"

aws elbv2 describe-load-balancers \
    --region "${cluster_region}" \
    --query 'LoadBalancers[?Tags[?Key==`elbv2.k8s.aws/cluster` && Value==`${cluster_name}`]].LoadBalancerArn' \
    --output text \
| xargs -I {} aws elbv2 delete-load-balancer --load-balancer-arn {}

```
