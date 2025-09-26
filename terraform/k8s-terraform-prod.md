
# Terraform Configuration for Production EKS Cluster

This Terraform configuration provisions a **production-ready Amazon EKS cluster** with:
- VPC, subnets, and security groups
- IAM roles and policies for EKS
- EKS cluster with managed node group
- Helm deployment for NGINX Ingress Controller
- Outputs for cluster details

---

## Variables

```hcl
variable "aws_region" { default = "us-east-1" }
variable "cluster_name" { default = "prod-eks-cluster" }
variable "node_group_desired_capacity" { default = 3 }
variable "node_group_max_capacity" { default = 5 }
variable "node_group_min_capacity" { default = 2 }
```

---

## Providers

```hcl
provider "aws" { region = var.aws_region }
provider "kubernetes" {
  host                   = aws_eks_cluster.eks_cluster.endpoint
  cluster_ca_certificate = base64decode(aws_eks_cluster.eks_cluster.certificate_authority[0].data)
  token                  = data.aws_eks_cluster_auth.cluster.token
}
provider "helm" {
  kubernetes {
    host                   = aws_eks_cluster.eks_cluster.endpoint
    cluster_ca_certificate = base64decode(aws_eks_cluster.eks_cluster.certificate_authority[0].data)
    token                  = data.aws_eks_cluster_auth.cluster.token
  }
}
```

---

## Resources

### VPC

```hcl
resource "aws_vpc" "eks_vpc" {
  cidr_block = "10.0.0.0/16"
  tags = { Name = "eks-vpc" }
}
```

### Subnets

```hcl
resource "aws_subnet" "eks_subnets" {
  count                   = 3
  vpc_id                  = aws_vpc.eks_vpc.id
  cidr_block              = cidrsubnet(aws_vpc.eks_vpc.cidr_block, 8, count.index)
  availability_zone      = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
  tags = { Name = "eks-subnet-${count.index}" }
}
```

### EKS Cluster

```hcl
resource "aws_eks_cluster" "eks_cluster" {
  name     = var.cluster_name
  role_arn = aws_iam_role.eks_cluster_role.arn
  vpc_config { subnet_ids = aws_subnet.eks_subnets[*].id }
}
```

### IAM Roles & Policies

Includes roles for cluster and node groups with required policies.

### Node Group

```hcl
resource "aws_eks_node_group" "eks_node_group" {
  cluster_name    = aws_eks_cluster.eks_cluster.name
  node_group_name = "prod-eks-node-group"
  node_role_arn   = aws_iam_role.eks_node_group_role.arn
  subnet_ids      = aws_subnet.eks_subnets[*].id
  scaling_config {
    desired_size = var.node_group_desired_capacity
    max_size     = var.node_group_max_capacity
    min_size     = var.node_group_min_capacity
  }
}
```

### Helm Chart for NGINX Ingress

```hcl
resource "helm_release" "nginx_ingress" {
  name       = "nginx-ingress"
  chart      = "ingress-nginx"
  repository = "https://kubernetes.github.io/ingress-nginx"
  namespace  = "kube-system"
  values = [<<EOF
controller:
  replicaCount: 2
  service:
    type: LoadBalancer
EOF
  ]
}
```

---

## Outputs

```hcl
output "cluster_endpoint" { value = aws_eks_cluster.eks_cluster.endpoint }
output "cluster_name" { value = aws_eks_cluster.eks_cluster.name }
output "node_group_name" { value = aws_eks_node_group.eks_node_group.node_group_name }
```

---

## Usage

1. Initialize Terraform:
```bash
terraform init
```

2. Apply configuration:
```bash
terraform apply -var='aws_region=us-east-1'
```

This will create a production-ready EKS cluster.
