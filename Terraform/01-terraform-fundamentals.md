## Whats is Terraform

Terraform is a tool for building, changing, and versioning infrastructure safely and efficiently. Terraform can manage existing, popular service providers and custom in-house solutions.

Configuration files describe to Terraform the components needed to run a single application or your entire data center. Terraform generates an execution plan describing what it will do to reach the desired state, and then executes it to build the described infrastructure. As the configuration changes, Terraform can determine what changed and create incremental execution plans that can be applied.

The infrastructure Terraform can manage includes both low-level components such as compute instances, storage, and networking, and high-level components such as DNS entries and SaaS features.

## Key Features

### Infrastructure as code

Infrastructure is described using a high-level configuration syntax. This allows a blueprint of your data center to be versioned and treated as you would any other code. Additionally, infrastructure can be shared and re-used.

### Execution plans

Terraform has a planning step in which it generates an execution plan. The execution plan shows what Terraform will do when you execute the apply command. This lets you avoid any surprises when Terraform manipulates infrastructure.

### Resource graph

Terraform builds a graph of all your resources and parallelizes the creation and modification of any non-dependent resources. Because of this, Terraform builds infrastructure as efficiently as possible, and operators get insight into dependencies in their infrastructure.

### Change automation

Complex changesets can be applied to your infrastructure with minimal human interaction. With the previously mentioned execution plan and resource graph, you know exactly what Terraform will change and in what order, which helps you avoid many possible human errors.

## Verifying Terraform installation

Pre-installed in Cloud Shell, we can verify it's availability with the following command:
```
terraform
```

## Build infrastructure

### Configuration

The set of files used to describe infrastructure in Terraform is simply known as a Terraform configuration. The format of the configuration files can be found in the [Terraform Language Documentation](https://developer.hashicorp.com/terraform/language). It is recommened to use JSON when creating configuration files.

```
echo 'resource "google_compute_instance" "terraform" {
  project      = "<PROJECT_ID>"
  name         = "terraform"
  machine_type = "n1-standard-1"
  zone         = "us-west1-c"
  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }
  network_interface {
    network = "default"
    access_config {
    }
  }
}' > instance.tf
```

Alternatively, you can create a configuration file using the below command
```
touch instance.tf
```

The configuration file can then be edited by clicking the 'Open Editor' button in Cloud Shell and pasting the below into the file:
```
resource "google_compute_instance" "terraform" {
  project      = "<PROJECT_ID>"
  name         = "terraform"
  machine_type = "n1-standard-1"
  zone         = "us-west1-c"
  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }
  network_interface {
    network = "default"
    access_config {
    }
  }
}
```

This is a complete configuration that Terraform is ready to apply. The general structure should be intuitive and straightforward.

The "resource" block defines a resource that exists within the infrastructure. A resource might be a physical component such as an VM instance.

The resource block has two strings before opening the block: the resource type and the resource name. In the above example, the resource type is google_compute_instance and the name is terraform. The prefix of the type maps to the provider: google_compute_instance automatically tells Terraform that it is managed by the Google provider.

Within the resource block itself is the configuration needed for the resource.

## Initalization

You can use the below command to check what configuration files exit in the current directory:
```
ls
```

It is important to note that Terraform will load all tf files when initializing

The first command to run for a new configuration (or after checking out an existing configuration from version control) is terraform init. This will initialize various local settings and data that will be used by subsequent commands.

Terraform uses a plugin-based architecture to support the numerous infrastructure and service providers available. Each "provider" is its own encapsulated binary that is distributed separately from Terraform itself. The terraform init command will automatically download and install any provider binary for the providers to use within the configuration, which in this case is just the Google provider.

To download the neccessary provider binaries, run the following command:
```
terraform init
```

The next step is to created an execution plan using the below command:
```
terraform plan
```

Terraform performs a refresh, unless explicitly disabled, and then determines what actions are necessary to achieve the desired state specified in the configuration files. This command is a convenient way to check whether the execution plan for a set of changes matches your expectations without making any changes to real resources or to the state. For example, you might run this command before committing a change to version control, to create confidence that it will behave as expected.

## Apply the changes

Run the below command
```
terraform apply
```

This output shows the Execution Plan, which describes the actions Terraform will take in order to change real infrastructure to match the configuration. The output format is similar to the diff format generated by tools like Git.

There is a + next to google_compute_instance.terraform, which means that Terraform will create this resource. Following that are the attributes that will be set. When the value displayed is <computed>, it means that the value won't be known until the resource is created.

If the plan was created successfully, Terraform will now pause and wait for approval before proceeding. At this point <b>No changes have been made to your infrastructure.</b> As such if anything in the Execution Plan seems incorrect it's easy to back out without implementing. If everything looks as expected, type yes to execute the plan and deploy/destroy the resources. This process can take some time. 

Terraform has written some data into the terraform.tfstate file. This state file is extremely important: it keeps track of the IDs of created resources so that Terraform knows what it is managing. We can inspect the current state with the following command
```
terraform show
```