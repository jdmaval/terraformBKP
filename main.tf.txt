
terraform {

  required_version = ">=0.12"

  required_providers {
    azurerm = {
      source = "hashicorp/azurerm"
      version = "~>2.0"
    }
  }
}


provider "azurerm" {
  features {}
}


#Create Resource Group

resource "azurerm_resource_group" "rg" {
  name = var.resource_group_name
  location = var.resource_group_location
}



#Create Vnet

resource "azurerm_virtual_network" "myterraformnetwork" {
    name                = var.virtual_network_name
    address_space       = var.vnet_address_space
    location            = var.location
    resource_group_name = azurerm_resource_group.rg.name
    depends_on          = [azurerm_resource_group.rg]
}



#Create Subnet

resource "azurerm_subnet" "myterraformsubnet" {
    name                = var.subnet_name
    address_prefixes    = var.address_prefixes
    resource_group_name = azurerm_resource_group.rg.name
    virtual_network_name = var.virtual_network_name
    depends_on           =[azurerm_virtual_network.myterraformnetwork]
}





#Create public IP

resource "azurerm_public_ip" "myterraformpublicip" {
    name                         = var.public_IP_name
    location                     = var.location
    resource_group_name          = azurerm_resource_group.rg.name
    allocation_method            = var.allocation_method
}






resource "azurerm_network_security_group" "myterraformnsg" {
    name                = var.nsg_name
    location            = "eastus"
    resource_group_name = azurerm_resource_group.rg.name

    security_rule {
        name                       = "SSH"
        priority                   = 1001
        direction                  = "Inbound"
        access                     = "Allow"
        protocol                   = "Tcp"
        source_port_range          = "*"
        destination_port_range     = "22"
        source_address_prefix      = "*"
        destination_address_prefix = "*"

       }
}






#Create Network Interface

resource "azurerm_network_interface" "myterraformnic" {

    name                      = var.nic_name
    location                  = var.location
    resource_group_name       = azurerm_resource_group.rg.name

    ip_configuration {
        name                          = "myNicConfiguration"
        subnet_id                     = azurerm_subnet.myterraformsubnet.id
        private_ip_address_allocation = "Dynamic"
        public_ip_address_id          = azurerm_public_ip.myterraformpublicip.id
    }
}







# Create storage account for boot diagnostics

resource "azurerm_storage_account" "mystorageaccount" {
    name                        = "diag${random_id.randomId.hex}"
    resource_group_name         = azurerm_resource_group.rg.name
    location                    = var.location
    account_tier                = var.account_tier
    account_replication_type    = var.account_replication_type

}











#Create Virtual machine

resource "azurerm_linux_virtual_machine" "myterraformvm" {
    name                  = var.vmname
    location              = var.location
    resource_group_name   = azurerm_resource_group.rg.name
    network_interface_ids = [azurerm_network_interface.myterraformnic.id]
    size                  = var.vmsize


    os_disk {
        name              = "myOsDisk"
        caching           = "ReadWrite"
        storage_account_type = "Standard_LRS"
    }

    source_image_reference {
        publisher = "Canonical"
        offer     = "UbuntuServer"
        sku       = "18.04-LTS"
        version   = "latest"
    }


 computer_name  = "myvm"
    admin_username = "azureuser"
    disable_password_authentication = true

    admin_ssh_key {
        username       = "azureuser"
        public_key    = file("~/.ssh/id_rsa.pub")

        #public_key     = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQD8Y28YRMW4jsgU7RvtLhgXZ2JzRfwAtJxb1dMjMDoSgriQX6kuZS25qYn3tzjJdBqhpLz+MCn/SWNh+1SihdaNC36XG6dRaHlPMIfr7XLInc7Jy6oUjxT01IClJzncokLQAITsIAWEy3jUy6uzdvezSpTXgQkA3DLfPMolUai/iHdHssMaqofck2k9RLzU/yyDWGG2zcQEUtSTAh+4AV6wwYXH7QCHcALbpw+XxDEcPDLqwy4hYtkWEFago93FyO9fa92nfclrCuwevYoXHhpEPTk4xTMunmp+7QewIEFZG7Jd4cjXBFz2LSEyPxDOSsfdTHXAhQ7ewH8cK9/BPIH5smQMLN2gL4Hun8oWZpRzrlF9twAsVq56CGpHgwJPaLWPIs79hrVIYTK5D32z/AiQTTnot1u7g5Pv//OfTZ2Zi3Vry2vbEPcjbxY1P44gIr3tg4DiO3kftYr3LNPFVanvUTQaIb8RxWHLxcwFmb6EGJbAR6TKxYKtTqr5ZxkCbEM= diego_maval@EPMXGUAW1378"
    }

    boot_diagnostics {
        storage_account_uri = azurerm_storage_account.mystorageaccount.primary_blob_endpoint
    }
}