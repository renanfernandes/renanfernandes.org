---
title: "Keep Your DNS Up-to-Date with Azure: A Dynamic DNS Script for Linux"
date: 2025-02-03T11:36:42-05:00
draft: false
toc: true
---

Keeping your DNS records current is a must if you're running services on a dynamic IP address. 
After I moved my [DNS Zone from NoIP to Azure ]({{< ref "/posts/azure-dns/" >}}), I needed to have a reliable way to update the DNS record of one of my machines overseas. As theres no "native" dyndns client solution for Azure, the task was to build my own. Which is not as bad as it sounds.

In this post, I'll introduce a Python script that automatically updates your Azure DNS record with your machine’s latest public IP address. The best part? The configuration is secure—sensitive details like your Azure subscription and DNS settings are stored in environment variables. You can check out the full code on my GitHub repository: [https://github.com/renanfernandes/my-stuff/blob/main/scripts/azure_ddns_updater.py](https://github.com/renanfernandes/my-stuff/blob/main/scripts/azure_ddns_updater.py).

## Script Overview

The script leverages the [Azure SDK for Python](https://learn.microsoft.com/azure/developer/python/) to interact with Azure DNS. It fetches your public IP using [ipify](https://www.ipify.org) and updates the DNS record if necessary. Sensitive configuration values are read from environment variables, which adds a layer of security by keeping credentials out of the code.

Below is the updated and secure version of the script:

```python
#!/usr/bin/env python3
"""
azure_ddns_updater.py: A dynamic DNS updater for Azure DNS.

This script retrieves your current public IP address and updates
the specified Azure DNS A record if it has changed.
"""

import sys
import os
import requests
from azure.identity import DefaultAzureCredential
from azure.mgmt.dns import DnsManagementClient
from azure.mgmt.dns.models import RecordSet, ARecord
from azure.core.exceptions import ResourceNotFoundError

# Retrieve configuration from environment variables.
SUBSCRIPTION_ID = os.getenv('SUBSCRIPTION_ID')
RESOURCE_GROUP = os.getenv('RESOURCE_GROUP')
DNS_ZONE = os.getenv('DNS_ZONE')
RECORD_NAME = os.getenv('RECORD_NAME')
TTL = 300  # Time-to-live in seconds

# Validate that all required environment variables are set.
required_vars = {
    'SUBSCRIPTION_ID': SUBSCRIPTION_ID,
    'RESOURCE_GROUP': RESOURCE_GROUP,
    'DNS_ZONE': DNS_ZONE,
    'RECORD_NAME': RECORD_NAME,
}

for var_name, value in required_vars.items():
    if value is None:
        print(f"Error: The environment variable {var_name} is not set.")
        sys.exit(1)

def get_public_ip():
    """Retrieve the current public IP address."""
    try:
        response = requests.get("https://api.ipify.org")
        response.raise_for_status()
        ip = response.text.strip()
        if not ip:
            raise ValueError("Empty IP address received.")
        return ip
    except Exception as e:
        print(f"Error retrieving public IP: {e}")
        sys.exit(1)

def main():
    # Step 1. Retrieve the current public IP.
    current_ip = get_public_ip()
    print(f"Current public IP: {current_ip}")

    # Step 2. Create Azure credentials and a DNS management client.
    try:
        credential = DefaultAzureCredential()
    except Exception as e:
        print(f"Error creating default credentials: {e}")
        sys.exit(1)

    dns_client = DnsManagementClient(credential, SUBSCRIPTION_ID)

    # Step 3. Attempt to retrieve the existing A record set.
    try:
        record_set = dns_client.record_sets.get(
            RESOURCE_GROUP,
            DNS_ZONE,
            RECORD_NAME,
            "A"
        )
        # Extract existing IP addresses from the record set.
        existing_ips = [record.ipv4_address for record in record_set.a_records] if record_set.a_records else []
        if current_ip in existing_ips:
            print("IP addresses match. No update needed.")
            sys.exit(0)
        else:
            print(f"Existing DNS A record IPs: {existing_ips}")
            print("IP addresses differ. Updating record.")
    except ResourceNotFoundError:
        print("No existing A record found. A new record set will be created.")

    # Step 4. Update (or create) the DNS record with the current public IP.
    new_a_record = ARecord(ipv4_address=current_ip)
    record_set_params = RecordSet(
        ttl=TTL,
        a_records=[new_a_record]
    )

    try:
        dns_client.record_sets.create_or_update(
            RESOURCE_GROUP,
            DNS_ZONE,
            RECORD_NAME,
            "A",
            record_set_params
        )
        print(f"DNS record updated successfully to IP: {current_ip}")
    except Exception as e:
        print(f"Error updating DNS record: {e}")
        sys.exit(1)

if __name__ == "__main__":
    main()
```
## Setting Up Your Environment Variables
Before running the script, ensure you set the following environment variables:
	•	SUBSCRIPTION_ID: Your Azure subscription ID.
	•	RESOURCE_GROUP: The Azure resource group where your DNS zone is located.
	•	DNS_ZONE: The DNS zone (e.g., example.com).
	•	RECORD_NAME: The DNS record name (e.g., home for home.example.com).

For example, on a Linux machine you can export these variables in your shell profile or in a dedicated configuration file, such as /etc/environment
```
export SUBSCRIPTION_ID="your-azure-subscription-id"
export RESOURCE_GROUP="your-resource-group"
export DNS_ZONE="your-dns-zone.com"
export RECORD_NAME="your-record-name"
```

## Running the script

And thats all!
Once you’ve set the environment variables and installed the azure sdk, simply run the script:

````
renanfernandes@coronel:~$ chmod +x azure_ddns_updater.py
renanfernandes@coronel:~$ ./azure_ddns_updater.py
Current public IP: XX.XX.XX.239
Existing DNS A record IPs: ['XX.XX.XX.33']
IP addresses differ. Updating record.
DNS record updated successfully to IP: XX.XX.XX.239
renanfernandes@coronel:~$
````

For continuous updates, consider scheduling the script with a cron job. For example, to run it every hour, add the following line to your crontab:

```
0 * * * * /usr/bin/env python3 /path/to/azure_ddns_updater.py >> /tmp/azure_ddns_updater.log 2>&1
```

## Conclusion
Thats all!

Interested in the full code or contributing improvements? Check out my GitHub repository: https://github.com/renanfernandes/my-stuff.

Give it a try and let me know your thoughts!
