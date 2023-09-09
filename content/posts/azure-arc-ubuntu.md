---
title: "Azure Arc Ubuntu"
date: 2023-09-08T11:32:06-04:00
draft: false
---
Following the Wireguard series, I wanted to modernize my lab. one quick way to address that is to extend my two networks via Azure Arc so I can manage my resources more adequadely

Table of Contents
---

* [Installing Azure CLI](#installing-azure-cli)
* [Creating the deployment script via Azure Arc Portal](#creating-the-deployment-script-via-azure-arc-portal)
* [Running the onboarding script](#running-the-onboarding-script)

## Installing Azure CLI
The steps to install Azure CLI are pretty well described ![here](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) so I wont take time to do this. however, make sure you have the latest Azure CLI installed.

One note is that by default, az login will span a web browser to authenticate. If you don't have any available in your setup, you can use `az login --use-device-code`, which will give you a device code so you can log from any other machine (like when you connect your Streaming Apps/Credentials). Pretty clever.


## Creating the deployment script via Azure Arc Portal
In Azure Portal, Type Azure Arc, then click on "Add" 
![Azure Arc Add Infrastructure ](/images/Azure_Arc_Add.png)

Then 
> Servers>Add a Single Server

and fill in the details for your machine.

```
Resource Group: Create new or select existing
Region: Select Region Closest to your server
Operating System: Linux
```
You can also add Physical Location tags to define where the server is located. After you finish you can download the script

## Running the onboarding script 

Download the file to your server, give permissions and run

This script will do the following :
1. Download an installation script from the Microsoft Download Center.
2. Configure the package manager to use and trust the packages.microsoft.com repository.
3. Download the agent from Microsoft's Linux Software Repository.
4. Install the agent on the server.
5. Create the Azure Arc-enabled server resource and associate it with the agent.

Note that during the installation the agent will span a web browser to authenticate. If you don't have any available in your setup, it will give you a device code so you can log from any other machine (like when you connect your Streaming Apps/Credentials). So be on the lookout for any outputs from the script.

If all goes well, the script will connect the machine to Azure and provide you a link for the machine overview page

![Azure Arc Machines](/images/Azure_Arc_Machines.png)