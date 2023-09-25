# TF?!: Power of Terraform - A Simple Guide

In this guide, we'll delve into Terraform's capabilities, explore its revamped licensing, and provide you with a step-by-step walkthrough for effective infrastructure automation. 

Terraform is a custodian of infrastructure code, meticulously curated by diligent developers. Its purpose? To automate and manage the provisioning and configuration of infrastructure resources, spanning virtual machines, networks, and storage, across various cloud and on-premises environments. This ingenious tool, conceived by HashiCorp, comprises hundreds of modules written in the HashiCorp Configuration Language (HCL), collectively supported by a vibrant community of developers dedicated to deploying cloud and on-premises infrastructure and services.

With Terraform, you can say goodbye to the tedium of clicking buttons or typing repetitive command lines for deploying JS or static frontend files. Terraform simplifies infrastructure management by automating the provisioning and configuration of resources across various cloud and on-premises environments.


## Big Changes
> Starting from version 1.6, HashiCorp transitioned to a BSL license from MPL v2, marking significant changes in its journey.

On August 10, 2023, HashiCorp announced a significant shift in licensing for its products, including Terraform. After nearly a decade of being open source under the MPL v2 license, Terraform transitioned to a non-open source BSL v1.1 license, starting with version 1.6.

While this change primarily impacts the business landscape, it has triggered notable developments:

- On August 25th, 2023, the OpenTF initiative officially unveiled an open-source fork of Terraform.
- On September 5th, 2023, the OpenTF repository went public, quickly amassing over 32,000 GitHub Stars.
- On September 20th, 2023, OpenTF became a part of the Linux Foundation and underwent a rebranding, emerging as OpenTofu.


## Unveiling the Inner Workings

Behind the scenes, Terraform employs a dependency resolution algorithm that *automagically* figures out the order in which resources need to be created or updated. The process unfolds as follows:

### 1. Configuration Parsing
Terraform meticulously formats and parses your configuration files for syntax correctness, employing `tf fmt` and `tf validate` commands.

### 2. Resource Discovery
Terraform communicates with the configured providers (AWS, Azure, Google Cloud, etc.) to uncover the present state of the infrastructure resources aligned with your configuration.

### 3. Dependency Graph Construction
Terraform then constructs a dependency graph based on the relationships defined in your configuration. Each resource becomes a node in this graph, and any resource with dependencies is connected via direct references or pointers.

### 4. Topological Sorting
Topological sort is a graph traversal algorithm in which each node is visited only after all its dependencies are visited. In other words, resource dependencies are created before the actual resource.

### 5. Error Handling
Terraform's dependency resolution algorithm includes error handling. If a resource encounters issues during creation or updating, Terraform attempts to clean up successfully created resources with dependencies on the problematic resource before proceeding to create resources with no dependencies.

### 6. Parallelization Planning
While determining the correct order of resource creation or updates, Terraform also identifies opportunities for parallelization. Resources that lack dependencies on each other can be created or updated concurrently.

### 7. State Management
Terraform meticulously tracks the state of each resource. It knows whether a resource has been created, needs an update, or should be obliterated, based on differences between the desired configuration and the current state.

### 8. Dry Run
Terraform introduces a dry run, enabling you to preview the intended changes before committing to infrastructure modifications.

### 9. Apply Phase
In the apply phase, Terraform executes the plan generated during the dry run. It creates, updates, or destroys resources as necessary.

### 10. State Management (Again)
After applying changes, Terraform updates its internal state file to reflect the current state of the infrastructure. This state file proves invaluable for tracking changes and ensuring future runs remain idempotent.



## Putting Terraform to the Test with a Static Site

Let's dive into Terraform by automating the deployment of a static website on AWS, making it readily accessible to users. Before we delve into the sequential process of hosting a static website, a few things to note:

- This is a basic site, and we won't be configuring a custom domain or a signed certificate.
- It's ideal for testing your staging environment.
- This approach is compatible with static ReactJS or Next.js sites.
- Our provider of choice will be AWS.

## Preparing for Liftoff

For those new to Terraform and AWS, we have some preliminary steps:

### Setting Up AWS Configuration

- If you're new to AWS, start by running `aws configure` to generate your `region`, `aws_secret_access_key`, and `aws_access_key_id` into your `config` file at `~/.aws/`. `~` or `$HOME` represents your home directory on both unix and Windows systems, applicable to both Unix and Windows systems.
- Retrieve your `aws_secret_access_key` and `aws_access_key_id` from the AWS console by following the [AWS documentation](https://docs.aws.amazon.com/sdkref/latest/guide/access-iam-users.html).
  - Select the AWS region closest to your location for `region`.

### Terraform Setup

Visit the [HashiCorp website](https://developer.hashicorp.com/terraform/downloads) to set up Terraform.
Don't forget to [alias](https://www.tecmint.com/create-alias-in-linux/) `terraform` as `tf`. 


Now that you're ready, let's proceed.

### Infrastructure Components

Our infrastructure will comprise the following core resources:

- **Amazon S3 Bucket**: Amazon S3 (Simple Storage Service) is an object storage service provided by AWS. It allows you to store and retrieve data, such as files, images, videos, and backups, in the cloud. We'll be working with various aspects of it, including:
  - AWS S3 Objects (`aws_s3_object`): Representing individual objects stored in an S3 bucket.
  - AWS S3 Bucket Website Configuration (`aws_s3_bucket_website_configuration`): Configuring static website hosting for an S3 bucket.
  - AWS S3 Bucket Public Access Policy (`aws_s3_bucket_public_access_block`): Defining policies to control public access to an S3 bucket, enhancing security.
  - AWS S3 Bucket Ownership Controls (`aws_s3_bucket_ownership_controls`): Enabling ownership controls and management for S3 buckets to enforce data governance.
  - AWS S3 Bucket Policy (`aws_s3_bucket_policy`): Specifying access policies and permissions for an S3 bucket, governing who can interact with the bucket and how.

- **CloudFront**: Amazon CloudFront is a content delivery network (CDN) service provided by AWS. It distributes content like web pages, videos, images, and other assets globally with low latency and high data transfer speeds. Our usage will encompass:
  - AWS CloudFront Distribution (`aws_cloudfront_distribution`): Facilitating the delivery of content from edge locations for faster and more reliable access.
  - AWS CloudFront Origin Access Identity (`aws_cloudfront_origin_access_identity`): Establishing an Origin Access Identity (OAI), enhancing security by controlling access to the origin of a CloudFront distribution.

## The Code
### Provider and AWS Credentials Declaration

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "4.65.0"
    }
  }
}

provider "aws" {
  profile = "terraform"  # AWS Credentials Profile configured on your local desktop terminal at $HOME/.aws/credentials
  # region  = "us-east-1"  # Uncomment this line if not running on your local machine
  # access_key = var.aws_access_key_id  # Uncomment

 this line if not running on your local machine
  # secret_key = var.aws_secret_access_key  # Uncomment this line if not running on your local machine
}
```

> Remember that `aws configure` command? I named my AWS credential as `terraform`. You can easily confirm the name of your profile by running `cat ~/.aws/credentials`.

## Creating Bucket Resources

```hcl
# Create bucket resources
resource "aws_s3_bucket" "staticSite" {
  bucket = "staticsite-${var.ID}"

  tags = {
    Name        = "Static-Site bucket name"
    Environment = "Staging"
  }

}

```

> The `bucket` variable is mandatory.

```hcl
resource "aws_s3_object" "out" {
  for_each = fileset("../out", "**/*")

  bucket = aws_s3_bucket.staticSite.id
  key    = each.value
  source = "../out/${each.value}"
  etag   = filemd5("../out/${each.value}")
  # content_type = "text/html"
  content_type = lookup(var.mime_types, element(split(".", each.key), length(split(".", each.key)) - 1))

  depends_on = [aws_s3_bucket.staticSite]

}

```

This section enables you to upload all your site's files, treating each file and subdirectory as an individual resource. Let's break it down:

- `for_each = fileset("../out", "**/*")`: The `for_each` attribute allows you to create multiple instances of a resource based on the elements of a `map` or `set` variable. In essence, it loops through a set (array) and creates resources accordingly. `fileset` is a built-in function generating a list of file paths by matching a specified pattern, helping us create the set needed for looping through and creating these resources. You provide the directory path and the regex pattern as arguments.
  
  - `bucket`, `key`, `source`, `etag`, and `content_type` are attributes of the `aws_s3_object` resource. Among these, `bucket` and `key` are mandatory.
    - `bucket` represents the bucket name or ARN value.
    - `key` denotes the name to assign to the file in the bucket.
    - `source` points to the file path.
    - `etag` triggers updates when the file changes.
    - `content_type` indicates the standard MIME type describing the format of the object.
    - `filemd5` is a cryptographic hash function used to generate a fixed-size string of characters.
    - `lookup(var.mime_types, element(split(".", each.key), length(split(".", each.key)) - 1))` extracts the file extension and retrieves the appropriate MIME type from the mapped string. This step is vital; without proper MIME types, files may not display correctly on the website. I created a variable called `mime_types` in `variable.tf` file to store the mapped MIME values:

```hcl
variable "mime_types" {
  type = map(string)

  default = {
    "html" = "text/html"
    "ico"  = "image/x-icon"
    "jpg"  = "image/jpeg"
    "py"   = "text/x-python"
    ...
    "json" = "application/json"
    "map"  = "application/json"
    "txt"  = "text/plain"
  }
}
```

```hcl
resource "aws_s3_bucket_website_configuration" "staticSite-website" {
  bucket = aws_s3_bucket.staticSite.id

  index_document {
    suffix = "index.html"
  }

  error_document {
    key = "404.html"
  }
}
```

> The `index_document` specifies the file to load as the homepage, and `error_document` steps in when a file is not found.

```hcl
resource "aws_s3_bucket_public_access_block" "staticSite-ACL" {
  bucket = aws_s3_bucket.staticSite.id

  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}
```

> This ensures that all public access is permitted for users to view (read) the files. We will define a policy that allows only read operations below.

```hcl
resource "aws_s3_bucket_ownership_controls" "staticSite-object_ownership" {
  bucket = aws_s3_bucket.staticSite.id

  rule {
    object_ownership = "ObjectWriter"
  }
}
```

> Ownership controls are used to enforce object ownership when uploading objects to the S3 bucket. In this context, the "ObjectWriter" rule means that only the AWS account that uploads an object is considered the object's owner.

```hcl
resource "aws_s3_bucket_policy" "staticSite-Policy" {
  bucket = aws_s3_bucket.staticSite.id

  policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadForGetBucketObjects",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::${aws_s3_bucket.staticSite.id}/*"
    }
  ]
}
EOF

  depends_on = [aws_s3_bucket.staticSite]
}
```

> This standard AWS policy allows read actions for all S3 files in the AWS ARN.

## CloudFront

```hcl
resource "aws_cloudfront_origin_access_identity" "CloudFrontOriginAccessIdentity" {
  comment = "Origin Access Identity for Serverless Static Website"
}
```

> This section creates an AWS CloudFront Origin Access Identity (OAI).

```hcl
resource "aws_cloudfront_distribution" "WebpageCDN" {
  origin {
    domain_name = "onlinesafety-${var.ID}.s3-website-us-east-1.amazonaws.com"
    origin_id   = "webpage"
    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "match-viewer"  # https-only | http-only | match-viewer
      origin_ssl_protocols   = ["SSLv3", "TLSv1", "TLSv1.1", "TLSv1.2"]  # Ensure compatibility with a wide range of clients and servers.
    }
  }

  enabled             = true
  default_root_object = "index.html"

  default_cache_behavior {
    forwarded_values {
      query_string = false

      cookies {
        forward = "none"
      }
    }
    target_origin_id       = "webpage"
    viewer_protocol_policy = "allow-all"  # Changed back from redirect-to-https


    allowed_methods        = ["GET", "HEAD", "OPTIONS"]
    cached_methods         = ["GET", "HEAD", "OPTIONS"]
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }

  depends_on = [aws_s3_bucket.OnlineSafetyBS, aws_s3_object.out]
}
```

This section creates the CloudFront distribution:

- `origin`:
  - `domain_name`: Specifies the domain name or endpoint of the origin server in your configuration.
  - `origin_id`: Defined by the user, serving as an identifier for the origin.

- `enabled`: The enabled attribute is set to true, signifying that the CloudFront distribution is operational.
- `default_root_object`: Defines the default object to serve when a viewer requests the root URL of your distribution.
- `default_cache_behavior`: Cache behavior settings should adhere to your region's Data Protection Regulations. Here, CloudFront does not forward any cookies to the origin server.
- `restrictions`: Typically set to none unless you wish to restrict access to certain regions, like China or North Korea, due to policy considerations.
- `viewer_certificate`: As we don't have a custom certificate, we allow AWS CloudFront to provide one for us, enabling HTTPS browsing.

```hcl
output "CloudFrontURL" {
  value       = "https://${aws_cloudfront_distribution.WebpageCDN.domain_name}"
  description = "URL for the CloudFront distribution"
}
```

> This outputs the CloudFront URL, making it readily accessible.

## Running Terraform Commands

### Initialization

```sh
terraform init
```

`init`, short for initialization, is typically the first command you run when working with Terraform. It accomplishes the following:

- Initializes the Working Directory: Checks if your Terraform configuration files are present in the current working directory.
- Downloads Provider Plugins: Downloads and installs necessary provider plugins. Providers declared in your configuration files could be AWS, Azure, Google Cloud, or others.
- Sets Up Backend Configuration: Backend configuration specifies where Terraform should store its state files. In this example, they are stored locally, but Terraform Cloud is an example of how they could be stored remotely.
- Creates and Initializes Modules: If modules are included, `terraform init` also initializes the modules, downloading any required module sources or dependencies.
- Generates Lock Files: Generates a `terraform.lock.hcl` file that records the versions of provider plugins.
- Verifies Configuration Files: Terraform checks the syntax and validity of your configuration files.

### Planning

```sh
terraform plan
```

- Previews Changes: Provides a detailed preview of the infrastructure changes that Terraform will make (Dry Run).
- Dependency Analysis: Terraform evaluates the dependencies between resources and shows you the order in which changes will be applied.

> Note that Terraform expects a Terraform variable file or inline input using `terraform plan`. You can add your variables to `terraform.tfvars` as key-value pairs or input them inline, e.g., `terraform plan -var "key1=value1" -var "key2=value2"`.

### Applying Changes

```sh
terraform apply
```

- Terraform `plan` effectively runs again to outline the changes that will be made.
- Request for Approval: Terraform seeks your approval.
- After your approval, Terraform proceeds to make the necessary changes to your infrastructure. It communicates with the cloud provider's APIs or other infrastructure management tools to create or modify resources.
- Terraform is designed to be idempotent, meaning it can be run multiple times without causing harm. If the desired state matches the actual state, Terraform takes no action.

### Destroying Resources

```sh
terraform destroy
```

- If you want to reverse changes made by a previous `terraform apply`, you can use `terraform destroy` to remove the created resources.
- Be cautious when using `destroy`, as it permanently deletes resources.


## Conclusion
Terraform stands as a powerful ally in the realm of infrastructure automation. Its declarative approach, dependency resolution, and seamless integration with major cloud providers empower DevOps professionals to efficiently orchestrate, manage, and scale complex infrastructures.

By following the steps and best practices outlined here, you'll harness the power of Terraform to efficiently manage your cloud resources. Whether you're a seasoned DevOps engineer or just starting your cloud journey.


---

~ *By **Adebayo Omolumo***

[<img src="https://img.shields.io/badge/LinkedIn-Adebayo_Omolumo-brightgreen?style=for-the-badge&labelColor=blue&logo=linkedin" />](https://www.linkedin.com/in/adebayo-omolumo/) 
[<img src="https://img.shields.io/badge/Portfolio-🚀_-brightgreen?style=for-the-badge&labelColor=blue&logo=cloudways" />](https://visit.adebayoomolumo.website/projects/onepage.html) 
