---
title: "Azure DNS"
date: 2024-03-28T09:08:40-04:00
draft: false
toc: true
---

Eu sempre usei o noip para gerenciar minhas Zonas DNS, mas conforme continuo adicionando domínios adicionais, isso se tornou caro. A Azure cobra apenas $0.50/mês pelas primeiras 25 zonas DNS, então isso é uma vantagem em comparação com os mais de $20 que pago pelo NoIP. Este guia mostra como criar uma zona e gerenciá-la dentro da Azure.

## Criando uma nova Zona DNS
Pesquise por `DNS Zones` na barra de pesquisa e clique em `Create dns zone`.
Selecione um Resource Group existente ou crie um novo para seus recursos.
Nos detalhes da instância, digite o nome de domínio que você comprou de um registrador de domínios e selecione a localização para seu grupo de recursos em *Resource group location*, conforme visto abaixo:

![New DNS Zone](/images/DNS_Zone1.png)

Adicione quaisquer tags aplicáveis e selecione `Create`

## Criando um registro DNS
Agora que você criou sua Zona, você precisa começar a popular os registros DNS. Para este exemplo, eu irei apenas criar um registro `CNAME` para apontar para o meu site. Se você quiser ler mais sobre registros DNS e tipos, vá para este [Artigo da Wikipedia](https://en.wikipedia.org/wiki/List_of_DNS_record_types)

### Criando um registro CNAME:
Dentro da sua Zona DNS, clique em `+ Record set` e preencha os detalhes:
```
Name: The DNS record you want to create. e.g. wwww, mail, etc.
Type: CNAME
Alias Record Set: This is a cool option, you can point your DNS record straight to an Azure resource instead. Use it if you like it, or just do the old fashion Alias way.
TTL: 1 hour
```
![New DNS Record](/images/DNS_Record1.png)

Clique em `Ok` e pronto!

## Etapas Finais
Você está quase lá. As etapas finais estão fora da Azure. Certifique-se de atualizar seu registrador para apontar para os registros DNS da Azure. Você encontrará os Nameserver 1, 2, 3 e 4 dentro da sua zona DNS, conforme visto abaixo:
![Name Servers](/images/DNS_Zone2.png)
Você está pronto! Aproveite sua nova Zona DNS gerenciada na Azure