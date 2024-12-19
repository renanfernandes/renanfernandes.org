---
title: "Dynamic DNS on Azure"
date: 2024-12-15T10:48:34-05:00
draft: false
toc: true
---

After I [migrated my DNS records to Azure]({{< ref "azure-dns.md" >}}), a problem came up: I have some machines sitting on a remote location with Dynamic IP addresses. In the past I could have easily use the DynDNS service from NoIP to reach out to these machines whenever I wanted to, but after I moved them over to Azure, I needed to replicate the same functionality somehow. 
This post will show you how to accomplish that. By creating an easy     I always used noip to manage my DNS Zones, but as I keep adding additional domains, it has become expensive. Azure charges only $0.50/month for the first 25 DNS zones, so thats a plus compared to $20+ I pay for NoIP. This guide shows how to create a zone and manage inside Azure.