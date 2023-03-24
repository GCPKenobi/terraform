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
```json
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
```json
variable "variable_name" {
  description = "Insert description of the variable"
  type        = Insert Type
  default     = "Insert the default value for these variable"
}
```

Where type can take the following values ``` string, list, map, boolean```. (** Note - type is not enclosed in speech marks)
In practice, let's say we identified two input variables, ```project_id``` and ```network_name```. We can add these into the ```variables.tf``` file:
```json
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
```json
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