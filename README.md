# Terraform Module: terraform-aws-DynamoDB
# This module facilitates the creation of AWS DynamoDB for different purposes.

## Overview
The `terraform-aws-DynamoDB` module includes examples to easily deploy AWS DynamoDB using official Terraform module as a source.

## Usage

Example for deploying on-demand DynamoDB

```hcl
provider "aws" {
  region = "us-east-1"
}

module "dynamodb_table" {
  source = "github.com/terraform-aws-modules/terraform-aws-dynamodb-table"

  name                        = "my-table"
  hash_key                    = "id"
  range_key                   = "title"
  table_class                 = "STANDARD"
  deletion_protection_enabled = false

  attributes = [
    {
      name = "id"
      type = "N"
    },
    {
      name = "title"
      type = "S"
    },
    {
      name = "age"
      type = "N"
    }
  ]

  global_secondary_indexes = [
    {
      name               = "TitleIndex"
      hash_key           = "title"
      range_key          = "age"
      projection_type    = "INCLUDE"
      non_key_attributes = ["id"]
    }
  ]

  tags = {
    Terraform   = "true"
    Environment = "staging"
  }
}
```

Example for deploying global DynamoDB
```hcl
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "uswest2"
  region = "us-west-2"
}

locals {
  tags = {
    Terraform   = "true"
    Environment = "staging"
  }
}

################################################################################
# Supporting Resources
################################################################################


resource "aws_kms_key" "primary" {
  description = "CMK for primary region"
  tags        = local.tags
}

resource "aws_kms_key" "secondary" {
  provider = aws.uswest2

  description = "CMK for secondary region"
  tags        = local.tags
}

################################################################################
# DynamoDB Global Table
################################################################################

module "dynamodb_table" {
  source = "github.com/terraform-aws-modules/terraform-aws-dynamodb-table"

  name             = "my-table"
  hash_key         = "id"
  range_key        = "title"
  stream_enabled   = true
  stream_view_type = "NEW_AND_OLD_IMAGES"

  server_side_encryption_enabled     = true
  server_side_encryption_kms_key_arn = aws_kms_key.primary.arn

  attributes = [
    {
      name = "id"
      type = "N"
    },
    {
      name = "title"
      type = "S"
    },
    {
      name = "age"
      type = "N"
    }
  ]

  global_secondary_indexes = [
    {
      name               = "TitleIndex"
      hash_key           = "title"
      range_key          = "age"
      projection_type    = "INCLUDE"
      non_key_attributes = ["id"]
    }
  ]

  replica_regions = [{
    region_name            = "eu-west-2"
    kms_key_arn            = aws_kms_key.secondary.arn
    propagate_tags         = true
    point_in_time_recovery = true
  }]

  tags = local.tags
}
```

Example for deploying autoscaling DynamoDB
```hcl
provider "aws" {
  region = "eu-west-1"
}


module "dynamodb_table" {
  source = "github.com/terraform-aws-modules/terraform-aws-dynamodb-table"

  name                                  = "my-table"
  hash_key                              = "id"
  range_key                             = "title"
  billing_mode                          = "PROVISIONED"
  read_capacity                         = 5
  write_capacity                        = 5
  autoscaling_enabled                   = true
  ignore_changes_global_secondary_index = true

  autoscaling_read = {
    scale_in_cooldown  = 50
    scale_out_cooldown = 40
    target_value       = 45
    max_capacity       = 10
  }

  autoscaling_write = {
    scale_in_cooldown  = 50
    scale_out_cooldown = 40
    target_value       = 45
    max_capacity       = 10
  }

  autoscaling_indexes = {
    TitleIndex = {
      read_max_capacity  = 30
      read_min_capacity  = 10
      write_max_capacity = 30
      write_min_capacity = 10
    }
  }

  attributes = [
    {
      name = "id"
      type = "N"
    },
    {
      name = "title"
      type = "S"
    },
    {
      name = "age"
      type = "N"
    }
  ]

  global_secondary_indexes = [
    {
      name               = "TitleIndex"
      hash_key           = "title"
      range_key          = "age"
      projection_type    = "INCLUDE"
      non_key_attributes = ["id"]
      write_capacity     = 10
      read_capacity      = 10
    }
  ]

  tags = {
    Terraform   = "true"
    Environment = "staging"
  }
}
```