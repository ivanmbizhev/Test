How to build your own EKS with Prometheus/Grafana

This is a small guide on how to build your own EKS with Grafana monitoring.
Pre-requisites:  The following accesses and credentials are needed, if you don’t have them you will need to raise a ticket with RnD
Mandatory: AWS Identity and Access Management (IAM) OpenID Connect (OIDC) 
The following list will be needed as well:
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "iam:UpdateAssumeRolePolicy",
                "iam:ListRoleTags",
                "iam:ListServerCertificates",
                "iam:CreateRole",
                "iam:AttachRolePolicy",
                "iam:ListServiceSpecificCredentials",
                "iam:PutRolePolicy",
                "iam:ListSigningCertificates",
                "iam:ListVirtualMFADevices",
                "iam:AddRoleToInstanceProfile",
                "iam:ListSSHPublicKeys",
                "iam:DetachRolePolicy",
                "iam:ListAttachedRolePolicies",
                "iam:ListOpenIDConnectProviderTags",
                "iam:ListSAMLProviderTags",
                "iam:ListRolePolicies",
                "iam:ListPolicies",
                "iam:GetRole",
                "iam:ListSAMLProviders",
                "iam:GetPolicy",
                "iam:ListEntitiesForPolicy",
                "iam:DeleteRole",
                "iam:UpdateRoleDescription",
                "iam:ListGroupsForUser",
                "iam:GetRolePolicy",
                "iam:GetAccountSummary",
                "iam:UntagRole",
                "iam:PutRolePermissionsBoundary",
                "iam:TagRole",
                "iam:ListPoliciesGrantingServiceAccess",
                "iam:DeletePolicy",
                "iam:ListInstanceProfileTags",
                "iam:ListMFADevices",
                "iam:GetServiceLinkedRoleDeletionStatus",
                "iam:ListInstanceProfilesForRole",
                "iam:PassRole",
                "iam:ListAttachedUserPolicies",
                "iam:ListAttachedGroupPolicies",
                "iam:ListPolicyTags",
                "iam:ListAccessKeys",
                "iam:ListGroupPolicies",
                "iam:ListRoles",
                "iam:ListUserPolicies",
                "iam:ListInstanceProfiles",
                "iam:CreatePolicy",
                "iam:CreateServiceLinkedRole",
                "iam:ListPolicyVersions",
                "iam:ListOpenIDConnectProviders",
                "iam:ListServerCertificateTags",
                "iam:ListAccountAliases",
                "iam:ListUsers",
                "iam:UpdateRole",
                "iam:ListGroups",
                "iam:ListMFADeviceTags",
                "iam:GetLoginProfile",
                "iam:ListUserTags"
            ],
            "Resource": "*"
        }
    ]
}

Read/Write:
OpenID Connect (OIDC)
iam:DeleteRolePolicy
iam:ReadRolePolicy
iam:CreateRolePolicy
iam:DetachRolePolicy

For more information please visit: 
https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html

Before we begin, I used several of these guides to launch my cluster:
https://www.nclouds.com/blog/amazon-eks-grafana-prometheus/
https://dev.to/aws-builders/monitoring-eks-cluster-with-prometheus-and-grafana-1kpb
https://www.stacksimplify.com/aws-eks/kubernetes-storage/install-aws-ebs-csi-driver-on-aws-eks-for-persistent-storage/
But my recommendation is to follow the AWS documentations as much as possible.

1.	 Create Cluster
eksctl create cluster \
  --region eu-central-1 \
  --node-type t2.small \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 2 \
  --name {name_your_cluster} \
  
2.	Create IAM role
 
  eksctl create iamserviceaccount \
  --name {name_your_account} \
  --namespace kube-system \
  --cluster {cluster_name} \
  --role-name "AmazonEKSVPCCNIRole" \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy \
  --override-existing-serviceaccounts \
  --approve
 
  
  
3.	EBS will be needed so that it can manage the lifecycle of Amazon EBS volumes for persistent volumes: 
https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html
https://kubernetes.io/docs/concepts/storage/persistent-volumes/

helm upgrade --install aws-ebs-csi-driver \
  --version=1.2.4 \
  --namespace kube-system \
  --set serviceAccount.controller.create=false \
  --set serviceAccount.snapshot.create=false \
  --set enableVolumeScheduling=true \
  --set enableVolumeResizing=true \
  --set enableVolumeSnapshot=true \
  --set serviceAccount.snapshot.name=ebs-csi-controller-irsa \
  --set serviceAccount.controller.name=ebs-csi-controller-irsa \
  aws-ebs-csi-driver/aws-ebs-csi-driver
  
  eksctl create addon --name aws-ebs-csi-driver --cluster {name} --service-account-role-arn arn:aws:iam::<id>:role/AmazonEKS_EBS_CSI_DriverRole --force

eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster ibizhev-eks \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --role-only \
  --role-name AmazonEKS_EBS_CSI_DriverRole


4. Create Prometheus

The best possible way to do is to use the document below:
https://docs.aws.amazon.com/eks/latest/userguide/prometheus.html
  
5. Create Grafana

cat << EoF > grafana.yaml
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus-server.prometheus.svc.cluster.local
      access: proxy
      isDefault: true
EoF

helm install grafana grafana/grafana \
    --namespace grafana \
    --set persistence.storageClassName="gp2" \
    --set persistence.enabled=true \
    --set adminPassword='EKS!sAWSome' \
    --values grafana.yaml \
    --set service.type=LoadBalancer  




Please take a look at the below document as well its regarding network LBs
https://docs.aws.amazon.com/eks/latest/userguide/network-load-balancing.html
The above installation uses the standard LBs but if you would like to use ALB:
https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html
https://github.com/aws/eks-charts/tree/master/stable/aws-load-balancer-controller

aws iam create-policy \
    --policy-name IB_AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam-policy.json
	
eksctl create iamserviceaccount \
  --cluster={cluster_name} \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name "IB_AmazonEKSLoadBalancerControllerRole" \
  --attach-policy-arn=arn:aws:iam::<id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
  
  
  helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName={cluster_name} \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
 helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName={cluster_name} --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller

helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName={name}
