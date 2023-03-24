## Build Base infrastructure

Terraform will recognise files ending in .tf or .tf.json

Lets begin by creating a basic configuration file (main.tf) for a vpc network:
```
echo 'terraform {
  required_providers {
    google = {
      source = "hashicorp/google"
    }
  }
}
provider "google" {
  version = "3.5.0"
  project = "<PROJECT_ID>"
  region  = "us-central1"
  zone    = "us-central1-c"
}
resource "google_compute_network" "vpc_network" {
  name = "terraform-network"
}' > main.tf
```

### Terraform block

This is defined within *terraform {}* and is used in order to ensure tha Terraform knows which provider to download from the [Terraform Registry](https://registry.terraform.io/). n the configuration above, the google provider's source is defined as hashicorp/google which is shorthand for registry.terraform.io/hashicorp/google.

### Providers

The provider block is used to configure the named provider, in this case google. A provider is responsible for creating and managing resources. Multiple provider blocks can exist if a Terraform configuration manages resources from different providers.

## Initialization

We beign with the following command to initalize various settings and data that are utilised by subsequent terraform commands
```
terraform init
```

## Creating resources

We can apply the configuration using the below command:
```
terraform apply
```

You will be prompted before any resources are provisioned, double check everything looks fine and then type yes to proceed

![](https://drive.google.com/uc?export=view&id=1qi6WAturILiXjQrHoR5fDvmG5mdZGzkt)


## Changing infrastructure

Infrastrucute can change and evolve over time and Terraform was built to manage and implement that change. AS we expand upon our configurations, Terraform builds execution plans that only modifies what is neccessary to reach the new desired state.

By using Terraform to implement change to infrastructure, we can implement version control to our configuration files and states to see how our infrastructure has evolved over time

### Adding resources

Adding additional resources is as simple as altering your configuration file. Lets edit our main.tf file to add a VM. We can add the following block of code to the bottom of our file:
```
resource "google_compute_instance" "vm_instance" {
  name         = "terraform-instance"
  machine_type = "f1-micro"
  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }
  network_interface {
    network = google_compute_network.vpc_network.name
    access_config {
    }
  }
}
```

#### Let's break out the components

This has a few more arguements than the basic vpc created, so let's take a look at the structure of what we just added to our script. Starting with name and machine type (which specify what we will call our vm and what type it will be initalised as):
```
name = "terraform-instance"
machine_type = "f1-micro"
```

The next section is boot_disk, which describes what os we are using on our VM:
```
boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }
  ```

Finally, the network interface is defined. 
```
  network_interface {
    network = google_compute_network.vpc_network.name
    access_config {
    }
  }
```
In the above, google_compute_network.vpc_network is the ID, matching the values in the block that defines the network, and name is a property of that resource. The presence of the access_config block, even without any arguments, ensures that the instance will be accessible over the internet.

We can provision this resource using ```terraform apply``` and can see from the output, that the VM is created.

![](https://drive.google.com/uc?export=view&id=1Ro_-MNWjXrcmMhtzkBB8NCkQYJJgQqbm)

### Changing resources

We can also use terraform to modify existing resources. Depending on the severity of the change, the resource may be updated in place (without having to destroy and re-provision the VM).

One example for an in-place change is adding tags to a VM. Let's add some tags by adding the following code to the VM resource:
```
tags = ["web", "dev"]
```

As such, your VM resource block will now look like this:
```
resource "google_compute_instance" "vm_instance" {
  name         = "terraform-instance"
  machine_type = "f1-micro"
  tags         = ["web", "dev"]
  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }
  network_interface {
    network = google_compute_network.vpc_network.name
    access_config {
    }
  }
}
```

If we run again ```terraform apply``` we can see the resource is updated in place, without having to destroy and re-provision the VM

![](https://drive.google.com/uc?export=view&id=1Ro_-MNWjXrcmMhtzkBB8NCkQYJJgQqbm)

### Destructive Changes

If the change is more severe, the change will require the provider to replace the existing resource rather than updating it. This happens when the provider doesn't support updating the resource in the way described in the configuration.

One example would be to change the boot disk of the VM. Let's do that by ammending the boot disk in our configuration to the following:
```
  boot_disk {
    initialize_params {
      image = "cos-cloud/cos-stable"
    }
  }
```

If we run again ```terraform apply`` we can see the resource has to be destroyed and recreated. The exectuion plan will also give details of what required the instance to be re-provisioned.

![](https://drive.google.com/uc?export=view&id=19Mk8qiyhWFPgG1iinZBnURPhyDpUQMP_)
![](https://drive.google.com/uc?export=view&id=19Mk8qiyhWFPgG1iinZBnURPhyDpUQMP_)

### Destroying infrastructure

If you wish to remove all resources from a configuration, this can be accomplished using the following command:
```
terraform destroy
```

![](https://drive.google.com/uc?export=view&id=1G0mYqagLvantOeyIv6SmAZkFfPN0GFI5)

## Resource dependencies

Real-world infrastructure has a diverse set of resources and resource types. Terraform configurations can contain multiple resources, multiple resource types, and these types can even span multiple providers. Let's take a look at how to configure multiple resources and how to use resource attributes to configure other resources.

We can create a static IP address for our VM by creating a google_compute_address resource. We add the following to our configuration file:
```
resource "google_compute_address" "vm_static_ip" {
  name = "terraform-static-ip"
}
```

To assign in to the VM, we alter the network_interface element with the following code:
```
network_interface {
network = google_compute_network.vpc_network.self_link
access_config {
    nat_ip = google_compute_address.vm_static_ip.address
}
}
```

Terraform will read this configuration and do the following:
1. create vm_static_ip before creating the vm_instance
2. Save the properties of vm_static_ip in the state
3. set nat_ip to the value of the properties above

We can view the plan and save it using the following command:
```
terraform plan -out static-ip
```

By saving the plan, we ensure we can apply the plan exactly the same in the future. So let's run this saved plan:
```
terraform apply "static_ip"
```

As expected, the static ip is created first and assigned to the VM network interface. This is because Terraform can infer the dependency between the two resources

### Implicit and explicit dependencies

Implicit dependencies are the primary way to inform Terraform about relationships between resources and should be used where possible. The static IP address above is an example of this as the refence ```google_compute_address.vm_static_ip.address``` creates an implicit dependency on the ```google_compute_address``` named ```vm_static_ip```.

Sometimes a dependency between resources can exist that is not visible to Terraform. For these, we define an explicit dependency using depends_on. The below is an example of a VM which has an explicit dependency to the Cloud Storage Bucket which is also being created. In order to run, make sure to replace <UNIQUE-BUCKET-NAME> with a unique name
```
# The storage bucket our application will use
resource "google_storage_bucket" "example_bucket" {
  name     = "<UNIQUE-BUCKET-NAME>"
  location = "US"
  website {
    main_page_suffix = "index.html"
    not_found_page   = "404.html"
  }
}
# A new instance that uses the above bucket
resource "google_compute_instance" "another_instance" {
  # Tells Terraform that this VM instance must be created only after the
  # storage bucket has been created.
  depends_on = [google_storage_bucket.example_bucket]
  name         = "terraform-instance-2"
  machine_type = "f1-micro"
  boot_disk {
    initialize_params {
      image = "cos-cloud/cos-stable"
    }
  }
  network_interface {
    network = google_compute_network.vpc_network.self_link
    access_config {
    }
  }
}
```

This can be added to the existing configuration file (or a new one), and we can use ```terraform plan``` and ```terraform apply``` to see exactly how its provisioned.

## Define a provisioner

Terraform can also use provisioners to upload files, run shell scripts, or install and trigger other software like configuration management tools.

Before we created a VM based on a Google image. We can use a custom operating system image by using a provider. To do this, we must add a provisioner into our resource code. This can be done as follows:
```
resource "google_compute_instance" "vm_instance" {
  name         = "terraform-instance"
  machine_type = "f1-micro"
  tags         = ["web", "dev"]
  provisioner "local-exec" {
    command = "echo ${google_compute_instance.vm_instance.name}:  ${google_compute_instance.vm_instance.network_interface[0].access_config[0].nat_ip} >> ip_address.txt"
  }
  boot_disk {
    initialize_params {
      image = "cos-cloud/cos-stable"
    }
  }
  network_interface {
    network = google_compute_network.vpc_network.self_link
    access_config {
      nat_ip = google_compute_address.vm_static_ip.address
    }
  }
}
```

In the above example, we use *local-exec* as a provisoner which will execute a command locally on the machine running Terraform (**not the VM we are provisioning**)

If we run ```terraform apply```, Terraform finds nothing to do. This is because provisioners only run when a resource is created. As such in order to see this in action we would have to recreate the resource. We can inform terraform to recreate the instance with the following command:
```
terraform taint google_compute_instance.vm_instance
```

After running the above command, the next time Terraform generates an execution plan, it will remove any tainted resources and re-provision them. This means we can now run ```terraform apply``` to see it in action