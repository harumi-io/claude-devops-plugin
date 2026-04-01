# Infrastructure Code Examples

Terraform code examples for common resources. Read this when implementing new resources to match existing patterns.

## S3 Bucket (AWS)

```hcl
module "s3_bucket" {
  source  = "cloudposse/s3-bucket/aws"
  version = "4.5.0"

  namespace = var.naming_namespace   # from .devops.yaml naming.namespace
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

## GCS Bucket (GCP)

```hcl
resource "google_storage_bucket" "this" {
  name          = "${var.project}-${var.environment}-${var.name}"
  location      = var.region
  storage_class = "STANDARD"

  uniform_bucket_level_access = true
  public_access_prevention    = "enforced"

  versioning {
    enabled = true
  }
}
```

## Azure Storage Account

```hcl
resource "azurerm_storage_account" "this" {
  name                     = "${var.prefix}${var.environment}${var.name}"
  resource_group_name      = var.resource_group_name
  location                 = var.location
  account_tier             = "Standard"
  account_replication_type = "LRS"

  min_tls_version                 = "TLS1_2"
  allow_nested_items_to_be_public = false
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

## Cloud Run Service (GCP)

```hcl
resource "google_cloud_run_v2_service" "this" {
  name     = "${var.project}-${var.name}"
  location = var.region

  template {
    containers {
      image = "${var.region}-docker.pkg.dev/${var.project}/${var.repository}/${var.image}:latest"

      resources {
        limits = {
          cpu    = var.cpu
          memory = var.memory
        }
      }

      dynamic "env" {
        for_each = var.environment_variables
        content {
          name  = env.value.name
          value = env.value.value
        }
      }
    }

    scaling {
      min_instance_count = var.min_instances
      max_instance_count = var.max_instances
    }

    vpc_access {
      connector = var.vpc_connector_id
      egress    = "PRIVATE_RANGES_ONLY"
    }
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

## GKE Cluster (GCP)

```hcl
resource "google_container_cluster" "this" {
  name     = "${var.project}-${var.environment}-gke"
  location = var.region

  initial_node_count       = 1
  remove_default_node_pool = true

  network    = var.network
  subnetwork = var.subnetwork

  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false
    master_ipv4_cidr_block  = var.master_cidr
  }
}

resource "google_container_node_pool" "primary" {
  name       = "primary"
  cluster    = google_container_cluster.this.name
  location   = var.region
  node_count = var.node_count

  node_config {
    machine_type = var.machine_type
    spot         = var.use_spot
    disk_size_gb = var.disk_size_gb
  }

  autoscaling {
    min_node_count = var.min_nodes
    max_node_count = var.max_nodes
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

# GCP GCS backend
data "terraform_remote_state" "core" {
  backend = "gcs"
  config = {
    bucket = var.state_bucket
    prefix = "core-infrastructure"
  }
}

# Azure azurerm backend
data "terraform_remote_state" "core" {
  backend = "azurerm"
  config = {
    resource_group_name  = var.state_rg
    storage_account_name = var.state_account
    container_name       = var.state_container
    key                  = "core-infrastructure.tfstate"
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
