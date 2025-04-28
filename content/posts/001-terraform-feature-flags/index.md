+++
date = '2025-04-27T08:00:00-05:00'
title = 'Terraform: Feature Flags Are Not Just for Application Code'
tags = ["terraform"]
+++

Have you ever wished there was an easy way to prevent the creation of resources without commenting, or deleting parts of your Terraform code? What about preventing the creation of all resources in a Terraform Root Module? Well, you're in luck. We can achieve this by introducing feature flags to our Terraform codebase.

This might resonate with those of you with development backgrounds, who are now working as DevOps Engineers.

Yes, even with our infrastructure as code, we should write our code to respect feature flags. There are two types of feature flags that we can have in Terraform. The __*global feature flag*__ and the __*resource feature flag*__.

### Global Feature Flag

The __*global feature flag*__ is just like it sounds. It is a global feature flag, or variable, that can control the creation of all resources within the Root Module.

#### Example

The below example shows how we are controlling the creation of our S3 Buckets and RDS Instances with a simple `create` variable. With this, we can easily toggle if the resources within those two modules should be created or not.

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

### Resource Feature Flag

The __*resource feature flag*__ is also very descriptive in it's name. It is a variable that will control the creation of a subset of resources. It can be applied to individual resources, or to an entire module.

#### Example

The below example is using a data driven approach to writing Terraform. 

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

### Example: Combining The Global Feature Flag and Resource Feature Flag

The true power of using feature flags within Terraform is using the __global feature flag__ and __resource feature flag__ in combination together.

In the example below, we are using our two previous example together. This will allow us to control the creation of resources holistically as well as on a per-resource basis.

```hcl
module "s3" {
  source   = "terraform-aws-modules/s3-bucket/aws"
  for_each = var.s3_buckets

  create_bucket = var.create && lookup(each.value, "create_bucket", true)
  # ... removed for brevity
}
```

### Benefits

So why should you write your Terraform using feature flags? What are the benefits that we get from this? This is a good question to ask as it can become quite tedious to ensure that all of your resources include a feature flag to control it's creation.

#### Clean Code & Flexibility

Terraform code to create the resource(s) stays in place and is there in the future if needed. In a data driven configuration, we can have some elements of a map have a `create` variable configured as `false`. We no longer have to delete code, or leave code in place, but commented to prevent creation of resources.

There is nothing worse than having to comment code in a codebase because it is not necessary any more for whatever the reason may be. That or completely deleting code that could be useful in the future.

#### Reviewing Pull Requests Becomes Much Easier

Pull requests become much easier to understand. As a reviewer, I will be able to see that you are only preventing the creation of a specific resource or a set of resources. Instead of commenting out or deleting potentially large chunks of code, the change in the pull request becomes a one liner, `create = false`. A reviewer can see this and determine if the request can be approved or declined within seconds.

#### Preventing Drift Detection False Positives

Have you ever been in the scenario where you executed a `terraform destroy` one day only to come jump online the next to see that the drift detection workflow failed? This can happen because in the context of the drift detection workflow, Terraform does not know that you intentially destroyed the resources in that workspaces. Terraform will come to the conclusion that a drift has occured and mark the workflow failed.

This is where the global feature flag can come in handy. If you intend to destroy all resources in a workspace, configuring this feature flag will survive the imperative `terraform destroy` command. With this feature flag being set in the `.tfvars` file for the workspace, the drift detection workflow will now pass because it is now aware of the fact that you do not intend to create any resources.

### Conclusion

So there we have it. We are able to manage the creation of all resources, or a subset of resources in a Terrform Root Module. We saw how we can achieve this with __global__ and __resource feature flags__. We also saw the benefits of doing so which included keeping our codebase clean and improving it's flexibility, how pull requests become much easier to review, and still achieving the ability to execute drift detection workflows.
