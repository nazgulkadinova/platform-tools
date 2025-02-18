C L U S T E R   A U T O   S C A L E R

About AWS EKS Cluster-Autoscaler: 

The all point of the cloud and k8s is to have an ability to scale. We want to be able to add new nodes as a existing to become full. and as a demands drop we want to be able to delete this nodes and scale them down.
The AWS EKS (Elastic Kubernetes Service) Cluster Autoscaler is a tool designed to automatically adjust the size of your EKS cluster by adjusting the number of Worker nodes based on the resource requirements of your running pods. This helps optimize the utilization of resources and ensures that your applications have the necessary resources to run efficiently.
*Here we are going to use managed node group and unmanaged node group. Autoscaler can work with both of them, but AWS recommend to work with managed node group(because it can gracefuly drain the node and reshedule those pods in different nodes before remove this node from the cluster), while unmanaged node group will simply derminate the node. By testing with both managed and unmanaged node groups, you can ensure that the Cluster Autoscaler is functioning correctly with different node provisioning methods and configurations.

* Components needed for Autoscaling in Kubernetes:
    * Service Account: Grants necessary permissions for interacting with other services in the environment.
    * ClusterRole and Role: Define the permissions needed for the Cluster Autoscaler at the cluster and namespace levels.
    * ClusterRoleBinding and RoleBinding: Bind the ServiceAccount to the ClusterRole or Role to grant the necessary permissions.
    * Deployment: Deploys the Cluster Autoscaler, which is responsible for adjusting the number of nodes in the cluster based on resource requirements.

Nessasary steps before start:
- Two labels must present in Auto Scaling Groups ( go to AWS - Auto Scaling Groups - choose your auto scaling - scrowdown to Tags) :
Key:                                                Value:
k8s.io/cluster-autoscaler/enabled                   true
k8s.io/cluster/your_cluster_name                    owned

Reasons: 
k8s.io/cluster-autoscaler/enabled: 
This label indicates whether the Cluster Autoscaler is enabled for the particular Kubernetes cluster. When is set to "true," it means the Cluster Autoscaler is active and doing its job for this group of nodes.

k8s.io/cluster/your_cluster_name: 
This label represent the Kubernetes cluster's identifier. In Kubernetes environments managed on AWS using EKS , each cluster is assigned a unique identifier, so, your_cluster_name appears to be the specific identifier for the Kubernetes cluster associated with this Auto Scaling Group.

These labels are important for managing and automating Kubernetes cluster resources, especially in dynamic environments where the cluster size needs to scale based on workload demands. By setting these labels appropriately, we can ensure that the Kubernetes cluster scales effectively and that resources are tagged for easy identification and management within the AWS ecosystem.
    * If the Cluster was already created (our case), then we want to ensure that these tags won't be deleted or lost, we can utilize AWS IAM policies to enforce tag compliance. To achieve this, we need to create an IAM policy that specifies the required tags and their values for the resources and attach this IAM Policy to IAM Users/Roles/Groups.
    Example of the policy in file : iam-policy-eks-cluster-tag.json , where <your_cluster_name> is your name of your cluster, and need to replace with your cluster name. 



* Make sure that EKS cluster's instance groups has different Min and Max capacity,  if set for example  Min to 1 and Max to 1, Auto Scaler won't be able to scale down or up nodes.  

- Identity provider must be created:

Copy the cluster's OpenID Connect provider URL ( go to EKS - Overview ), then navigate to IAM - Identity providers - Add provider - choose OpenID Connect - insert OpenID Connect provider URL to Provider URL, in Audience type: sts.amazonaws.com. 

Reason: Identity providers are essential for the EKS Auto Scaler because they determine who or what has permission to make changes to your EKS cluster.


- Policy for Auto Scaler must be created:

Go to IAM - Policies - create Policy - choose JSON policy - insert from iam-policy.json.
 This policy allows the specified actions to be performed on all resources related to Auto Scaling Groups and EC2 instances within the AWS account. It grants broad permissions for managing and inspecting these resources. 
 For example:
    autoscaling:DescribeTags:
         Allows describing tags associated with Auto Scaling Groups and EC2 instances.

    autoscaling:SetDesiredCapacity: 
        Allows setting the desired number of instances for an Auto Scaling Group.

    autoscaling:TerminateInstanceInAutoScalingGroup: 
        Allows terminating instances within an Auto Scaling Group.

    ec2:DescribeLaunchTemplateVersions: 
        Allows describing versions of EC2 Launch Templates, which are used to launch EC2 instances with specific configurations.

    Resource: 
        Specifies the AWS resources to which the actions apply. In this policy, * is used as a wildcard, which means the actions are allowed for all resources.

Reason: 
creating IAM policies is essential for defining the specific permissions granted to users, roles, or services interacting with our EKS cluster and its resources, including the Cluster Autoscaler.

- Role must be created:

Go to IAM - Roles - create Role - choose Web identity - select the Identity provider you created - in Audience type: sts.amazonaws.com - atach the Policy for Auto Scaler you created. 

Reason:
Roles in AWS IAM serve a crucial role in granting permissions to entities such as AWS services, applications, or AWS users.

* 2 things in IAM role after creating must be ajusted :
    IAM Role:
    go to IAM - Roles - choose your role - Trust relationships - Edit trust policy - in section "StringEquals" replace "aud" to "sub" ( for example if role is oidc.eks.us-east-1.amazonaws.com/id/E32A80A0000DF0D0AE00000BC00E00C00F:aud": "sts.amazon.com", should be oidc.eks.us-east-1.amazonaws.com/id/E32A80A0000DF0D0AE00000BC00E00C00F:sub":"sts.amazon.com"), 
    also instead of "sts.amazon.com" type "system:serviceaccount:kube-system:cluster-autoscaler" where cluster-autoscaler is your name of your service account. 
    So final version based on examples must be: "oidc.eks.us-east-1.amazonaws.com/id/E32A80A554DF4D0AE7616BC99E33C26F:sub": "system:serviceaccount:kube-system:cluster-autoscaler"

Now we ready to deploy the Autoscaler :-)

When you creating Service Account and Deployment of Claster Auto Scaler update this parameters: 

In Service Account : in Annotations:Eks.amazonaws.com/role-arn  insert your arn role from: - go to IAM - Roles - choose the role you created.

In Deployment of Claster Auto Scaler:  in Spec:Containers:image version must be match with your version of your EKS cluster.
   also in Spec:Containers:Command - command --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/eks-cluster-dev  - where eks-cluster-dev is name of your EKS cluster.

Rest of resources can stay the same. 



## Issues
[Document any issues faced while working on this project]

* Images for cluster-autoscaler deployment can be found at : https://github.com/kubernetes/autoscaler/releases

* Cluster version must match with autoscaler image. for example if version of cluster is 1.28, in cluster-autoscaler spec:containers:image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.28.0.

* Working set of configuration for dev-values.yaml should be in key-value pairs in plain text format, not the key-value pairs.
Example of key-value pairs in plain text format:

service_account_k8s_addon: cluster-autoscaler.addons.k8s.io and etc.

Non-working Configuration (key-value pairs): 
service:
    k8s-addon:cluster-autoscaler.addons.k8s.io



## How to Run/Execute
[Instructions on how to deploy and manage the Cluster Autoscaler]

## Resources
[List any useful resources such as articles, documentation, or courses]

## Additional Information
[Any other relevant information]