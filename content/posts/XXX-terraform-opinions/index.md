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

Recently, I have started writing my Terraform Root Modules in a Data Driven approach. Writing Terraform in a Data Driven approach is to write Terraform code that is generic and will operate off of a variable to create resources instead of creating very specific resources with very specific names in your Terraform code.

Let's compare two examples, that achieve the same end result of creating an AWS S3 Bucket for a static website, but are written the more typical way vs written with a data driven approach.

#### Example: Before using a data driven approach
```hcl
# terraform.tfvars
bucket_name   = "my-static-website-123"
force_destroy = true

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

#### Benefits

A single Root Module could theoretically handle all use cases, instead of having several Root Modules where the root module itself is named with a specific name and its resources/sub-modules have specific names. 

For example, prior to writing Terraform with this approach, I would end up with several "networking" root modules. A networking root module for the AWS Transit Gateway, implementing the hub and spoke networking module. A networking root module for each shared services VPC, workload VPC, etc. Depending on the complexity of your internal network, this could result in several root modules with a lot of duplicative code and the only differences really being the names of resources.

The ability to configure features of your Terraform with ease becomes second nature. Need to destroy an S3 Bucket, easy, configure `create = false` in your variables file. Need to add an S3 Bucket Policy to only one of your buckets? Easy, configure a `bucket_policy` variable with the policy that you need.

Many of the following points will assist in implementing a data driven approach to writing Terraform.

---

### Avoid Lists, Opt for Maps

Have you every iterated over a list to create multiple instances of resources or modules only to eventually need to delete one and when you do, the terraform plan wants to delete and recreate resources that you were not expecting to be touched?

I think this is a right of passage for all Terraform developers. It is the most natural way to think and write Terraform when you need to create several types of resources. The problem here is that we are creating physical resources in the cloud. These resources are assigned a path in the Terraform state file. By changing the path in the state file, Terraform will determine the resource needs to be recreated. Actually, not even that it needs to be recreated, but Terraform views them as two different resources. So it will attempt to destroy a resource and create a new resource.

Lists are indexed with integers, so the list

```hcl
private_subnets = [
    "",
    "",
    "",
    ""
]
``

is iterated as 

```hcl
private_subnets[0] = ""
private_subnets[1] = ""
private_subnets[2] = ""
private_subnets[3] = ""
```

---

### Use Feature Flags
This might resonate with those of you with development backgrounds, who are now working as DevOps Engineers.

Yes, even with our infrastructure as code, we should write our code to respect feature flags. There are two types of feature flags that we can have in Terraform. The global feature flag and the resource feature flag.

#### Global Feature Flag

The __*global feature flag*__ is just like it sounds. It is a global feature flag, or variable, that can control the creation of all resources.

##### Example

The below example shows how we are controlling the creating of our S3 Buckets and RDS Instances with a simple `create` variable. With this, we can easily toggle if the resources within those two modules should be created or not.

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

The __*resource feature flag*__ is also very descriptive in it's name. It is a variable that will control the creation of a subset of resources. It can be applied to individual resources, or to an entire module.

##### Example

The below example is using our data driven approach to writing Terraform. 

The variable `s3_buckets` is a map of objects that describe the properties of our S3 Buckets. 

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

##### Clean Code

Terraform code to create the resource(s) stays in place and is there in the future if needed. In a data driven configuration, we can have some elements of a map have a `create` variable configured as `false`. We no longer have to delete code, or leave code in place, but commented to prevent creation of resources.

There is nothing worse than having to comment code out because it is not necessary any more for whatever the reason may be. That or completely deleting code that could be useful in the future.

##### The Lives of Pull Request Reviewers Becomes Much Nicer

Pull requests become much easier to understand. As a reviewer, I will be able to see that you are only preventing the creation of a specific resource or set of resources. Instead of commenting out or deleting potentially large chunks of code, the change in the pull request becomes a one liner, `create = false`. A reviewer can see this and determine if the request can be approved or not within seconds.

##### Preventing Drift Detection False Positives

Have you ever been in the scenario where you executed a `terraform destroy` one day only to come jump online the next to see that the drift detection workflow failed? This can happen because in the context of the drift detection workflow, Terraform does not know that you intentially destroyed the resources in that workspaces. Terraform will come to the conclusion that a drift has occured and mark the workflow failed.

This is where the global feature flag can come in handy. If you intend to destroy all resources in a workspace, configuring this feature flag will survive the imperative `terraform destroy` command. In fact, `terraform destroy` becomes a `terraform apply`. With this variable being set in the `.tfvars` file for the workspace, the drift detection workflow will now pass because it is now aware of the fact that you do intend to not create any resources.

---

### Avoid the Object Type, Use the Any Type

Once you begin down the path of writing Data Driven Terraform, it will feel natural to use the `object` variable type. At first this seems the obvious choice, but there is a drawback with this type. 

The drawback of the `object` type is that if you have a variable that is of type `map(object(...))`, you will quickly run into Terraform complaining that not all objects in the map are of the same type. We need to work around this somehow because there will definitely be times where our objects do not have same structure. 

For example, we could want to set a default value if a value is not given in the variable by whoever is consuming this module.

How can we get around this? Enter `type = any`. By setting our variables type to be `type = any`, we can now give it any type we would like. It could be a map, list, number, string, etc. This is how many of the popular Terraform modules work under the hood.

You would likely want to be extra careful when writing documentation for this variable. There is a good chance that someone consuming this module, or providing a variable file will not know the structure that the variable will need to be in. Open source modules use this technique often and usually my biggest question, do they expect a map of objects or a list of objects?

You may be wondering how to consume the different properties of an object if the type is `any`. It may be tempting to access the property directly, such as `myvar["my_property_that_might_exist"]`. I won't blame you for trying this, but this will fail if the property does not exist. We don't want our Terraform blowing up for this reason.

Terraform has a solution for this. Terraform has a `lookup` function that takes three parameters, the object, the name of the property that you want to select within the object, and the default value if the property does not exist.

```hcl
# This will cause Terraform to blow up if the property "force_destroy" does not exist in the variable "my_bucket"
force_destroy = var.my_bucket["force_destroy"]

# This will succeed regardless if the property exists or not
force_destroy = lookup(var.my_bucket, "force_destroy", true)
```

#### Example

The below example shows how we can provide a single variable for creating many AWS S3 Buckets. You can see that we are defining properties for two buckets as objects, but their structure do not match. The second bucket has an additional `create` property configured. 

If you are writing Terraform with feature flags, this will be a very common use case where the objects of a map may not match.

```hcl
# variables.tf
variable "create" {
  type    = bool
  default = true
}

variable "buckets" {
  type    = any
  default = {}
}

# terraform.tfvars
buckets = {
  "bucketA" = {
    force_destroy = true
  },
  "bucketB" = {
    create       = false
    force_destroy = false
  }
}

# main.tf
module "s3" {
  source   = "terraform-aws-modules/s3-bucket/aws"
  for_each = var.buckets

  create_bucket = var.create && lookup(each.value, "create_bucket", true)
  force_destroy = lookup(each.value, "force_destroy", true)
  # ... removed for brevity
}
```

#### Benefits

1. We are able to use have a map of objects without using `type = map(object(...))` as Terraform will blow up if the objects' structures do not match
1. We can safely pluck the value of a property in an object without having to worry if the property exists or not
1. We can provide a default value when the property does not exist in the object

---

### Avoid Using Resources Directly in a Root Module

This is more of a pet peeve of mine. I personally would could call this an anti-pattern. 

Terraform is built on top of the idea of modules. So why should we be using resources directly in the Root Module? We should be grouping like resources together into a module where we can reference that module inside of the Root Module.

There are times when you are using a public, community developed moduule that does not support your use case. The most obvious solution to this is to use this community module as normal and create whatever resources necessary along side it to achieve the functionality of your use case.

While this is definitely an option, and a widely used and accepted solution to this problem, I would suggest another solution. Instead of having the instances of these resources exist directly in the Root Module. I would instead create a new module where this module contains the community module and the additional resources.

#### Benefits

##### Modularity

As mentioned before, Terraform is built on top of the idea of modules. By embracing this idea of modules, we end of with a module that is written in a generalized and reusable way. This will allow us to easily expand the module in the future, or possibly share our module with others without others being privvy to our use case.

Our future selves with thank us for this, trust me!

##### Readability

I often find myself context switching between Terraform Root Modules multiple times in a day. It is important for my own efficiency that when I context switch, I can quickly gather what the Terraform code is doing and resources that it manages. 

With the use of modules, I can very quickly infer that the AWS VPC module is for managing network resources, or that the AWS IAM EKS Role module is for managing a role that is to be assumed by an EKS Service Account.

By looking at the name of the module and it's inputs, these things are quick to infer. Without these modules, I would have to look at a file that could have 500, 800, or possibly even 1000 lines of Terraform code. This is too much to read through and quickly determine what resources are being created and managed.

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