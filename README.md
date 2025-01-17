<!-- BEGINNING OF PRE-COMMIT-TERRAFORM DOCS HOOK -->

# Why

To spin up complete eks with all necessary components.
Those include:
- vpc
- eks cluster
- alb ingress controller
- fluentbit
- external secrets
- metrics to cloudwatch

## How to run
```hcl
data "aws_availability_zones" "available" {}

locals {
    vpc_name = "dasmeta-prod-1"
    cidr     = "10.1.0.0/16"
    availability_zones = data.aws_availability_zones.available.names
    private_subnets = ["10.1.1.0/24", "10.1.2.0/24", "10.1.3.0/24"]
    public_subnets  = ["10.1.4.0/24", "10.1.5.0/24", "10.1.6.0/24"]
    cluster_enabled_log_types = ["audit"]

    # When you create EKS, API server endpoint access default is public. When you use private this variable value should be equal false.
    cluster_endpoint_public_access = true
    public_subnet_tags = {
        "kubernetes.io/cluster/production"  = "shared"
        "kubernetes.io/role/elb"            = "1"
    }
    private_subnet_tags = {
        "kubernetes.io/cluster/production"  = "shared"
        "kubernetes.io/role/internal-elb"   = "1"
    }
   cluster_name = "your-cluster-name-goes-here"
  alb_log_bucket_name = "your-log-bucket-name-goes-here"

  fluent_bit_name = "fluent-bit"
  log_group_name  = "fluent-bit-cloudwatch-env"
}

# Minimum

module "cluster_min" {
  source  = "dasmeta/eks/aws"
  version = "0.1.1"

  cluster_name        = local.cluster_name
  users               = local.users
  vpc_name            = local.vpc_name
  cidr                = local.cidr
  availability_zones  = local.availability_zones
  private_subnets     = local.private_subnets
  public_subnets      = local.public_subnets
  public_subnet_tags  = local.public_subnet_tags
  private_subnet_tags = local.private_subnet_tags
}

# Max @TODO: the max param passing setup needs to be checked/fixed

module "cluster_max" {
  source  = "dasmeta/eks/aws"
  version = "0.1.1"

  ### VPC
  vpc_name              = local.vpc_name
  cidr                  = local.cidr
  availability_zones    = local.availability_zones
  private_subnets       = local.private_subnets
  public_subnets        = local.public_subnets
  public_subnet_tags    = local.public_subnet_tags
  private_subnet_tags   = local.private_subnet_tags
  cluster_enabled_log_types = local.cluster_enabled_log_types
  cluster_endpoint_public_access = local.cluster_endpoint_public_access

  ### EKS
  cluster_name          = local.cluster_name
  manage_aws_auth       = true

  # IAM users username and group. By default value is ["system:masters"]
  user = [
          {
            username = "devops1"
            group    = ["system:masters"]
          },
          {
            username = "devops2"
            group    = ["system:kube-scheduler"]
          },
          {
            username = "devops3"
          }
  ]

  # You can create node use node_group when you create node in specific subnet zone.(Note. This Case Ec2 Instance havn't specific name).
  # Other case you can use worker_group variable.

  node_groups = {
    example =  {
      name  = "nodegroup"
      name-prefix     = "nodegroup"
      additional_tags = {
          "Name"      = "node"
          "ExtraTag"  = "ExtraTag"
      }

      instance_type   = "t3.xlarge"
      max_capacity    = 1
      disk_size       = 50
      create_launch_template = false
      subnet = ["subnet_id"]
    }
  }

  node_groups_default = {
      disk_size      = 50
      instance_types = ["t3.medium"]
    }

  worker_groups = {
    default = {
      name              = "nodes"
      instance_type     = "t3.xlarge"
      asg_max_size      = 3
      root_volume_size  = 50
    }
  }

  workers_group_defaults = {
    launch_template_use_name_prefix = true
    launch_template_name            = "default"
    root_volume_type                = "gp2"
    root_volume_size                = 50
  }

  ### ALB-INGRESS-CONTROLLER
  alb_log_bucket_name = local.alb_log_bucket_name

  ### FLUENT-BIT
  fluent_bit_name = local.fluent_bit_name
  log_group_name  = local.log_group_name

  # Should be refactored to install from cluster: for prod it has done from metrics-server.tf
  ### METRICS-SERVER
  # enable_metrics_server = false
  metrics_server_name     = "metrics-server"
}
```
**/

## Requirements

| Name | Version |
|------|---------|
| <a name="requirement_terraform"></a> [terraform](#requirement\_terraform) | >= 0.14.11 |
| <a name="requirement_aws"></a> [aws](#requirement\_aws) | >= 3.31 |
| <a name="requirement_helm"></a> [helm](#requirement\_helm) | >= 2.4.1 |

## Providers

| Name | Version |
|------|---------|
| <a name="provider_aws"></a> [aws](#provider\_aws) | >= 3.31 |

## Modules

| Name | Source | Version |
|------|--------|---------|
| <a name="module_alb-ingress-controller"></a> [alb-ingress-controller](#module\_alb-ingress-controller) | ./modules/aws-load-balancer-controller | n/a |
| <a name="module_cloudwatch-metrics"></a> [cloudwatch-metrics](#module\_cloudwatch-metrics) | ./modules/cloudwatch-metrics | n/a |
| <a name="module_eks-cluster"></a> [eks-cluster](#module\_eks-cluster) | ./modules/eks | n/a |
| <a name="module_external-secrets"></a> [external-secrets](#module\_external-secrets) | ./modules/external-secrets | n/a |
| <a name="module_fluent-bit"></a> [fluent-bit](#module\_fluent-bit) | ./modules/fluent-bit | n/a |
| <a name="module_metrics-server"></a> [metrics-server](#module\_metrics-server) | ./modules/metrics-server | n/a |
| <a name="module_sso-rbac"></a> [sso-rbac](#module\_sso-rbac) | ./modules/sso-rbac | n/a |
| <a name="module_vpc"></a> [vpc](#module\_vpc) | ./modules/vpc | n/a |
| <a name="module_weave-scope"></a> [weave-scope](#module\_weave-scope) | ./modules/weave-scope | n/a |

## Resources

| Name | Type |
|------|------|
| [aws_caller_identity.current](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/caller_identity) | data source |
| [aws_region.current](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/region) | data source |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| <a name="input_account_id"></a> [account\_id](#input\_account\_id) | AWS Account Id to apply changes into | `string` | `null` | no |
| <a name="input_alb_log_bucket_name"></a> [alb\_log\_bucket\_name](#input\_alb\_log\_bucket\_name) | n/a | `string` | `""` | no |
| <a name="input_alb_log_bucket_path"></a> [alb\_log\_bucket\_path](#input\_alb\_log\_bucket\_path) | ALB-INGRESS-CONTROLLER | `string` | `""` | no |
| <a name="input_availability_zones"></a> [availability\_zones](#input\_availability\_zones) | List of VPC availability zones, e.g. ['eu-west-1a', 'eu-west-1b', 'eu-west-1c']. | `list(string)` | n/a | yes |
| <a name="input_bindings"></a> [bindings](#input\_bindings) | Variable which describes group and role binding | <pre>list(object({<br>    group     = string<br>    namespace = string<br>    roles     = list(string)<br><br>  }))</pre> | `[]` | no |
| <a name="input_cidr"></a> [cidr](#input\_cidr) | CIDR ip range. | `string` | n/a | yes |
| <a name="input_cluster_enabled_log_types"></a> [cluster\_enabled\_log\_types](#input\_cluster\_enabled\_log\_types) | A list of the desired control plane logs to enable. For more information, see Amazon EKS Control Plane Logging documentation (https://docs.aws.amazon.com/eks/latest/userguide/control-plane-logs.html) | `list(string)` | <pre>[<br>  "audit"<br>]</pre> | no |
| <a name="input_cluster_endpoint_public_access"></a> [cluster\_endpoint\_public\_access](#input\_cluster\_endpoint\_public\_access) | n/a | `bool` | `true` | no |
| <a name="input_cluster_name"></a> [cluster\_name](#input\_cluster\_name) | Creating eks cluster name. | `string` | n/a | yes |
| <a name="input_cluster_version"></a> [cluster\_version](#input\_cluster\_version) | Allows to set/change kubernetes cluster version, kubernetes version needs to be updated at leas once a year. Please check here for available versions https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html | `string` | `"1.22"` | no |
| <a name="input_create"></a> [create](#input\_create) | Whether to create cluster and other resources or not | `bool` | `true` | no |
| <a name="input_enable_metrics_server"></a> [enable\_metrics\_server](#input\_enable\_metrics\_server) | METRICS-SERVER | `bool` | `false` | no |
| <a name="input_enable_sso_rbac"></a> [enable\_sso\_rbac](#input\_enable\_sso\_rbac) | Enable SSO RBAC integration or not | `bool` | `false` | no |
| <a name="input_external_secrets_namespace"></a> [external\_secrets\_namespace](#input\_external\_secrets\_namespace) | The namespace of external-secret operator | `string` | `"kube-system"` | no |
| <a name="input_fluent_bit_name"></a> [fluent\_bit\_name](#input\_fluent\_bit\_name) | FLUENT-BIT | `string` | `""` | no |
| <a name="input_log_group_name"></a> [log\_group\_name](#input\_log\_group\_name) | n/a | `string` | `""` | no |
| <a name="input_manage_aws_auth"></a> [manage\_aws\_auth](#input\_manage\_aws\_auth) | n/a | `bool` | `true` | no |
| <a name="input_map_roles"></a> [map\_roles](#input\_map\_roles) | Additional IAM roles to add to the aws-auth configmap. | <pre>list(object({<br>    rolearn  = string<br>    username = string<br>    groups   = list(string)<br>  }))</pre> | `[]` | no |
| <a name="input_metrics_server_name"></a> [metrics\_server\_name](#input\_metrics\_server\_name) | n/a | `string` | `"metrics-server"` | no |
| <a name="input_node_groups"></a> [node\_groups](#input\_node\_groups) | Map of EKS managed node group definitions to create | `any` | <pre>{<br>  "default": {<br>    "desired_size": 2,<br>    "instance_types": [<br>      "t3.medium"<br>    ],<br>    "max_size": 4,<br>    "min_size": 2<br>  }<br>}</pre> | no |
| <a name="input_node_groups_default"></a> [node\_groups\_default](#input\_node\_groups\_default) | Map of EKS managed node group default configurations | `any` | <pre>{<br>  "disk_size": 50,<br>  "instance_types": [<br>    "t3.medium"<br>  ]<br>}</pre> | no |
| <a name="input_node_security_group_additional_rules"></a> [node\_security\_group\_additional\_rules](#input\_node\_security\_group\_additional\_rules) | n/a | `any` | <pre>{<br>  "ingress_cluster_8443": {<br>    "description": "Metric server to node groups",<br>    "from_port": 8443,<br>    "protocol": "tcp",<br>    "source_cluster_security_group": true,<br>    "to_port": 8443,<br>    "type": "ingress"<br>  }<br>}</pre> | no |
| <a name="input_private_subnet_tags"></a> [private\_subnet\_tags](#input\_private\_subnet\_tags) | n/a | `map(any)` | `{}` | no |
| <a name="input_private_subnets"></a> [private\_subnets](#input\_private\_subnets) | Private subnets of VPC. | `list(string)` | n/a | yes |
| <a name="input_public_subnet_tags"></a> [public\_subnet\_tags](#input\_public\_subnet\_tags) | n/a | `map(any)` | `{}` | no |
| <a name="input_public_subnets"></a> [public\_subnets](#input\_public\_subnets) | Public subnets of VPC. | `list(string)` | n/a | yes |
| <a name="input_region"></a> [region](#input\_region) | AWS Region name. | `string` | `null` | no |
| <a name="input_roles"></a> [roles](#input\_roles) | Variable describes which role will user have K8s | <pre>list(object({<br>    actions   = list(string)<br>    resources = list(string)<br>  }))</pre> | `[]` | no |
| <a name="input_send_alb_logs_to_cloudwatch"></a> [send\_alb\_logs\_to\_cloudwatch](#input\_send\_alb\_logs\_to\_cloudwatch) | Whether send alb logs to CloudWatch or not. | `bool` | `true` | no |
| <a name="input_users"></a> [users](#input\_users) | List of users to open eks cluster api access | `list(any)` | `[]` | no |
| <a name="input_vpc_name"></a> [vpc\_name](#input\_vpc\_name) | Creating VPC name. | `string` | n/a | yes |
| <a name="input_weave_scope_config"></a> [weave\_scope\_config](#input\_weave\_scope\_config) | Weave scope namespace configuration variables | <pre>object({<br>    create_namespace        = bool<br>    namespace               = string<br>    annotations             = map(string)<br>    ingress_host            = string<br>    ingress_class           = string<br>    ingress_name            = string<br>    service_type            = string<br>    weave_helm_release_name = string<br>  })</pre> | <pre>{<br>  "annotations": {},<br>  "create_namespace": true,<br>  "ingress_class": "",<br>  "ingress_host": "",<br>  "ingress_name": "weave-ingress",<br>  "namespace": "meta-system",<br>  "service_type": "NodePort",<br>  "weave_helm_release_name": "weave"<br>}</pre> | no |
| <a name="input_weave_scope_enabled"></a> [weave\_scope\_enabled](#input\_weave\_scope\_enabled) | Weather enable Weave Scope or not | `bool` | `false` | no |
| <a name="input_worker_groups"></a> [worker\_groups](#input\_worker\_groups) | Worker groups. | `any` | `{}` | no |
| <a name="input_workers_group_defaults"></a> [workers\_group\_defaults](#input\_workers\_group\_defaults) | Worker group defaults. | `any` | <pre>{<br>  "launch_template_name": "default",<br>  "launch_template_use_name_prefix": true,<br>  "root_volume_size": 50,<br>  "root_volume_type": "gp2"<br>}</pre> | no |

## Outputs

| Name | Description |
|------|-------------|
| <a name="output_cluster_certificate"></a> [cluster\_certificate](#output\_cluster\_certificate) | EKS cluster certificate used for authentication/access in helm/kubectl/kubernetes providers |
| <a name="output_cluster_host"></a> [cluster\_host](#output\_cluster\_host) | EKS cluster host name used for authentication/access in helm/kubectl/kubernetes providers |
| <a name="output_cluster_iam_role_name"></a> [cluster\_iam\_role\_name](#output\_cluster\_iam\_role\_name) | n/a |
| <a name="output_cluster_id"></a> [cluster\_id](#output\_cluster\_id) | n/a |
| <a name="output_cluster_primary_security_group_id"></a> [cluster\_primary\_security\_group\_id](#output\_cluster\_primary\_security\_group\_id) | n/a |
| <a name="output_cluster_security_group_id"></a> [cluster\_security\_group\_id](#output\_cluster\_security\_group\_id) | n/a |
| <a name="output_cluster_token"></a> [cluster\_token](#output\_cluster\_token) | EKS cluster token used for authentication/access in helm/kubectl/kubernetes providers |
| <a name="output_eks_auth_configmap"></a> [eks\_auth\_configmap](#output\_eks\_auth\_configmap) | n/a |
| <a name="output_eks_module"></a> [eks\_module](#output\_eks\_module) | n/a |
| <a name="output_eks_oidc_root_ca_thumbprint"></a> [eks\_oidc\_root\_ca\_thumbprint](#output\_eks\_oidc\_root\_ca\_thumbprint) | Grab eks\_oidc\_root\_ca\_thumbprint from oidc\_provider\_arn. |
| <a name="output_map_user_data"></a> [map\_user\_data](#output\_map\_user\_data) | n/a |
| <a name="output_oidc_provider_arn"></a> [oidc\_provider\_arn](#output\_oidc\_provider\_arn) | ## CLUSTER |
| <a name="output_role_arns"></a> [role\_arns](#output\_role\_arns) | n/a |
| <a name="output_role_arns_without_path"></a> [role\_arns\_without\_path](#output\_role\_arns\_without\_path) | n/a |
| <a name="output_vpc_cidr_block"></a> [vpc\_cidr\_block](#output\_vpc\_cidr\_block) | The cidr block of the vpc |
| <a name="output_vpc_default_security_group_id"></a> [vpc\_default\_security\_group\_id](#output\_vpc\_default\_security\_group\_id) | The ID of default security group created for vpc |
| <a name="output_vpc_id"></a> [vpc\_id](#output\_vpc\_id) | The newly created vpc id |
| <a name="output_vpc_nat_public_ips"></a> [vpc\_nat\_public\_ips](#output\_vpc\_nat\_public\_ips) | The list of elastic public IPs for vpc |
| <a name="output_vpc_private_subnets"></a> [vpc\_private\_subnets](#output\_vpc\_private\_subnets) | The newly created vpc private subnets IDs list |
| <a name="output_vpc_public_subnets"></a> [vpc\_public\_subnets](#output\_vpc\_public\_subnets) | The newly created vpc public subnets IDs list |
<!-- END OF PRE-COMMIT-TERRAFORM DOCS HOOK -->
