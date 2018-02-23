# terraform-azurerm-vnet #
[![Build Status](https://travis-ci.org/Azure/terraform-azurerm-vnet.svg?branch=master)](https://travis-ci.org/Azure/terraform-azurerm-vnet)

Create a basic virtual network in Azure
==============================================================================

This Terraform module deploys a Virtual Network in Azure with a subnet or a set of subnets passed in as input parameters.

The module does not create nor expose a security group. This would need to be defined separately as additional security rules on subnets in the deployed network.

Usage
-----

```hcl
module "vnet" {
    source              = "Azure/vnet/azurerm"
    resource_group_name = "myapp"
    location            = "westus"
    address_space       = "10.0.0.0/16"
    subnet_prefixes     = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
    subnet_names        = ["subnet1", "subnet2", "subnet3"]

    tags                = {
                            environment = "dev"
                            costcenter  = "it"
                          }
}

```

Example adding a network security rule for SSH:
-----------------------------------------------

```hcl
variable "resource_group_name" { }

module "vnet" {
  source              = "Azure/vnet/azurerm"
  resource_group_name = "${var.resource_group_name}"
  location            = "westus"
  address_space       = "10.0.0.0/16"
  subnet_prefixes     = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  subnet_names        = ["subnet1", "subnet2", "subnet3"]

  tags = {
    environment = "dev"
    costcenter  = "it"
  }
}

resource "azurerm_subnet" "subnet" {
  name  = "subnet1"
  address_prefix = "10.0.1.0/24"
  resource_group_name = "${var.resource_group_name}"
  virtual_network_name = "acctvnet"
  network_security_group_id = "${azurerm_network_security_group.ssh.id}"
}

resource "azurerm_network_security_group" "ssh" {
  depends_on          = ["module.vnet"]
  name                = "ssh"
  location            = "westus"
  resource_group_name = "${var.resource_group_name}"

  security_rule {
    name                       = "test123"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "22"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

}
```

Example adding a network security rule for SSH:
-----------------------------------------------
```hcl
variable "resource_group_name" { }

module "vnet" {
  source              = "Azure/vnet/azurerm"
  resource_group_name = "${var.resource_group_name}"
  location            = "westus"
  address_space       = "10.0.0.0/16"
  subnet_prefixes     = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  subnet_names        = ["subnet1", "subnet2", "subnet3"]
  route_tables_ids    = ["azurerm_route_table.rt-subnet1.id", "azurerm_route_table.rt-subnet2.id", "azurerm_route_table.rt-subnet3.id"]

  tags = {
    environment = "dev"
    costcenter  = "it"
  }
}

resource "azurerm_route_table" "rt-subnet1" {
  depends_on          = ["module.vnet"]
  name                = "rt-subnet1
  location            = "westus"
  resource_group_name = "${var.resource_group_name}"
}

resource "azurerm_route" "subnet1_default_gw" {
  name                = "subnet1_default_gw"
  resource_group_name = "${var.resource_group_name}"
  route_table_name    = "${azurerm_route_table.rt-subnet1.name}"
  address_prefix      = "0.0.0.0/0"
  next_hop_type       = "VirtualAppliance"
  next_hop_in_ip_address = "10.0.1.254"
}

resource "azurerm_route_table" "rt-subnet2" {
  depends_on          = ["module.vnet"]
  name                = "rt-subnet2
  location            = "westus"
  resource_group_name = "${var.resource_group_name}"
}

resource "azurerm_route" "subnet2_default_gw" {
  name                = "subnet2_default_gw"
  resource_group_name = "${var.resource_group_name}"
  route_table_name    = "${azurerm_route_table.rt-subnet2.name}"
  address_prefix      = "0.0.0.0/0"
  next_hop_type       = "VirtualAppliance"
  next_hop_in_ip_address = "10.0.2.254"
}

resource "azurerm_route_table" "rt-subnet3" {
  depends_on          = ["module.vnet"]
  name                = "rt-subnet3
  location            = "westus"
  resource_group_name = "${var.resource_group_name}"
}

resource "azurerm_route" "subnet3_default_gw" {
  name                = "subnet3_default_gw"
  resource_group_name = "${var.resource_group_name}"
  route_table_name    = "${azurerm_route_table.rt-subnet3.name}"
  address_prefix      = "0.0.0.0/0"
  next_hop_type       = "VirtualAppliance"
  next_hop_in_ip_address = "10.0.2.254"
}



```


Test
-----
### Configurations
- [Configure Terraform for Azure](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/terraform-install-configure)

We provide 2 ways to build, run, and test module on local dev box:

### Native(Mac/Linux)

#### Prerequisites
- [Ruby **(~> 2.3)**](https://www.ruby-lang.org/en/downloads/)
- [Bundler **(~> 1.15)**](https://bundler.io/)
- [Terraform **(~> 0.11.0)**](https://www.terraform.io/downloads.html)

#### Environment setup
We provide simple script to quickly set up module development environment:
```sh
$ curl -sSL https://raw.githubusercontent.com/Azure/terramodtest/master/tool/env_setup.sh | sudo bash
```
#### Run test
Then simply run it in local shell:
```sh
$ bundle install
$ rake build
$ rake e2e
```

### Docker

We provide a Dockerfile to build a new image based `FROM` the `microsoft/terraform-test` Docker hub image which adds additional tools / packages specific for this module (see Custom Image section).  Alternatively use only the `microsoft/terraform-test` Docker hub image [by using these instructions](https://github.com/Azure/terraform-test).

#### Prerequisites

- [Docker](https://www.docker.com/community-edition#/download)

#### Custom Image

This builds the custom image:

```sh
$ docker build --build-arg BUILD_ARM_SUBSCRIPTION_ID=$ARM_SUBSCRIPTION_ID --build-arg BUILD_ARM_CLIENT_ID=$ARM_CLIENT_ID --build-arg BUILD_ARM_CLIENT_SECRET=$ARM_CLIENT_SECRET --build-arg BUILD_ARM_TENANT_ID=$ARM_TENANT_ID -t azure-network .
```

This runs the build and unit tests:

```sh
$ docker run --rm azure-network /bin/bash -c "bundle install && rake build"
```

This runs the end to end tests:

```sh
$ docker run -v ~/.ssh:/root/.ssh/ --rm azure-network /bin/bash -c "bundle install && rake e2e"
```


Authors
=======
Originally created by [Eugene Chuvyrov](http://github.com/echuvyrov)
