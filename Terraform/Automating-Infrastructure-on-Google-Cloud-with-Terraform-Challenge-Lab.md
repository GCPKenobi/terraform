## Task 1. Create the configuration files
Run the following in Cloud Shell to create the following Terraform configuration files:
```
mkdir storage
mkdir modules
mkdir modules/instances
touch main.tf variables.tf
cd storage
touch storage.tf outputs.tf variables.tf
cd ..
cd modules/instances
touch instances.tf outputs.tf variables.tf
cd
```

In Cloud Shell editor, edit both ```variables.tf``` files with the following (make sure to replace the values for REGION, ZONE AND PROJECT_ID):
```
variable "region" {
  type        = string
  default     = "<REGION>"
}
variable "zone" {
  type        = string
  default     = "<ZONE>"
}
variable "project_id" {
  description = "Name of the buckets to create."
  type        = string
  default     = "<PROJECT_ID>"
}
```

Edit the ```main.tf``` with the following to add the Google Provider:
```
provider "google" {
  project     = var.project_id
  region      = var.region
  zone        = var.zone
}
```

## Task 2. Import infrastructure

Add the following to the bottom of ```main.tf```:
```
module "instances" {
  source     = "./modules/instances"
}
```

We now update ```instances.tf``` to add the resource configurations for the existing VMs:
```
resource "google_compute_instance" "tf-instance-1" {
  name         = "tf-instance-1"
  machine_type = "n1-standard-1"
  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-10"
    }
  }
  network_interface {
    network = "default"
    access_config {
    }
  }
  metadata_startup_script = <<-EOT
        #!/bin/bash
    EOT
allow_stopping_for_update = true
}

resource "google_compute_instance" "tf-instance-2" {
  name         = "tf-instance-2"
  machine_type = "n1-standard-1"
  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-10"
    }
  }
  network_interface {
    network = "default"
    access_config {
    }
  }
  metadata_startup_script = <<-EOT
        #!/bin/bash
    EOT
allow_stopping_for_update = true
}
```

We can now run the two commands to import the existing VMS into our instances module
```
terraform import module.instances.google_compute_instance.tf-instance-1 tf-instance-1
terraform import module.instances.google_compute_instance.tf-instance-2 tf-instance-2
```

## Task 3. Configure a remote backend

Create the Cloud Storage Bucket resources inside ```storage.tf``` by adding the following (remember to replace the BUCKET_NAME):
```
resource "google_storage_bucket" bucket" {
  name               = "<BUCKET_NAME>"
  location           = "US"
  force_destroy      = true
  uniform_bucket_level_access = true
}
```

Update ```main.tf``` by adding the resource refence by pasting the following into the bottom of the file:
```
module "storage" {
  source     = "./modules/storage"
}
```

Run the following commands to initalize the modeul and create the Cloud Storage Bucket. When prompted type **yes**
```
terraform init
terraform apply
```

We can now update ```main.tf``` by adding the following at the top (remember to replace BUCKET_NAME with your value):
```
terraform{
    backend "gcs" {
        bucket = "<BUCKET_NAME>"
        prefix = "terraform/state"
    }
}
```

We can now run ```terraform init``` to initalize the remote backend. To confirm, type **yes** when prompted

## Task 4. Modify and update infrastructure

Inside the ```instances.tf``` amend the machine type to use ```n1-standard2```
```
resource "google_compute_instance" "tf-instance-1" {
  name         = "tf-instance-1"
  machine_type = "n1-standard-2"
  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-10"
    }
  }
  network_interface {
    network = "default"
    access_config {
    }
  }
  metadata_startup_script = <<-EOT
        #!/bin/bash
    EOT
allow_stopping_for_update = true
}

resource "google_compute_instance" "tf-instance-2" {
  name         = "tf-instance-2"
  machine_type = "n1-standard-2"
  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-10"
    }
  }
  network_interface {
    network = "default"
    access_config {
    }
  }
  metadata_startup_script = <<-EOT
        #!/bin/bash
    EOT
allow_stopping_for_update = true
}
```

We can then add the following at the bottom of the file to create the new VM (replace INSTANCE_NAME with the lab provided value):
```
resource "google_compute_instance" <INSTANCE_NAME>" {
  name         = "<INSTANCE_NAME>"
  machine_type = "n1-standard-2"

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-10"
    }
  }

  network_interface {
 network = "default"
  }

metadata_startup_script = <<-EOT
#!/bin/bash
EOT

allow_stopping_for_update = true
}
```

Run ```terraform apply``` to apply the change


## Task 5. Destroy resources

Go back into ```instance.tf``` and remove the code just added. Run ```terraform apply``` to delete the newly created VM


## Task 6. Use a module from the Registry

Add the following to ```main.tf``` (replace VPC_NAME with the lab provided value):
```
module "vpc" {
    source  = "terraform-google-modules/network/google"
    version = "~> 6.0.0"
    project_id   = var.project_id
    network_name = "<VPC_NAME>"
    routing_mode = "GLOBAL"

    subnets = [
        {
            subnet_name           = "subnet-01"
            subnet_ip             = "10.10.10.0/24"
            subnet_region         = "us-east1"
        },
        {
            subnet_name           = "subnet-02"
            subnet_ip             = "10.10.20.0/24"
            subnet_region         = "us-east1"
        }
    ]
}
```

Run ```terraform init``` and ```terraform apply``` to create the VPC network. Afterwards we can update ```instances.tf``` and change the network values as follows (replace VPC_NAME with the lab provided value):
```
resource "google_compute_instance" "tf-instance-1" {
  name         = "tf-instance-1"
  machine_type = "n1-standard-2"
  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-10"
    }
  }
  network_interface {
    network = "<VPC_NAME>"
    subnetwork = "subnet-01"
    access_config {
    }
  }
  metadata_startup_script = <<-EOT
        #!/bin/bash
    EOT
allow_stopping_for_update = true
}

resource "google_compute_instance" "tf-instance-2" {
  name         = "tf-instance-2"
  machine_type = "n1-standard-2"
  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-10"
    }
  }
  network_interface {
    network = "<VPC_NAME>"
    subnetwork = "subnet-02"
    access_config {
    }
  }
  metadata_startup_script = <<-EOT
        #!/bin/bash
    EOT
allow_stopping_for_update = true
}
```

Run ```terraform apply``` to apply the network changes to the VMs

## Task 7. Configure a firewall

Create the firewall resource in ```main.tf``` be adding the following (replace VPC_NAME with the lab provided value):
```
resource "google_compute_firewall" "tf-firewall" {
  name    = "tf-firewall"
  network = "projects/qwiklabs-gcp-00-4a27c5443ca7/global/networks/<VPC_NAME>"
  allow {
    protocol = "tcp"
    ports    = ["80"]
  }
  source_ranges = ["0.0.0.0/0"]
}
```

Run ```terraform init``` and ```teraform apply``` to create the firewall resource
