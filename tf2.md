# <Terraform article title here>(tf?!)
- If you often find your self clicking buttons on your favorite cloud services portal in order to deploy your nextjs or static frontend files, you might want ot pay attention to this devops tool
- Terraform was basically created by lazy developers (like me) to not go through the hassle of clicking the same buttons or writing the same command prompt over and over. Just kidding
- I would define terraform as a custodian of infrastructure code written by extremely hardworking and diligent developers that want to automate and manage the provisioning and configuration of infrastructure resources, such as virtual machines, networks, and storage, across various cloud and on-premises environments. The tool itself (Terraform) was designed by Hashicorp and is composed of hundred of modules written in the hashicorp configuration language by communities of developers supporting cloud and on-premises infrastructure and services deployment 
- Terraform took a page out of the `Chef` handbook and uses a declarative syntax, which means you define what you want the infrastructure to look like rather than specifying the step-by-step procedures to create it. This makes Terraform configurations concise and easy to understand.
- While it has a large community behind the big three cloud provider. Terraform also supports major Cloud Providers like digital ocean and infrastructure services. It also supports on-premises and third-party services through custom providers

### What is happening behind the scenes?
Terraform uses a dependency resolution algorithm to automatically determine the order in which resources need to be created or updated.
- Configuration Parsing: Terraform formats and parses your configuration files for syntax correctness. `tf fmt` and `tf validate`
- Resource Discovery: Terraform communicates with the configured providers (e.g., AWS, Azure, Google Cloud) to discover the current state of the infrastructure resources that match your configuration.
- Resources can have dependencies on other resources. For example, a virtual machine may depend on a network interface, which in turn depends on a subnet and a virtual network. 
- Dependency Graph Construction: Terraform then constructs a dependency graph based on the relationships defined in the configuration you have written. Each resource is represented as a node in the graph, and any resource with a dependency is connected through direct references or pointers.
- Terraform then performs a topological sorting of the nodes. Topological sort is a graph traversal algorithm in which each node is visited only after all its dependencies are visited.
- Error Handling: Terraform's dependency resolution algorithm also includes error handling. If a resource cannot be created or updated due to an error. Terraform will to attempt to clean up any resources that were successfully created and has dependecny to the resource fatally created. If there are resources that has no dependency, goes ahead to create those resources. 
- Parallelization Planning: While Terraform determines the correct order of resource creation or updates, it also identifies opportunities for parallelization. Resources that do not have dependencies on each other can be created or updated concurrently
- Terraform keeps track of the state of each resource. It knows whether a resource has been created, needs to be updated, or should be destroyed based on the differences between the desired configuration and the current state.
- Dry Run: Dry run, which includes some of the steps mentioned before allows you to preview the changes it intends to make before making changes to infrastructure.
- Apply Phase: During the apply phase, Terraform executes the plan it generated during the dry run. It creates, updates, or destroys resources
- State Management: After applying changes, Terraform updates its internal state file to reflect the current state of the infrastructure. This state file is essential for tracking changes and ensuring future runs are idempotent.


# Test out how terraform works with an example (the waters with static site)
Let's use Terraform to automate the deployment of a static website on AWS, making it easily accessible to users.

But before we delve into the sequential process of hosting a static website, you should know:
- This is a basic site, we will not be using our own domain or a signed certificate
- This is best utilized to test your staging environment
- It should work with your static reactjs or nextjs site
- We will be using AWS as our provider

## PRE
If you already have experience with using terraform with AWS, you can skip this section.
- As mentioned earlier, we will be using AWS configuration with our terraform
- For first timers your should run aws configure to generate your `region`, `aws_secret_access_key` and `aws_access_key_id` into your `config` file at `~/.aws/`. Note `~` represents your home directory on both unix and Windows systems
  - You can get this values for `aws_secret_access_key` and `aws_access_key_id` from [AWS docs](https://docs.aws.amazon.com/sdkref/latest/guide/access-iam-users.html)
  - Pick the closest AWS region to you for your `region`
- Setup terraform from the [hashicorp site](https://developer.hashicorp.com/terraform/downloads).
  - Don't forget to alias `terraform` as `tf`

## Now you are ready
Let's jump in
You should know that our infrastructure will comprise of the following main resources:

- **Amazon S3 Bucket**:
Amazon S3 Bucket is an object storage service provided by Amazon Web Services (AWS) that allows you to store and retrieve data, such as files, images, videos, and backups, in the cloud.

  - AWS S3 Object (`aws_s3_object`): Represents any individual object stored in an AWS S3 bucket.
  - AWS S3 Bucket Website Configuration (`aws_s3_bucket_website_configuration`): Configures a static website hosting configuration for an AWS S3 bucket.
  - AWS S3 Bucket Public Access Policy (`aws_s3_bucket_public_access_block`): Defines policies to control public access to an AWS S3 bucket, enhancing security.
  - AWS S3 Bucket Ownership Controls (`aws_s3_bucket_ownership_controls`): Enables ownership controls and management for AWS S3 buckets to enforce data governance.
  - AWS S3 Bucket Policy (`aws_s3_bucket_policy`): Specifies access policies and permissions for an AWS S3 bucket, governing who can interact with the bucket and how.


- **Cloudfront**: Amazon CloudFront is a content delivery network (CDN) service provided by Amazon Web Services (AWS). It helps distribute content, such as web pages, videos, images, and other assets, to users globally with low latency and high data transfer speeds.
  - AWS Cloudfront Distribution (`aws_cloudfront_distribution`): Helps deliver content from edge locations for faster and more reliable access.

  - AWS Cloudfront Origin Access Identity (`aws_cloudfront_origin_access_identity`): Defines an Origin Access Identity (OAI), enhancing security by controlling access to the origin of a CloudFront distribution.


### Declaring Preovider and AWS credentials
``` py

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "4.65.0"
    }
  }
}


provider "aws" {
  profile = "terraform" # AWS Credentials Profile configured on your local desktop terminal  $HOME/.aws/credentials
#   region  = "us-east-1" # Only when not on your local machine
#   access_key = var.aws_access_key_id # Only when not on your local machine
#   secret_key = var.aws_secret_access_key # Only when not on your local machine
}

```
Remember that `aws configure` command, I named my AS credential as `terraform` You can easily confirm the name of your profile with `cat ~/.aws/credentials`




### Creating bucket resources
```py
locals {
  truncated_unique_id = "x7as9"
}

```


```py
resource "aws_s3_bucket" "staticSite" {
  bucket = "staticsite-${local.truncated_unique_id}" # bucket name and it most be gloablly unique

  tags = {
    Name        = "Static-Site bucket name"
    Environment = "Staging"
  }

}

```
> *bucket variable is compulsory 


```py
resource "aws_s3_object" "out" {
  for_each = fileset("../../frontend/out", "**/*") 

  bucket = aws_s3_bucket.staticSite.id
  key    = each.value
  source = "../../frontend/out/${each.value}"
  etag   = filemd5("../../frontend/out/${each.value}")
  # content_type = "text/html"
  content_type = lookup(var.mime_types, element(split(".", each.key), length(split(".", each.key)) - 1))

  depends_on = [aws_s3_bucket.staticSite]

}

```
Section allows you to upload all your sites files and treat each files and subdirectory files as an individual resource. Let break down the code
- `for_each = fileset("../../frontend/out", "**/*")`
  - `for_each`: the for_each meta-argument allows you to create multiple instances of a resource or module based on the elements of a `map` or `set` variable. In simple terms, it loops through a set(array) and creates resources based on what is provided
  - `fileset`: is a built-in function that allows you to generate a list of file paths by matching a specified pattern. This help us create that `set` we need to loop through and create those resources. You provide the directory path and the regex pattern as args
  - `bucket`, `key`, `source`, `etag`, and `content_type` are the aws_s3_object resources attributes and only `bucket`, `key` are required.
    - `bucket` is the bucket name or `arn` value. `key` is what name to give file in bucket. `source` is the path to the file. `etag` triggers updates when the file changes. `content_type` is standard MIME type describing the format of the object.
    - `filemd5` is cryptographic hash functions used to produce a fixed-size string of characters. 
    - `lookup(var.mine_types, element(split(".", each.key), length(split(".", each.key)) - 1))`
      - I created a variable called mime_types that maps all file format to their MIME
      - the line above extracts the file extension and looks up the proper MIME from the mapped string
      - This very important. The files will not be read properly on the webite if it doesn't have its corresponding MIME.
      - I created a variable.tf file to store the mapped MIME values

    ```py
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




```py
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
The index_document specifies the file to load at the homepage and error_document is loaded when a file is not found.



```py
resource "aws_s3_bucket_public_access_block" "staticSite-ACL" {
  bucket = aws_s3_bucket.staticSite.id

  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}

```
All of Public access should be allowed for users to view(read) the files.
A policy that will allow only read operations will be defined below


```py
resource "aws_s3_bucket_ownership_controls" "staticSite-object_ownership" {
  bucket = aws_s3_bucket.staticSite.id

  rule {
    object_ownership = "ObjectWriter"
  }
}


```
Ownership controls are used to enforce object ownership when objects are uploaded to the S3 bucket. The "ObjectWriter" rule, in this context, means that only the AWS account that uploads an object is considered the object's owner.



```py
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
Standard AWS policy that allows read action for all s3 files in the AWS ARN.



## Cloudfront
```py
resource "aws_cloudfront_origin_access_identity" "CloudFrontOriginAccessIdentity" {
  comment = "Origin Access Identity for Serverless Static Website"
}

```
To create an AWS CloudFront Origin Access Identity (OAI)


```py
resource "aws_cloudfront_distribution" "WebpageCDN" {
  origin {
    domain_name = "onlinesafety-${var.ID}.s3-website-us-east-1.amazonaws.com"
    origin_id   = "webpage"
    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "match-viewer" # https-only | http-only | match-viewer
      origin_ssl_protocols   = ["SSLv3", "TLSv1", "TLSv1.1", "TLSv1.2"] # to ensure compatibility with a wide range of clients and servers.
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
    viewer_protocol_policy = "allow-all" # changed back from redirect-to-https
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
This section creates the cloudfront distribution.
- `origin`:
  - `domain_name`: Specifies the domain name or endpoint of the origin server. In your configuration.
  - `origin_id`: Is set by the user, is an identifier for the origin. It's used to uniquely identify this origin.
- `enabled`: enabled attribute is set to true, indicating that the CloudFront distribution is enabled and operational.
- `default_root_object`: specifies the default object to serve when a viewer requests the root URL of your distribution.
- `default_cache_behavior`: Cache behavior settings should correpond to your regions' Data Protection Regulation. Here CloudFront does not forward any cookies to the origin server.
- `restrictions`: This is usually none, unless you want to restrict certain places like China or North Korea cause of policies
- `viewer_certificate`: Since we don't have a custom certificate, we will allow AWS Cloudfront provide one for us to browse with HTTPS


```py
output "CloudFrontURL" {
  value       = "https://${aws_cloudfront_distribution.WebpageCDN.domain_name}"
  description = "URL for the CloudFront distribution"
}
```

You can output the value of the cloudfront URL after creation.

We are done with the HCL configuration now let's talk cli commands

### INIT
```sh
terraform init
```
`init` from the word initialization, is often the first command you run when working with terraform. It does the following
- Initializing a Working Directory: Checks your current working directory if your Terraform configuration files is present
- Downloading Provider Plugins: download and install the necessary provider plugins. Providers declared in your configuration files could be AWS, Azure, or Google Cloud
- Initializing Backend Configuration: Backend configuration includes specifying where Terraform should store its state files. In this example they are stored locally. Terraform CLoud is an example of how they could be stored remotely 
- Creating and Initializing Modules: If modules are inlcluded, `terraform init` will also initialize the modules, downloading any required module sources or dependencies.
- Generating Lock Files: generates a terraform.lock.hcl file that records the versions of provider plugins
- Verifying Configuration Files: Terraform checks the syntax and validity of your configuration files

<Put pic here>

### PLAN
```sh
terraform plan
```

- Preview Changes: Detailed preview of the infrastructure changes that Terraform will make (Dry Run)
- Dependency Analysis: Terraform evaluates the dependencies between resources and shows you the order in which changes will be applied.

> Note that terraform will expect that you have a terraform variable file or you directly input it at inline command withe `terraform plan`
>  e.g add your variables to terraform.tfvars as key-value pairs or
> add them at inline like `terraform plan -var "key1=value1" -var "key2=value2"`
<Put pic here>

### APPLY
```sh
terraform apply
```
- Terraform plan sort of runs the `terraform plan` again to outline the changes to be made
- Request for approval
- Once you approve the plan, Terraform proceeds to make the necessary changes to your infrastructure. It communicates with the cloud provider's APIs or other infrastructure management tools to create or modify resources.
- Terraform is designed to be idempotent, meaning it can be run multiple times without causing harm. If the desired state matches the actual state, Terraform takes no action.


### DESTROY
```sh
terraform destroy
```
- If you want to undo changes made by a previous terraform apply, you can use terraform destroy to destroy the created resources. 
- Be cautious when using destroy, as it permanently deletes resources.