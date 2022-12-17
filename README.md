# gd5
## overview
eks cluster with public facing ssl using Lets Encrypt
vpc
eks cluster
eks user/roles
config map
IAM role
cluster autoscaling
aws lb controller using Helm
test ingress

## authentication
> Configuration for the AWS Provider can be derived from several sources, which are applied in the following order:
> **note** *This order matches the precedence used by the AWS CLI and the AWS SDKs*
- Parameters in the provider configuration
- Environment variables
- Shared credentials files
- Shared configuration files
- Container credentials
- Instance profile credentials and region

### advanced authentication methods
- The AWS Provider supports assuming an IAM role, either in the provider configuration block parameter assume_role or in a named profile.

- The AWS Provider supports assuming an IAM role using web identity federation and [OpenID Connect (OIDC)](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-role.html#cli-configure-role-oidc). 
  + This can be configured either using environment variables or in a named profile.
  +  When using a named profile, the AWS Provider also supports sourcing credentials from an external process.

- Shared Configuration and Credentials Files
  + By default, these files are located at `$HOME/.aws/config` and `$HOME/.aws/credentials`
- If no named profile is specified, the `default` profile is used. Use the profile parameter or AWS_PROFILE environment variable to specify a named profile

#### AWS CLI from scratch (MAC)
##### installation
- homebrew
- /opt/homebrew/bin/aws
#####  configuration

1. create a non-root user and provide policies for day-to-day administration
  + you can likely use some built-in AWS policies to get started
  + for example "AdministratorAccess"
2. create access keys (as root account) for the user, in the AWS Console
  + download CVS file or copy keys from UI
3. cd ~
  - by default you will not have a .aws directory in the $HOME directory
4. `aws configure`
  + AWS access key ID
  + AWS secret access key
  + default region
5. after the `aws configure` command is run you will see the "config" and the "credentials" files
6. **How does Terraform know to use these credentials when you run Terraform from your local account?**
  + if no other credentials sources are configured (eg env variables, file paths in TF etc), AWS Provider will search for these credential files in $HOME/.aws
7. after completing steps 1-6 you should be able to do do `aws cli` commands as well as `terraform init` and `terraform plan`

## Design Notes
### VPC
- need two subnets for EKS cluster
- generally you deploy the k8s nodes in the public subnets with a default route to the NAT gateway
### EKS
- to grant access to applications running in your EKS cluster, you can  attach an IAM Role with the required IAM  policies to the nodes
- the more secure option is to use IAM Roles for Service Accounts (IRSA)
#### IRSA
- IRSA allows you to limit the IAM role to a single pod
#### Nodes
- to run the workload(s) on your EKS cluster, you need Instance Groups. There are three basic options
  + EKS managed nodes
#### IAM and access to EKS workloads
- granting access to EKS for other IAM users and other IAM roles
- access to EKS is managed by using an EKS auth config map in the `kubesystem` namespace
- initally only the user that created the cluster can access and modfiy that config map
- to grant access to EKS to members of a team you can:
  + add IAM users directly to the auth config map --this will require updating the config map each time a new access (user) is desired
  + a better approach is to grant access (just once) to an IAM role using the aws auth config map, then users outside of EKS can be allowed to assume that role
    - since IAM groups are not known in EKS, this is the preferred option
    - create an IAM role with the necessary permissions and allow an IAM user(s) to assume that role
- the "3-iam.tf" module creates a test user ("user-1") that gets access to EKS
  + the test user is allowed to assume the IAM role for eks managment ("eks_admins_iam_role")
  + the test user is put into a group ("eks_admins_iam_group") and the group is given the abiltiy to assume the eks admin role (`custom_group_policy_arns`)
- to add the eks iam admin role to the EKS cluster we will need to update the AWS auth config map (`manage_aws_auth_configmap`)
- we also authorize Terraform to access the kubernetes API and modify the AWS auth config map
  + this requires a Terraform kubernetes provider ("2-Eks.tf")
- to authenticate with the cluster from Terraform, you can user either:
  + token --has an expiration time
  + exec bloc --retrieve the token on each TF run
- if you wanted, for instance, to grant 'read only' access to the EKS cluster for say, a group of Developers
  + you could create a custom Kubernetes RBAC group and map it to the IAM role
- the design provisions OpenID provider to enable IAM Roles for service accounts

#### auto-scaling in EKS
- **karpenter**
  + creates k8s nodes using ec2 instances
  + based on the workload, it can select the correct ec2 instance type
  + karpenter is not tied to the autoscaling group
  + is something between Cluster Autosclaer  and Fargate profile
- **cluster autosclaer**
  + uses auto-scaling groups
  + adjusts desired size of group based on the k8s workload 
  + to create an IAM Role for cluster autocaler we will use "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"
  + this will require a policy that gives permissions to access and modify AWS autocaling groups
    + role_name: "cluster-autoscaler"
    + namespace: kube-system
##### aws-load-balancer-controller
- deployed via helm chart, via Terraform
- can be used to create ingresses as well as services of type `loadbalancer`
- for ingress, load balancer controller creates an `application` loadbalancer
- for service, it creates a `network` loadbalancer
- this design deploys the `aws-load-balancer-controller` with a Terraform `helm_release`
- the load-balancer controller has an IAM Role with permissions to create and manage aws load balancers
- by default the Helm chart creates two replicas, we have scaled this down to "1" in this code
- the Helm provisioning of the aws load-balancer controller will require:
  + cluster name
  + service account
  + an annotation to allow the Service Account to assume the IAM role (this is IRSA at work!)
  + the aws load balancer to use tags to discover which subnets in can create loadbalancers in
- the EKS cluster must allow access from the EKS control plane to the webhook port
##### cluster autosclaler
- deployed in plan YAML (to demonstrate other options)
- the `kubectl` provider is capable of waiting until EKS is ready, and then deploy YAML
- (`kubernetes` provider does not support waiting until EKS is provisioned before deploying YAML manifests)

### validation and checkout commands
#### kubectl
  - `aws eks update-kubeconfig --name gd5 --region us-east-2`
  - `k get pods -n kube-system`
#### deployment test
  - `k logs -f -n kube-system -l app=cluster-autoscaler`
  - `k apply -f k8s/nginx.yaml`
  - `aws sts get-caller-identity --profile eks-admin`
```
aws eks update-kubeconfig \
--name my-eks \
--region us-east-1 \
--profile eks-admin
```

`error: You must be logged into the server (Unauthorized)`
kubectl auth can-i "*" "*"

## links
https://registry.terraform.io/providers/hashicorp/aws/latest/docs
https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-role.html#cli-configure-role-oidc