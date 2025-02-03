---
title: "Keep Your DNS Up-to-Date with Azure: A Dynamic DNS Script for Linux"
date: 2025-02-03T11:36:42-05:00
draft: false
toc: true
---

Manter seus registros DNS atualizados é essencial se você está executando serviços com um endereço IP dinâmico.  
Após mover minha [Zona DNS do NoIP para o Azure]({{< ref "/posts/azure-dns/" >}}), eu precisei de uma maneira confiável de atualizar o registro DNS de uma das minhas máquinas no exterior. Como não existe uma solução "nativa" de cliente dyndns para o Azure, a tarefa foi criar a minha própria. O que não é tão ruim quanto parece.

Neste post, vou apresentar um script em Python que atualiza automaticamente seu registro DNS do Azure com o último endereço IP público da sua máquina. A melhor parte? A configuração é segura—detalhes sensíveis como sua assinatura do Azure e as configurações de DNS são armazenados em variáveis de ambiente. Você pode conferir o código completo no meu repositório no GitHub: [https://github.com/renanfernandes/my-stuff/blob/main/scripts/azure_ddns_updater.py](https://github.com/renanfernandes/my-stuff/blob/main/scripts/azure_ddns_updater.py).

## Visão Geral do Script

O script utiliza o [Azure SDK para Python](https://learn.microsoft.com/azure/developer/python/) para interagir com o Azure DNS. Ele obtém seu IP público usando [ipify](https://www.ipify.org) e atualiza o registro DNS se necessário. Os valores de configuração sensíveis são lidos a partir de variáveis de ambiente, o que adiciona uma camada de segurança ao manter as credenciais fora do código.

Abaixo está a versão atualizada e segura do script:

```python
#!/usr/bin/env python3
"""
azure_ddns_updater.py: A dynamic DNS updater for Azure DNS.

This script retrieves your current public IP address and updates
the specified Azure DNS A record if it has changed.

Changelog:
- 2025-02-03:
    - Initial version created.
    - Fixed property name from `arecords` to `a_records` for compatibility with the current Azure SDK.
    - Added configuration via environment variables for sensitive details (SUBSCRIPTION_ID, RESOURCE_GROUP, DNS_ZONE, RECORD_NAME).
    - Implemented validation for required environment variables.
    - Enhanced error handling and logging.
    - Added log_message() function to include timestamps in log messages.
"""

import sys
import os
import requests
from datetime import datetime
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

def log_message(message):
    timestamp = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
    print(f"[{timestamp}] {message}")

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
        log_message(f"Error retrieving public IP: {e}")
        sys.exit(1)

def main():
    # Step 1. Retrieve the current public IP.
    current_ip = get_public_ip()
    log_message(f"Current public IP: {current_ip}")

    # Step 2. Create Azure credentials and a DNS management client.
    try:
        credential = DefaultAzureCredential()
    except Exception as e:
        log_message(f"Error creating default credentials: {e}")
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
            log_message("IP addresses match. No update needed.")
            sys.exit(0)
        else:
            log_message(f"Existing DNS A record IPs: {existing_ips}")
            log_message("IP addresses differ. Updating record.")
    except ResourceNotFoundError:
        log_message("No existing A record found. A new record set will be created.")

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
        log_message(f"DNS record updated successfully to IP: {current_ip}")
    except Exception as e:
        log_message(f"Error updating DNS record: {e}")
        sys.exit(1)

if __name__ == "__main__":
    main()
```
## Configurando Suas Variáveis de Ambiente

Antes de executar o script, certifique-se de definir as seguintes variáveis de ambiente:
	•	SUBSCRIPTION_ID: Seu ID de assinatura do Azure.
	•	RESOURCE_GROUP: O grupo de recursos do Azure onde sua zona DNS está localizada.
	•	DNS_ZONE: A zona DNS (ex.: example.com).
	•	RECORD_NAME: O nome do registro DNS (ex.: home para home.example.com).
    
Por exemplo, em uma máquina Linux você pode exportar essas variáveis no seu perfil de shell ou em um arquivo de configuração dedicado, como /etc/environment:

```
export SUBSCRIPTION_ID="your-azure-subscription-id"
export RESOURCE_GROUP="your-resource-group"
export DNS_ZONE="your-dns-zone.com"
export RECORD_NAME="your-record-name"
```

## Executando o Script

E é isso!
Depois de configurar as variáveis de ambiente e instalar o SDK do Azure, basta executar o script:

````
renanfernandes@coronel:~$ chmod +x azure_ddns_updater.py
renanfernandes@coronel:~$ ./azure_ddns_updater.py
Current public IP: XX.XX.XX.239
Existing DNS A record IPs: ['XX.XX.XX.33']
IP addresses differ. Updating record.
DNS record updated successfully to IP: XX.XX.XX.239
renanfernandes@coronel:~$
````

Para executar automáticamente, considere agendar o script com um cron job. Por exemplo, para executá-lo a cada hora, adicione a seguinte linha ao seu crontab:

```
0 * * * * /usr/bin/env python3 /path/to/azure_ddns_updater.py >> /tmp/azure_ddns_updater.log 2>&1
```

## Conclusão

É isso!

Interessado no código completo ou em contribuir com melhorias? Confira meu repositório no GitHub: https://github.com/renanfernandes/my-stuff.


