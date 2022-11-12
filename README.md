# 4640-trfrm2
Configure Terraform
create a new API token on Digitalocean
create a .env file in the dev directory
copy your token and save it to the .env file (export TF_VAR_do_token=<my API tocken>)
source your .env file using ``` source .env ```

Create 3 files having following configuration in directory dev
main.tf 
```
terraform {
  required_providers {
    digitalocean = {
      source  = "digitalocean/digitalocean"
      version = "~> 2.0"
    }
  }
}


# Configure the DigitalOcean Provider
provider "digitalocean" {
  token = var.do_token
}

data "digitalocean_ssh_key" "ACIT-4640" {
  name = "ACIT-4640"
}

data "digitalocean_project" "lab_project" {
  name = "first-project"
}


# Create a new tag
# Use tag to create load balancer
resource "digitalocean_tag" "do_tag" {
  name = "Web"
}

# Create a new VPC
resource "digitalocean_vpc" "web_vpc" {
  name   = "web"
  region = var.region
}

# Create a new Web Droplet in the sfo3 region
resource "digitalocean_droplet" "web" {
  image    = "rockylinux-9-x64"
  count    = var.droplet_count
  name     = "web-${count.index + 1}"
  region   = var.region
  size     = "s-1vcpu-512mb-10gb"
  tags     = [digitalocean_tag.do_tag.id]
  ssh_keys = [data.digitalocean_ssh_key.ACIT-4640.id]
  vpc_uuid = digitalocean_vpc.web_vpc.id

  lifecycle {
    create_before_destroy = true
  }
}

# Add new web-1 droplet to existing 4640_Labs project
resource "digitalocean_project_resources" "project_attach" {
  project = data.digitalocean_project.lab_project.id
  resources = flatten([ digitalocean_droplet.web.*.urn ])
}

resource "digitalocean_loadbalancer" "public" {
  name = "loadbalancer-1"
  region = var.region

  forwarding_rule {
    entry_port = 80
    entry_protocol = "http"

    target_port = 80
    target_protocol = "http"
  }

  healthcheck {
    port = 22
    protocol = "tcp"
  }

  droplet_tag = "Web"
  vpc_uuid = digitalocean_vpc.web_vpc.id
}


output "server_ip" {
  value = digitalocean_droplet.web.*.ipv4_address
}
```
terraform.tfvars
```
#set the number of droplets to create
droplet_count = 3
```
variable.tf

```
#Terraform varibales
#The API token
variable "do_token" {}

#Set the default region to sfo3
variable "region" {
 type = string
 default = "sfo3"
}

#Set the default for droplet count
variable "droplet_count"{
 type = number
 default = 2
}
```


Create a another directory and create 2 files in it with following configuration.
ansible.cfg
 ```
 [defaults]
host_key_checking = False
inventory = inventory

[inventory]
enabled_plugins = ini, community.digitalocean.digitalocean
```
nginx_setup.yml
```
---
- name: install and enable nginx
  hosts: webserver
  tasks:
    - name: install nginx
      package:
        name: nginx
        state: present
    - name: enable and start nginx
      service:
        name: nginx
        enabled: yes
        state: started
```

create a file in the dev directory called "do_token"
copy the same token from last step and save it to the do_token file (export DO_API_TOKEN=<my API tocken>)
source the do_token file by running ``` source do_token ```


Create a inventory directory in mgnt and create a file inside it 
hosts.digitalocean.yml
```
plugin: community.digitalocean.digitalocean
attributes:
    - id
    - name
    - memory
    - vcpus
    - disk
    - size
    - image
    - networks
    - volume_ids
    - tags
    - region

groups:
    webserver: "'Web' in (do_tags)"
compose:
    ansible_host: do_networks.v4 | selectattr('type','eq','public')
              | map(attribute='ip_address') | first
 ```
 
 After that your tree structure should look like this. 
 ![image](https://user-images.githubusercontent.com/78824700/201452830-3519003b-ffea-4b16-828a-224d80d820de.png)
 
 In dev directory run ``` terraform validate ```
![image](https://user-images.githubusercontent.com/78824700/201452870-d8a6241f-79dd-4a54-a212-ca78be92a6d5.png)

then run ``` terraform apply ``

After this terraform has created 3 droplets and 1 loadbalancer in digital ocean. 

go to mgnt directory and run ``` ansible -m ping -u root webserver ```
![image](https://user-images.githubusercontent.com/78824700/201452487-f286f511-09af-497f-97b9-a8e3fe52affd.png)

and then run ``` ansible-playbook nginx_setup.yml -u root ```
Go to web browser and check the ip address for loadbalancer 
![image](https://user-images.githubusercontent.com/78824700/201452495-1593d47d-5972-4efa-a0a0-10230ce6393d.png)
