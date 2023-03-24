## What is a module?
A Terraform module is a set of Terraform configuration files in a single directory. Even a simple configuration consisting of a single directory with one or more .tf files is a module. When you run Terraform commands directly from such a directory, it is considered the root module. So in this sense, every Terraform configuration is part of a module.

### Calling modules
Terraform commands will only directly use the configuration files in one directory, which is usually the current working directory. However, your configuration can use module blocks to call modules in other directories. When Terraform encounters a module block, it loads and processes that module's configuration files.

A module that is called by another configuration is sometimes referred to as a "child module" of that configuration.

### Local and remote modules
Modules can be loaded from either the local filesystem or a remote source. Terraform supports a variety of remote sources, including the Terraform Registry, most version control systems, HTTP URLs, and Terraform Cloud or Terraform Enterprise private module registries.

### Module best practices
In many ways, Terraform modules are similar to the concepts of libraries, packages, or modules found in most programming languages, and they provide many of the same benefits. Just like almost any non-trivial computer program, real-world Terraform configurations should almost always use modules to provide the benefits mentioned above.

It is recommended that every Terraform practitioner use modules by following these best practices:

- Start writing your configuration with a plan for modules. Even for slightly complex Terraform configurations managed by a single person, the benefits of using modules outweigh the time it takes to use them properly.

- Use local modules to organize and encapsulate your code. Even if you aren't using or publishing remote modules, organizing your configuration in terms of modules from the beginning will significantly reduce the burden of maintaining and updating your configuration as your infrastructure grows in complexity.

- Use the public Terraform Registry to find useful modules. This way you can quickly and confidently implement your configuration by relying on the work of others.

- Publish and share modules with your team. Most infrastructure is managed by a team of people, and modules are an important tool that teams can use to create and maintain infrastructure. As mentioned earlier, you can publish modules either publicly or privately. You will see how to do this in a later lab in this series.

## Using modules from the Registry

Modules are available from the [Terraform Registry](https://registry.terraform.io/). We will be working with a [module](https://registry.terraform.io/modules/terraform-google-modules/network/google/3.3.0) that will create a VPC Network for us. 

## Create Terraform coniguration

Beging by using Cloud Shell to clone a simple example project, cd into the folder and then checkout a specific branch:
```
git clone https://github.com/terraform-google-modules/terraform-google-network
cd terraform-google-network
git checkout tags/v6.0.1 -b v6.0.1
```

If we look at the ```main.tf``` file we see it contains the following:
```
module "test-vpc-module" {
  source       = "terraform-google-modules/network/google"
  version      = "~> 6.0"
  project_id   = var.project_id # Replace this with your project ID
  network_name = "my-custom-mode-network"
  mtu          = 1460
  subnets = [
    {
      subnet_name   = "subnet-01"
      subnet_ip     = "10.10.10.0/24"
      subnet_region = "us-west1"
    },
    {
      subnet_name           = "subnet-02"
      subnet_ip             = "10.10.20.0/24"
      subnet_region         = "us-west1"
      subnet_private_access = "true"
      subnet_flow_logs      = "true"
    },
    {
      subnet_name               = "subnet-03"
      subnet_ip                 = "10.10.30.0/24"
      subnet_region             = "us-west1"
      subnet_flow_logs          = "true"
      subnet_flow_logs_interval = "INTERVAL_10_MIN"
      subnet_flow_logs_sampling = 0.7
      subnet_flow_logs_metadata = "INCLUDE_ALL_METADATA"
      subnet_flow_logs_filter   = "false"
    }
  ]
}
```

When working with modules, some input variables are required. This means that no default value is provided and as such, an explicit value must be defined (otherwise Terraform will error). Looking at the above ```test-vpc-module``` we can see the following inputs variables:
- ```network_name```: name of the VPC network to create
- ```project_id```: project in which we create the VPC network
- ```subents```: subnets we wish to create

For more information around the inputs, we can refer to the documentation in the [Terraform Registry](https://registry.terraform.io/modules/terraform-google-modules/network/google/3.3.0?tab=inputs)

### Define root input varaibles

To use input variables, identify module input varaibles that you may wish to change in the future, and then create a matching configuration in the ```variables.tf``` with sensible defaults. These variables can then be passed to the module block as arguements. Below shows the syntax of how these variables look:
```
variable "variable_name" {
  description = "Insert description of the variable"
  type        = Insert Type
  default     = "Insert the default value for these variable"
}
```

Where type can take the following values ``` string, list, map, boolean```. (** Note - type is not enclosed in speech marks)
In practice, let's say we identified two input variables, ```project_id``` and ```network_name```. We can add these into the ```variables.tf``` file:
```
variable "project_id" {
  description = "The project ID to host the network in"
  type        =  string
  default     = "FILL IN YOUR PROJECT ID HERE"
}
variable "network_name" {
  description = "The name of the VPC network being created"
  type        = string
  default     = "example-vpc"
}
```

We can then use these variables in the ```main.tf``` file by doing the following:
```
project_id   = var.project_id
network_name = var.network_name
```

### Define root output varaibles

Modules also have output values and are defined within the module with the ```output``` keyword. They can be accessed by referring to them with the syntax ```module.<MODULE NAME>.<OUTPUT NAME>```. Module outputs are usually either passed to other parts of your configuration or defined as outputs in your root module.

Like inputs, we can see them in the [documentation](https://registry.terraform.io/modules/terraform-google-modules/network/google/3.3.0?tab=outputs) under the outputs tab. 

In this module, the ```output.tf``` file looks like this
```
output "network_name" {
  value       = module.test-vpc-module.network_name
  description = "The name of the VPC being created"
}
output "network_self_link" {
  value       = module.test-vpc-module.network_self_link
  description = "The URI of the VPC being created"
}
output "project_id" {
  value       = module.test-vpc-module.project_id
  description = "VPC project id"
}
output "subnets_names" {
  value       = module.test-vpc-module.subnets_names
  description = "The names of the subnets being created"
}
output "subnets_ips" {
  value       = module.test-vpc-module.subnets_ips
  description = "The IP and cidrs of the subnets being created"
}
output "subnets_regions" {
  value       = module.test-vpc-module.subnets_regions
  description = "The region where subnets will be created"
}
output "subnets_private_access" {
  value       = module.test-vpc-module.subnets_private_access
  description = "Whether the subnets will have access to Google API's without a public IP"
}
output "subnets_flow_logs" {
  value       = module.test-vpc-module.subnets_flow_logs
  description = "Whether the subnets will have VPC flow logs enabled"
}
output "subnets_secondary_ranges" {
  value       = module.test-vpc-module.subnets_secondary_ranges
  description = "The secondary ranges associated with these subnets"
}
output "route_names" {
  value       = module.test-vpc-module.route_names
  description = "The routes associated with this VPC"
}
```

We can now run the following commands to make the VPC network:
```
terraform init
terraform apply
```

Due to the ```output.tf``` file, upon Terraform completing the process of provisioning the VPC you will see something similar to the below output
```bash
Outputs:

network_name = "my-custom-mode-network"
network_self_link = "https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-01-7e726fc23fac/global/networks/my-custom-mode-network"
project_id = "qwiklabs-gcp-01-7e726fc23fac"
route_names = []
subnets_flow_logs = [
  false,
  true,
  true,
]
subnets_ips = [
  "10.10.10.0/24",
  "10.10.20.0/24",
  "10.10.30.0/24",
]
subnets_names = [
  "subnet-01",
  "subnet-02",
  "subnet-03",
]
subnets_private_access = [
  false,
  true,
  false,
]
subnets_regions = [
  "us-west1",
  "us-west1",
  "us-west1",
]
subnets_secondary_ranges = [
  tolist([]),
  tolist([]),
  tolist([]),
]
```

## Building your own module

### Module structure

A typical module structure looks like this:

![](https://drive.google.com/uc?export=view&id=1g8xICqsR172UZHgne30BR1FjA2aFAzkl)

Let's take a look at the purpose of each of these files
- ```LICENSE```: contains details about the license which your module will be distributed. This lets the end user know what terms it has been made available under. The file is not used by Terraform
- ```README.md```: a markdown file that contains documentation on how to use the module. Like the above, it isn't used by Terraform
- ```main.tf```: the main set of configurations for the module
- ```variables.tf```: the variable definitions for your module
- ```outputs.tf```: the output definitions for your module

The below files are files that do not need to be including in your module (and as such it's recommended to setup a .gitignore file to avoid tracking changes)
- ```terraform.tfstate```
- ```terraform.tfstate.backup```
- ```.terraform``` directory
- ``` *.tfvars```

## Create a module
In Cloud Shell let's create a directory for the new module and change into the newly created directory:
```
mkdir -p modules/gcs-static-website-bucket
cd modules/gcs-static-website-bucket
```

Let's create the ```LICENSE``` and ```README.md``` files using the following commands:
```
tee -a LICENSE <<EOF
Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at
    http://www.apache.org/licenses/LICENSE-2.0
Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
EOF
```
```
tee -a README.md <<EOF
# GCS static website bucket
This module provisions Cloud Storage buckets configured for static website hosting.
EOF
```

We can great the following command to create the ```website.tf```, ```variables.tf``` and ```outputs.tf``` files:
```
touch website.tf variables.tf outputs.tf
```

Open up the editor and modify these files as such:
```website.tf```
```
resource "google_storage_bucket" "bucket" {
  name               = var.name
  project            = var.project_id
  location           = var.location
  storage_class      = var.storage_class
  labels             = var.labels
  force_destroy      = var.force_destroy
  uniform_bucket_level_access = true
  versioning {
    enabled = var.versioning
  }
  dynamic "retention_policy" {
    for_each = var.retention_policy == null ? [] : [var.retention_policy]
    content {
      is_locked        = var.retention_policy.is_locked
      retention_period = var.retention_policy.retention_period
    }
  }
  dynamic "encryption" {
    for_each = var.encryption == null ? [] : [var.encryption]
    content {
      default_kms_key_name = var.encryption.default_kms_key_name
    }
  }
  dynamic "lifecycle_rule" {
    for_each = var.lifecycle_rules
    content {
      action {
        type          = lifecycle_rule.value.action.type
        storage_class = lookup(lifecycle_rule.value.action, "storage_class", null)
      }
      condition {
        age                   = lookup(lifecycle_rule.value.condition, "age", null)
        created_before        = lookup(lifecycle_rule.value.condition, "created_before", null)
        with_state            = lookup(lifecycle_rule.value.condition, "with_state", null)
        matches_storage_class = lookup(lifecycle_rule.value.condition, "matches_storage_class", null)
        num_newer_versions    = lookup(lifecycle_rule.value.condition, "num_newer_versions", null)
      }
    }
  }
}
```

```variables.tf```
```
variable "name" {
  description = "The name of the bucket."
  type        = string
}
variable "project_id" {
  description = "The ID of the project to create the bucket in."
  type        = string
}
variable "location" {
  description = "The location of the bucket."
  type        = string
}
variable "storage_class" {
  description = "The Storage Class of the new bucket."
  type        = string
  default     = null
}
variable "labels" {
  description = "A set of key/value label pairs to assign to the bucket."
  type        = map(string)
  default     = null
}
variable "bucket_policy_only" {
  description = "Enables Bucket Policy Only access to a bucket."
  type        = bool
  default     = true
}
variable "versioning" {
  description = "While set to true, versioning is fully enabled for this bucket."
  type        = bool
  default     = true
}
variable "force_destroy" {
  description = "When deleting a bucket, this boolean option will delete all contained objects. If false, Terraform will fail to delete buckets which contain objects."
  type        = bool
  default     = true
}
variable "iam_members" {
  description = "The list of IAM members to grant permissions on the bucket."
  type = list(object({
    role   = string
    member = string
  }))
  default = []
}
variable "retention_policy" {
  description = "Configuration of the bucket's data retention policy for how long objects in the bucket should be retained."
  type = object({
    is_locked        = bool
    retention_period = number
  })
  default = null
}
variable "encryption" {
  description = "A Cloud KMS key that will be used to encrypt objects inserted into this bucket"
  type = object({
    default_kms_key_name = string
  })
  default = null
}
variable "lifecycle_rules" {
  description = "The bucket's Lifecycle Rules configuration."
  type = list(object({
    # Object with keys:
    # - type - The type of the action of this Lifecycle Rule. Supported values: Delete and SetStorageClass.
    # - storage_class - (Required if action type is SetStorageClass) The target Storage Class of objects affected by this Lifecycle Rule.
    action = any
    # Object with keys:
    # - age - (Optional) Minimum age of an object in days to satisfy this condition.
    # - created_before - (Optional) Creation date of an object in RFC 3339 (e.g. 2017-06-13) to satisfy this condition.
    # - with_state - (Optional) Match to live and/or archived objects. Supported values include: "LIVE", "ARCHIVED", "ANY".
    # - matches_storage_class - (Optional) Storage Class of objects to satisfy this condition. Supported values include: MULTI_REGIONAL, REGIONAL, NEARLINE, COLDLINE, STANDARD, DURABLE_REDUCED_AVAILABILITY.
    # - num_newer_versions - (Optional) Relevant only for versioned objects. The number of newer versions of an object to satisfy this condition.
    condition = any
  }))
  default = []
}
```

```outputs.tf```
```
output "bucket" {
  description = "The created storage bucket"
  value       = google_storage_bucket.bucket
}
```

We can now go back to the root directory and create the following files:
``` cd ~
touch main.tf outputs.tf varaibles.tf
```

Open up the editor and modify these files as such (make sure for the ```vairables.tf``` to modify the values of variables ```project_id``` and ```name```):
```main.tf```
```
module "gcs-static-website-bucket" {
  source = "./modules/gcs-static-website-bucket"
  name       = var.name
  project_id = var.project_id
  location   = "us-east1"
  lifecycle_rules = [{
    action = {
      type = "Delete"
    }
    condition = {
      age        = 365
      with_state = "ANY"
    }
  }]
}
```

```variables.tf```
```
variable "project_id" {
  description = "The ID of the project in which to provision resources."
  type        = string
  default     = "FILL IN YOUR PROJECT ID HERE"
}
variable "name" {
  description = "Name of the buckets to create."
  type        = string
  default     = "FILL IN A (UNIQUE) BUCKET NAME HERE"
}
```

```outputs.tf```
```
output "bucket-name" {
  description = "Bucket names."
  value       = "module.gcs-static-website-bucket.bucket"
}
```

We can now provisions this static website by running:
```
terraform init
terraform apply
```

Right now there is nothing in our bucket, so nothing to view on the statis website. To test it works lets add some files. Start by cloning the following:
```
cd ~
curl https://raw.githubusercontent.com/hashicorp/learn-terraform-modules/master/modules/aws-s3-static-website-bucket/www/index.html > index.html
curl https://raw.githubusercontent.com/hashicorp/learn-terraform-modules/blob/master/modules/aws-s3-static-website-bucket/www/error.html > error.html
```

Then copy the files over to bucket we created (remember to replace ```YOUR-BUCKET-NAME``` with you newly created bucket name)
```
gsutil cp *.html gs://YOUR-BUCKET-NAME
```

We can then go the the website ```https://storage.cloud.google.com/YOUR-BUCKET-NAME/index.html``` and you should see a basic HTML web page that says **Nothing to see here**(remember to replace ```YOUR-BUCKET-NAME``` with you newly created bucket name)

## Clean Up

Once done, we can remove the provisioned resources by running ```terraform destroy```