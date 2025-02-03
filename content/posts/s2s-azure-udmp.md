---
title: "Building Site to Site VPN with Azure and UDM Pro"
date: 2025-02-01T13:37:16-05:00
draft: true
toc: true
---

I always had a Site to Site VPN connection between our house in US and our Condo in Brazil. While it works great most of the time, the ISP connection in Brazil is not that reliable, adding to the fact that my connection down there runs a dynamic ip. So even having a dynamic dns solution, sometimes its a pain.

To improve the reliability of my Labs, I decided to start moving some of my resources to Azure, start adopting Azure ARC to manage my local servers, etc. So why not set up a Site to Site VPN between my two sites and Azure? If you are looking to set up a S2S between your Ubiquiti 

## Setting up - Azure Side
[Microsoft Learn](https://learn.microsoft.com/en-us/azure/vpn-gateway/tutorial-site-to-site-portal) has a great guide on how to Create a Site-to-Site VPN connection in the Azure Portal. To summarize what I did:

### Setting up the Virtual Network
First step is to create a new Virtual Network in the Azure Portal. Pretty straightforward, select your Subscription and Resource Group and define a Network Range. In my case, i used `10.1.0.0/16` as I'm a `10.0.0.0` guy

![Azure Virtual Network Config](/images/s2s_azure.png)

Once your network is deployed, you need to move to the next step, which is creating a new VPN Gateway

