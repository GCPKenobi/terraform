## Purpose of Terraform state

State is a necessary requirement for Terraform to function. People sometimes ask whether Terraform can work without state or not use state and just inspect cloud resources on every run. In the scenarios where Terraform may be able to get away without state, doing so would require shifting massive amounts of complexity from one place (state) to another place (the replacement concept). This section will help explain why Terraform state is required.

### Mapping to the real world
Terraform requires some sort of database to map Terraform config to the real world. When your configuration contains a resource resource "google_compute_instance" "foo", Terraform uses this map to know that instance i-abcd1234 is represented by that resource.

Terraform expects that each remote object is bound to only one resource instance, which is normally guaranteed because Terraform is responsible for creating the objects and recording their identities in the state. If you instead import objects that were created outside of Terraform, you must verify that each distinct object is imported to only one resource instance.

If one remote object is bound to two or more resource instances, Terraform may take unexpected actions against those objects because the mapping from configuration to the remote object state has become ambiguous.

### Metadata
In addition to tracking the mappings between resources and remote objects, Terraform must also track metadata such as resource dependencies.

Terraform typically uses the configuration to determine dependency order. However, when you remove a resource from a Terraform configuration, Terraform must know how to delete that resource. Terraform can see that a mapping exists for a resource that is not in your configuration file and plan to destroy. However, because the resource no longer exists, the order cannot be determined from the configuration alone.

To ensure correct operation, Terraform retains a copy of the most recent set of dependencies within the state. Now Terraform can still determine the correct order for destruction from the state when you delete one or more items from the configuration.

This could be avoided if Terraform knew a required ordering between resource types. For example, Terraform could know that servers must be deleted before the subnets they are a part of. The complexity for this approach quickly becomes unmanageable, however: in addition to understanding the ordering semantics of every resource for every cloud, Terraform must also understand the ordering across providers.

Terraform also stores other metadata for similar reasons, such as a pointer to the provider configuration that was most recently used with the resource in situations where multiple aliased providers are present.

### Performance

In addition to basic mapping, Terraform stores a cache of the attribute values for all resources in the state. This is an optional feature of Terraform state and is used only as a performance improvement.

When running a terraform plan, Terraform must know the current state of resources in order to effectively determine the changes needed to reach your desired configuration.

For small infrastructures, Terraform can query your providers and sync the latest attributes from all your resources. This is the default behavior of Terraform: for every plan and apply, Terraform will sync all resources in your state.

For larger infrastructures, querying every resource is too slow. Many cloud providers do not provide APIs to query multiple resources at the same time, and the round trip time for each resource is hundreds of milliseconds. In addition, cloud providers almost always have API rate limiting, so Terraform can only request a limited number of resources in a period of time. Larger users of Terraform frequently use both the -refresh=false flag and the -target flag in order to work around this. In these scenarios, the cached state is treated as the record of truth.

### Syncing

In the default configuration, Terraform stores the state in a file in the current working directory where Terraform was run. This works when you are getting started, but when Terraform is used in a team, it is important for everyone to be working with the same state so that operations will be applied to the same remote objects.

Remote state is the recommended solution to this problem. With a fully featured state backend, Terraform can use remote locking as a measure to avoid multiple different users accidentally running Terraform at the same time; this ensures that each Terraform run begins with the most recent updated state.

### State locking

If supported by your backend, Terraform will lock your state for all operations that could write state. This prevents others from acquiring the lock and potentially corrupting your state.

State locking happens automatically on all operations that could write state. You won't see any message that it is happening. If state locking fails, Terraform will not continue. You can disable state locking for most commands with the -lock flag, but it is not recommended.

If acquiring the lock is taking longer than expected, Terraform will output a status message. If Terraform doesn't output a message, state locking is still occurring.

Not all backends support locking. View the list of backend types for details on whether a backend supports locking.

### Workspaces

Each Terraform configuration has an associated backend that defines how operations are executed and where persistent data such as the Terraform state is stored.

The persistent data stored in the backend belongs to a workspace. Initially the backend has only one workspace, called default, and thus only one Terraform state is associated with that configuration.

Certain backends support multiple named workspaces, which allows multiple states to be associated with a single configuration. The configuration still has only one backend, but multiple distinct instances of that configuration can be deployed without configuring a new backend or changing authentication credentials

## Task 1 Working with backends

A backend in Terraform determines how state is loaded and how an operation such as apply is executed. This abstraction enables non-local file state storage, remote execution, etc.

By default, Terraform uses the "local" backend, which is the normal behavior of Terraform you're used to. This is the backend that was being invoked throughout the previous labs.

Here are some of the benefits of backends:

- Working in a team: Backends can store their state remotely and protect that state with locks to prevent corruption. Some backends, such as Terraform Cloud, even automatically store a history of all state revisions.
- Keeping sensitive information off disk: State is retrieved from backends on demand and only stored in memory.
- Remote operations: For larger infrastructures or certain changes, terraform apply can take a long time. Some backends support remote operations, which enable the operation to execute remotely. You can then turn off your computer, and your operation will still complete. Combined with remote state storage and locking (described above), this also helps in team environments.

### Backends are completely optional

You can successfully use Terraform without ever having to learn or use backends. However, they do solve pain points that afflict teams at a certain scale. If you're working as an individual, you can probably succeed without ever using backends.

Even if you only intend to use the "local" backend, it may be useful to learn about backends because you can also change the behavior of the local backend.

## Add a local backend

When configuring a backend for the first time (moving from no defined backend to explicitly configuring one), Terraform will give you the option to migrate your state to the new backend. This lets you adopt backends without losing any existing state.

Configuring a backend for the first time is no different from changing a configuration in the future: create the new configuration and run terraform init. Terraform will guide you the rest of the way.

Let's begin by using Cloud Shell to create a ```main.tf``` file (be sure to inster your project id where neccessary):
```
echo 'terraform {
  backend "local" {
    path = "terraform/state/terraform.tfstate"
  }
}
provider "google" {
  project     = "qwiklabs-gcp-00-640482df5bfc"
  region      = "us-central-1"
}
resource "google_storage_bucket" "test-bucket-for-state" {
  name        = "qwiklabs-gcp-00-640482df5bfc"
  location    = "US"
  uniform_bucket_level_access = true
}' > main.tf
```

Let's take a look at the backend section of code in the above:
```
terraform {
  backend "local" {
    path = "terraform/state/terraform.tfstate"
  }
}
```

From the path we can see it refrences terraform.tfstate in the directory terraform/state

The local backend stores state on the local filesystem, locks that state using system APIs, and performs operations locally.

Terraform must initialize any configured backend before use. To do this, you will run ```terraform init```. This command should always be run as the first step. It is safe to execute multiple times and performs all the setup actions required for a Terraform environment, including initializing the backend.

The ```init``` command must be called in the following scenarios:

- On any new environment that configures a backend
- On any change of the backend configuration (including type of backend)
- On removing backend configuration completely

It's worht noting that you don't need to remember these exact cases. Terraform will detect when initialization is required and present an error message if you try to use other commands (such as ```terraform apply```)

If we know run:
```
terraform init
terraform apply
terraform show
```

The Cloud Shell Editor should now display the state file called terraform.tfstate in the terraform/state directory. Your google_storage_bucket.test-bucket-for-state resource should be displayed.

## Add a Cloud Storage backend

A Cloud Storage backend stores the state as an object in a configurable prefix in a given bucket on Cloud Storage. This backend also supports state locking. This will lock your state for all operations that could write state. This prevents others from acquiring the lock and potentially corrupting your state.

State locking happens automatically on all operations that could write state. You won't see any message that it is happening. If state locking fails, Terraform will not continue. You can disable state locking for most commands with the -lock flag, but this is not recommended.

Let's update the ```main.tf``` to change the backend from local to Cloud Storage Bucket by replacing the backend section with the following (make sure to replace with your bucket name):
```
terraform {
  backend "gcs" {
    bucket  = "# REPLACE WITH YOUR BUCKET NAME"
    prefix  = "terraform/state"
  }
}
```

Now we can initalize the backend again, but this time automatically migrate the state using the below command:
```
terraform init -migrate-state
```

We can verify it worked by checking the bucket to see the file ```terraform/state/default.tfstate```

### Refresh the state

he terraform refresh command is used to reconcile the state Terraform knows about (via its state file) with the real-world infrastructure. This can be used to detect any drift from the last-known state and to update the state file.

This does not modify infrastructure, but does modify the state file. If the state is changed, this may cause changes to occur during the next plan or apply.

Let's go into the storage bucket, select the checbox next to the ```.tfstate``` file, click add label, and set a key and value then save

We can refresh the state file using the following command:
```
terraform refresh
```

And if we examine it using:
```
terraform show
```

We should see the "key" = "value" label that we defined.

### Workspace cleanup

To clean up our workspace, begin by reverting the backend to local in the ```main.tf```:
```
terraform {
  backend "local" {
    path = "terraform/state/terraform.tfstate"
  }
}
```

And run:
```
terraform init migrate-state
```

Let's edit the ```main.tf``` again by adding the ```force_destroy``` arguement to the cloud bucket resource:
```
resource "google_storage_bucket" "test-bucket-for-state" {
  name        = "qwiklabs-gcp-03-c26136e27648"
  location    = "US"
  uniform_bucket_level_access = true
  force_destroy = true
}
```
This arguement is needed as it ensures that when a bucket is deleted, all the object contained within the bucket will be deleted as well. Without this, terraform will attempt to delete the bucket that contains objects and thus will fail

We can now run the following to clean our workspace;
```
terraform apply
terraform destroy
```

## Task 2. Import Terraform configuration

In this section, you will import an existing Docker container and image into an empty Terraform workspace. By doing so, you will learn strategies and considerations for importing real-world infrastructure into Terraform.

The default Terraform workflow involves creating and managing infrastructure entirely with Terraform.

- Write a Terraform configuration that defines the infrastructure you want to create.
- Review the Terraform plan to ensure that the configuration will result in the expected state and infrastructure.
- Apply the configuration to create your Terraform state and infrastructure.

After you create infrastructure with Terraform, you can update the configuration and plan and apply those changes. Eventually you will use Terraform to destroy the infrastructure when it is no longer needed. This workflow assumes that Terraform will create an entirely new infrastructure.

However, you may need to manage infrastructure that wasn’t created by Terraform. Terraform import solves this problem by loading supported resources into your Terraform workspace’s state.

The import command doesn’t automatically generate the configuration to manage the infrastructure, though. Because of this, importing existing infrastructure into Terraform is a multi-step process.

Bringing existing infrastructure under Terraform’s control involves five main steps:

- Identify the existing infrastructure to be imported.
- Import the infrastructure into your Terraform state.
- Write a Terraform configuration that matches that infrastructure.
- Review the Terraform plan to ensure that the configuration matches the expected state and infrastructure.
- Apply the configuration to update your Terraform state.

### Create Docker Container

Lets begin by creating a container name ```hashicorp-learn``` using the latest GINX image from docker hub.
```
docker run --name hashicorp-learn --detach --publish 8080:80 nginx:latest
```

We can verify the container is running using this command:
```
docker ps
```

Within Cloud Shell, if we Click ```Web Preview``` and then ```Preview on port 8080```, Cloud Shell will open the preview url and we should see the default NGINX index page

### Import container into Terraform

Let's clone the example repo and cd into the directory:
```
git clone https://github.com/hashicorp/learn-terraform-import.git
cd learn-terraform-import
```

This contains two files ```main.tf``` and ```docker.tf```. Run ```terraform init``` commmand before heading into Cloud Shell Editor, open the file ```learn-terraform-import/main.tf``` and it should contain:
```
provider "docker" {
   host    = "npipe:////.//pipe//docker_engine"
}
```

Comment or delete the host line (this is a known workaround for an issue with Docker initilization)

Next lets edit the ```learn-terraform-import/docker.tf``` file. Under the commented out code, let's define an empty ```docker_container``` resource by adding the following:
```
resource "docker_container" "web" {}
```

We can now run the below command to attach the existing docker container to the resource we created above:
```
terraform import docker_container.web $(docker inspect -f {{.ID}} hashicorp-learn)
```

To verify the container has been imported use the ```terraform show``` command

### Create Configuration
Before Terraform can manage the container, we need to create the configuration for it. By running ```terraform plan``` which will reult in errors for the missing required arguements ```image``` and ```name```

There are two approaches to update the configuration in docker.tf to match the state you imported. You can either accept the entire current state of the resource into your configuration as-is or select the required attributes into your configuration individually. Each of these approaches can be useful in different circumstances.

- Using the current state is often faster, but can result in an overly verbose configuration because every attribute is included in the state, whether it is necessary to include in your configuration or not.
- Individually selecting the required attributes can lead to more manageable configuration, but requires you to understand which attributes need to be set in the configuration.

Let's use the current state by copying the state into the ```docker.tf``` file using the below command:
```
terraform show -no-color > docker.tf
```

This will replace the entire contents of the file. If you do not wish to replace everything within the file than you would need to manually merge the configuration in

We can now run ```terraform plan``` which will show warnings and errors about a deprecated argument ('links'), and several read-only arguments (ip_address, network_data, gateway, ip_prefix_length, id).

These read-only arguments are values that Terraform stores in its state for Docker containers but that it cannot set via configuration because they are managed internally by Docker. Terraform can set the links argument with configuration, but still displays a warning because it is deprecated and might not be supported by future versions of the Docker provider.

Because the approach shown here loads all of the attributes represented in Terraform state, your configuration includes optional attributes whose values are the same as their defaults. Which attributes are optional, and their default values, will vary from provider to provider, and are listed in the provider documentation.

The next step is to remove these optional attributes and keep only the required ones: ```image```, ```name```, and ```port```. Once we remove these the configuration will look like this:
```
resource "docker_container" "web" {
    image = "sha256:87a94228f133e2da99cb16d653cd1373c5b4e8689956386c1c12b60a20421a02"
    name  = "hashicorp-learn"
    ports {
        external = 8080
        internal = 80
        ip       = "0.0.0.0"
        protocol = "tcp"
    }
}
```

If unsure what each arguemnet does we can consult the [Docker provider documnetation](https://registry.terraform.io/providers/kreuzwerker/docker/latest/docs/resources/container#links)

Re-running ```terraform plan``` will confirm that the errors are now gone, which means we can now run ```terraform apply```

### Create image resource
In some cases, you can bring resources under Terraform's control without using the terraform import command. This is often the case for resources that are defined by a single unique ID or tag, such as Docker images.

In your docker.tf file, the docker_container.web resource specifies the SHA256 hash ID of the image used to create the container. This is how docker stores the image ID internally, and so terraform import loaded the image ID directly into your state. However the image ID is not as human readable as the image tag or name, and it may not match your intent. For example, you might want to use the latest version of the "nginx" image.

We can retrieve the image's tag name by running the below command (replacing the <IMAGE-ID> with the image ID from ```docker.tf```)
```
docker image inspect <IMAGE-ID> -f {{.RepoTags}}
```

Let's now update the ```docker.tf``` to represent the image as a resource:
```
resource "docker_image" "nginx" {
  name         = "nginx:latest"
}
```

Before we replace the value in the ```docker_container.web``` we should run ```terraform apply```. If we don't, terraform will destroy and recreate the container because the docker_image.ngnix resources hasn't been loaded into state yet.

We can now refrence the image resource in the container:
```
resource "docker_container" "web" {
    image = docker_image.nginx.latest
    name  = "hashicorp-learn"
    ports {
        external = 8080
        internal = 80
        ip       = "0.0.0.0"
        protocol = "tcp"
    }
}
```

Running ```terraform apply``` will result in no changes since the values match

### Manage the container with Terraform

Now that Terraform manages the Docker container, use Terraform to change the configuration.

Let's alter the ```docker.tf``` to change the external port:
```
resource "docker_container" "web" {
  name  = "hashicorp-learn"
  image = docker_image.nginx.latest
  ports {
    external = 8081
    internal = 80
    ip       = "0.0.0.0"
    protocol = "tcp"
  }
}
```

Running ```terraform apply``` will cause the container to be destroyed and recreated with the new port configuration. By running ```docker ps``` we will notice the container ID has changed due to the new container being provisioned and the old one being destroyed.

### Destroy infrastructure

We can clean up the resources by running
```
terraform destroy
```

To verify the container was destroyed we can run the following:
```
docker ps --filter "name=hashicorp-learn"
```

## Limitations and considerations

There are several important things to consider when importing resources into Terraform.

Terraform import can only know the current state of infrastructure as reported by the Terraform provider. It does not know:

- Whether the infrastructure is working correctly.
- The intent of the infrastructure.
- Changes you've made to the infrastructure that aren't controlled by Terraform; for example, the state of a Docker container's filesystem.

Importing involves manual steps which can be error-prone, especially if the person importing resources lacks the context of how and why those resources were created originally.

Importing manipulates the Terraform state file; you may want to create a backup before importing new infrastructure.

Terraform import doesn’t detect or generate relationships between infrastructure.

Terraform doesn’t detect default attributes that don’t need to be set in your configuration.

Not all providers and resources support Terraform import.

Importing infrastructure into Terraform does not mean that it can be destroyed and recreated by Terraform. For example, the imported infrastructure could rely on other unmanaged infrastructure or configuration.

Following infrastructure as code (IaC) best practices such as [immutable infrastructure](https://www.hashicorp.com/resources/what-is-mutable-vs-immutable-infrastructure) can help prevent many of these problems, but infrastructure created manually is unlikely to follow IaC best practices.

Tools such as [Terraformer](https://github.com/GoogleCloudPlatform/terraformer) can automate some manual steps associated with importing infrastructure. However, these tools are not part of Terraform itself and are not endorsed or supported by HashiCorp.