+++
date = '2025-04-02T12:03:48-05:00'
draft = true
title = '7 Years of Terraform Lessons'
+++

I began my DevOps journey in 2019, and nearly right out of the gate I began working with Terraform. It has been nearly 7 years since then and I have picked up a trick or two when it comes to writing Terraform and I thought I should share that knowledge. So here we go.

For easy navigation, the tips, tricks, patterns, etc that I will be discussing are:
1. [Write Data Driven Terraform](#write-data-driven-terraform)
1. [Avoid Lists, Opt for Maps](#avoid-lists-opt-for-maps)
1. [Use Feature Flags](#use-feature-flags)
1. [Avoid the Object Type, Use the Any Type](#avoid-the-object-type-use-the-any-type))
1. [Avoid Using Resources Directly in a Root Module](#avoid-using-resources-directly-in-a-root-module)
1. [Use a Single Locals Block](#use-a-single-locals-block)
1. [Use Workspaces](#use-workspaces)
1. [Include the Workspace Name in Your Variables File Name](#include-the-workspace-name-in-your-variables-file-name)

### Write Data Driven Terraform

Recently, I have began writing my Terraform Root Modules in a Data Driven approach. What do I mean by this? I mean instead of creating very specific resources with very specific names in your Terraform code, write code that is generic and will operate off of a variable or local variable to create resources.

Let's compare two examples, that achieve the same end result of creating an AWS S3 Bucket for a static website.

#### Example: Before using a data driven approach
```hcl
# terraform.tfvars

# variables.tf
variable "bucket_name" {
  type = string
}

variable "force_destroy" {
  type    = bool
  default = false
}

# main.tf
module "website" {
  bucket_name   = var.bucket_name
  force_destroy = var.force_destroy
  # ... removed for brevity
}
```

#### Example: After using a data driven approach
```hcl
# terraform.tfvars
buckets = {
  "my-static-website-123" = {
    create = true
    force_destroy = true
  }
}

# variables.tf
variable "buckets" {
  type    = any
  default = {}
}

# main.tf
module "buckets" {
  for_each      = var.buckets
  create        = lookup(each.value, "create", true)
  bucket_name   = each.key
  force_destroy = lookup(each.value, "force_destroy", false)
  # ... removed for brevity
}
```

Many of the following points will assist in implementing a data driven approach to writing Terraform.

---

### Avoid Lists, Opt for Maps
Have you every iterated over a list to create multiple instances of resources or modules only to eventually need to delete one and when you do, the terraform plan wants to delete and recreate resources that you were not expecting to be touched?

---

### Use Feature Flags
This might resonate with those of you with development backgrounds, who are now working as DevOps Engineers.

Yes, even with our infrastructure as code, we should write our code to respect feature flags. There are two types of feature flags that we can have in Terraform. The global feature flag and the resource feature flag.

#### Global Feature Flag

The __*global feature flag*__ is just like it sounds. It is a global feature flag, or variable, that can control the creation of all resources.

#### Example

The below example shows how we are controlling the creating of our S3 Buckets and RDS Instances with a simple `create` variable. With this, we can easily toggle if all resources should be created or not.

```hcl
module "s3" {
  source   = "terraform-aws-modules/s3-bucket/aws"
  for_each = var.s3_buckets

  create_bucket = var.create
  # ... removed for brevity
}

module "rds" {
  source   = "terraform-aws-modules/rds/aws"
  for_each = var.rds_instances

  create_db_instance = var.create
  # ... removed for brevity
}
```


#### Resource Feature Flag

The __*resource feature flag*__ is also very descriptive in name. It is a variable that will control the creation of a subset of resources. It can be applied to individual resources, or to an entire module.

#### Example

The below example is using our data driven approach to writing Terraform. 

The variable `s3_buckets` is a list of maps which defines the properties of our S3 Buckets. 

We can include a property `create_bucket` that will control if this particular bucket should be created or not. If this property exists in the object, the value of this property will be passed down into the module where the module itself has been written using feature flags, otherwise `true` will be passed down.

```hcl
module "s3" {
  source   = "terraform-aws-modules/s3-bucket/aws"
  for_each = var.s3_buckets

  create_bucket = lookup(each.value, "create_bucket", true)
  # ... removed for brevity
}
```

#### Example: Combining The Global Feature Flag and Resource Feature Flag

The true power of using feature flags within Terraform is using the global feature flag and resource feature flag in combination together.

In the example below, we are using our two previous example together. This will allow us to control the creation of resources holistically as well as on a per-resource basis.

```hcl
module "s3" {
  source   = "terraform-aws-modules/s3-bucket/aws"
  for_each = var.s3_buckets

  create_bucket = var.create && lookup(each.value, "create_bucket", true)
  # ... removed for brevity
}
```

#### Benefits

So why should you write your Terraform using feature flags? What are the benefits that we get from this? This is a good question to ask as it become quite tedious to ensure that all of your resources include a feature flag to control it's creation.

##### Keeping Code Clean

Terraform code to create the resource(s) stays in place and is there in the future if needed. In a data driven configuration, we can have some elements of a map have a `create` variable configured as `false`. We no longer have to delete code, or leave code in place, but commented to prevent creation of resources.

##### The Lives of Pull Request Reviewers Becomes Much Nicer
Pull requests become much easier to understand. As a reviewer, I will be able to see that you are only preventing the creation of a specific resource or set of resources. The change in the pull request is a one liner, `create = false`. A reviewer can see this and determine if the request can be approved or not in seconds.

##### Preventing Drift Detection False Positives
A global `create` variable will allow us to prevent the creation of the entire root module if necessary. This is useful in the scenario where you have a drift detection workflow. By destroying resources with `terraform destroy` instead of configuring the global `create` variable to `false`, the drift detection workflow will determine that drift has occurred since the `terrform plan` will determine the resources need to be created.

---

### Avoid the Object Type, Use the Any Type

---

### Avoid Using Resources Directly in a Root Module

This is more of a pet peeve of mine. I would could call it an anti-pattern. 

---

### Use a Single Locals Block

I can't tell you how many times where I have attempted to create a `local` variable with a name just to execute a `terraform plan` to find out that a `local` variable with the same name already exists. 

There has also been many times where multiple `local` variables have been created that contain the same value, but have different names because they were created by different DevOps Engineers and defined in different places in the code base.

The code just simply is not readable when there are various `locals` blocks scattered throughout different files, sometimes multiple occurrences in the same file.

How do we fix this?

Easy, keep all of your `local` variables together in a single `locals` block. This block could be in a dedicated `locals.tf` file, or maybe at the top of your `main.tf`. The point is, there is one place that you and your team will go to see what `local` variables are available and place to add a new one if necessary. 

#### Example
```hcl
locals {
  # Add examples...
}
```

#### Benefits
1. All `local` variables are in a single place
1. I know where to look to view the existing `local` variables
1. I know where to define a new `local` variable

---

### Use Workspaces

It is the year of Our Lord 2025, but I still see people using directories or git branches to isolate environments. There are several problems with these approaches. It is these reasons that Hashicorp listened to the community and developed workspaces to begin with.

#### Benefits
1. Code de-duplication
1. Separate state files

---

### Include the Workspace Name in Your Variables File Name

If you name your variable files where the name of the file includes the workspace name, it becomes very easy to execute a plan, or any command for that matter the requires the variable files. In the majority of my root module repositories, I have a single variable file for each workspace. I can execute a plan quickly by using the bellow command.

```text
terraform plan -var-file=vars/$(terraform workspace show).tfvars -out tfplan
```

This command could easily be turned into a bash alias as well.