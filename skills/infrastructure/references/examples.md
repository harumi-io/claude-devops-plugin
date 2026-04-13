# Infrastructure Code Examples

Terraform code examples for common resources. Read this when implementing new resources to match existing patterns.

## S3 Bucket (AWS)

```hcl
module "s3_bucket" {
  source  = "cloudposse/s3-bucket/aws"
  version = "4.5.0"

  namespace = var.naming_namespace   # from harumi.yaml naming.namespace
  stage     = var.environment
  name      = var.name

  acl                = "private"
  versioning_enabled = true

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

## ECS Fargate Service (AWS)

```hcl
module "container_definition" {
  source         = "cloudposse/ecs-container-definition/aws"
  version        = "0.60.0"
  container_name = "${var.container_name}-${var.environment}"
  container_image = "${data.aws_caller_identity.current.account_id}.dkr.ecr.${var.region}.amazonaws.com/${var.image}:latest"
  essential      = true
  environment    = var.container_environment

  log_configuration = {
    logDriver = "awslogs"
    options = {
      "awslogs-group"         = "${var.container_name}-container"
      "awslogs-region"        = var.region
      "awslogs-create-group"  = "true"
      "awslogs-stream-prefix" = "logs"
    }
  }

  port_mappings = [for port in var.container_ports : {
    containerPort = port
    hostPort      = port
    protocol      = "tcp"
  }]
}

resource "aws_ecs_task_definition" "this" {
  execution_role_arn       = var.ecs_task_execution_role
  task_role_arn            = var.task_arn
  container_definitions    = module.container_definition.json_map_encoded_list
  cpu                      = var.task_cpu
  family                   = "${var.container_name}-${var.environment}-ecs-task"
  memory                   = var.task_memory
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
}

resource "aws_ecs_service" "this" {
  cluster              = var.ecs_cluster_id
  desired_count        = var.service_count
  launch_type          = var.use_spot_instances ? null : "FARGATE"
  name                 = "${var.container_name}-${var.environment}-ecs-task"
  task_definition      = aws_ecs_task_definition.this.arn
  force_new_deployment = true
  enable_execute_command = true

  lifecycle {
    ignore_changes = [desired_count]
  }

  dynamic "capacity_provider_strategy" {
    for_each = var.use_spot_instances ? [1] : []
    content {
      capacity_provider = "FARGATE_SPOT"
      weight            = 100
    }
  }

  network_configuration {
    security_groups  = var.security_group_ids
    subnets          = var.private_subnets
    assign_public_ip = false
  }
}
```

## EKS Cluster (AWS)

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "${var.naming_namespace}-${var.environment}-eks"
  cluster_version = var.kubernetes_version

  cluster_endpoint_public_access = true

  vpc_id     = var.vpc_id
  subnet_ids = var.private_subnet_ids

  eks_managed_node_groups = {
    spot = {
      instance_types = ["t3.large", "t3a.large", "m5.large"]
      capacity_type  = "SPOT"
      min_size       = var.spot_min_size
      max_size       = var.spot_max_size
      desired_size   = var.spot_desired_size
    }
    on_demand = {
      instance_types = ["t3.large"]
      capacity_type  = "ON_DEMAND"
      min_size       = 1
      max_size       = var.on_demand_max_size
      desired_size   = 1
    }
  }
}
```

## Remote State Reference

```hcl
# AWS S3 backend
data "terraform_remote_state" "core" {
  backend = "s3"
  config = {
    bucket = var.state_bucket
    key    = "core-infrastructure/terraform.tfstate"
    region = var.region
  }
}

```

## Feature Flags Pattern

```hcl
# tfvars
enable_spot_instances    = true
remove_legacy_nodegroup  = false  # ALWAYS default false for removal flags

# Conditional creation
resource "aws_eks_node_group" "spot" {
  count         = var.enable_spot_instances ? 1 : 0
  capacity_type = "SPOT"
}

resource "aws_eks_node_group" "legacy" {
  count         = var.remove_legacy_nodegroup ? 0 : 1
  capacity_type = "ON_DEMAND"
}
```

## Locals Pattern

```hcl
locals {
  common_tags = {
    Environment = var.environment
    Project     = var.naming_namespace
    ManagedBy   = "terraform"
  }
}
```
