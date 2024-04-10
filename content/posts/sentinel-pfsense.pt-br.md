---
title: "Conectando seu pfSense ao Microsoft Sentinel"
date: 2023-08-19T14:55:27-04:00
draft: false
toc: true
---


Apenas uma coleção de modelos, scripts, arquivos de configuração e observações de teste para facilitar minha vida. Isso faz parte do meu próprio repositório no GitHub (e um dos poucos que eu torno público), então use o conteúdo e scripts aqui por sua própria conta e risco :)

Integrando Sentinel e a Infraestrutura Residencial
À medida que minha rede residencial continua a crescer e mais dispositivos IoT são adicionados, senti a necessidade de melhorar minha postura de segurança residencial e, francamente, ter uma melhor visibilidade sobre o que está acontecendo em minha rede.

Para começar, o diagrama abaixo mostra como minha rede residencial está atualmente configurada. Sem entrar em muitos detalhes, tenho três VLANs, 1) IoT, 2) Rede Confiável, 3) Servidores e 4) Rede Não Confiável/Hóspedes.

![Diagrama de Rede](/images/diagram.png)

Minha VLAN de IoT tem acesso à internet, mas não tem acesso à minha VLAN Confiável, exceto para a transmissão mdns que, graças ao [Avahi](http://avahi.org), me permite controlar meus dispositivos IoT quando estou em casa.
Tudo isso é controlado pelo pfSense rodando em meu Netgate SG-1100. Estou expandindo a rede, então após as festas de fim de ano, substituirei o SG-1100 por algo um pouco mais poderoso.

A ideia é começar a enviar meus logs do pfSense para o [Microsoft Sentinel](https://azure.microsoft.com/pt-br/services/azure-sentinel/), onde posso monitorar melhor minha rede e detectar atividades suspeitas.

O Microsoft Sentinel fornece um modelo de [preço](https://azure.microsoft.com/pt-br/pricing/details/azure-sentinel/) Pay-As-You-Go atualmente a US$ 2,46 por GB ingerido, o que deve manter os custos mínimos para meu uso residencial e me dar muita flexibilidade para criar regras, orquestração e automação. Portanto, é uma escolha óbvia :)

Ok! Para reunir tudo, criei este rascunho para mostrar o que estarei implementando como minha solução final:

![Arquitetura de Rede](/images/architecture.png)

Implantarei um servidor Syslog no Azure, permitindo-me dimensionar a solução e encaminhar eventos de syslog de alguns servidores-chave e aplicativos que hospedo localmente.
Meus dados de syslog serão armazenados no Log Analytics Workspace que, no final, será adicionado ao Sentinel para ingestão e análise.

Então vamos lá!

*Aviso Legal: Este guia é apenas uma coleção e notas da minha experiência neste projeto divertido. Usei vários recursos e referências para construir este guia, como: este incrível [Post da Comunidade Técnica da Microsoft](https://techcommunity.microsoft.com/t5/microsoft-sentinel/pfsense-syslog-to-azure-sentinel-guide/m-p/2004352) e este ótimo [repositório](https://github.com/noodlemctwoodle/pf-azure-sentinel/tree/main/Logstash-Configuration) mantido por noodlemctwoodle. Atualizei as instruções abaixo com as últimas configurações e alterações necessárias.*

## Servidor Syslog
Antes de tudo, precisamos de um local para hospedar nosso servidor Syslog. A maneira como funciona é que seu Firewall (no meu caso, meu Netgate SG-1100) enviará seus logs para o servidor Syslog, que eventualmente encaminhará os logs para o Azure Sentinel.

Você pode executar um servidor Windows ou Linux para seu servidor Syslog e pode hospedar localmente ou na nuvem. Como meu objetivo final é enviar os logs para o Sentinel, decidi criar uma VM Linux e hospedar no Azure.

Isso dará espaço para expandir e me permitirá conectar meus servidores no exterior a esta infraestrutura, mas isso é assunto para outro dia.

### Provisionamento da VM
Como não estou fazendo nada complicado e tudo o que preciso é coletar syslog, provisionei uma única VM com Ubuntu 21.10 e usei o tamanho **Standard_B2s**. Isso me dá 2 vCPUs e 4 GB de RAM na Região East US 2. Custando aproximadamente US$ 30,37/mês, não é ruim!

![Configuração do Syslog](/images/syslog-azure1.png)

Agora que sua VM está pronta, é hora de configurar o Servidor Rsyslog.

### Configurando o Servidor Syslog
Primeiro as coisas mais básicas... Certifique-se de ter a versão mais atualizada de seus pacotes e SO executando `apt update` e `apt upgrade`:
```
sudo apt update
sudo apt upgrade
```

Por uma questão de consistência, vamos garantir que seu fuso horário esteja correto. No meu caso, America/New_York:

```
sudo timedatectl set-timezone America/New_York
```

**Aviso Legal 2!!**
A partir de agora, confiarei neste incrível [Post da Comunidade Técnica da Microsoft](https://techcommunity.microsoft.com/t5/microsoft-sentinel/pfsense-syslog-to-azure-sentinel-guide/m-p/2004352) para configurar e instalar o Elastic. Você pode seguir o guia no link ou consultar minhas instruções abaixo, que fornecem algumas poucas alterações na configuração.

Ok, voltando às instruções. Primeiro de tudo, precisaremos instalar o Logstash. Para isso, primeiro, baixe e instale a chave de assinatura GPG pública:

```
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
```

Em seguida, precisamos instalar o apt-transport-https para permitir que o apt acesse repositórios https (neste caso, Elastic)


```
sudo apt-get install apt-transport-https
```

Por fim, adicionaremos os Repositórios da Elastic

```
echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
```

*Aviso: O ElasticSearch 6.8.9+ e 7.8.9+ não são vulneráveis à [vulnerabilidade CVE-2021-44228 do Apache Log4j2](https://discuss.elastic.co/t/apache-log4j2-remote-code-execution-rce-vulnerability-cve-2021-44228-esa-2021-31/291476), portanto, certifique-se de estar instalando a versão suportada mais recente.*

Agora vamos atualizar o repositório apt e instalar o Logstash:

```
sudo apt update
sudo apt install logstash
```

Se tudo correr bem, o Logstash será instalado em breve. Você pode verificar o status do logstash verificando o status do daemon:

```
sudo service logstash status
```

Agora vamos configurar o Logstash e aplicar o padrão grok.

Primeiro, crie os seguintes diretórios:

```
sudo mkdir /etc/logstash/conf.d/{databases,patterns,templates}
```

Em seguida, baixe os arquivos de configuração deste incrível [repositório](https://github.com/noodlemctwoodle/pf-azure-sentinel/tree/main/Logstash-Configuration) mantido por noodlemctwoodle

```
sudo wget https://raw.githubusercontent.com/noodlemctwoodle/pfsense-azure-sentinel/main/Logstash-Configuration/etc/logstash/conf.d/01-inputs.conf -PO /etc/logstash/conf.d/
sudo wget https://raw.githubusercontent.com/noodlemctwoodle/pfsense-azure-sentinel/main/Logstash-Configuration/etc/logstash/conf.d/02-types.conf -PO /etc/logstash/conf.d/
sudo wget https://raw.githubusercontent.com/noodlemctwoodle/pfsense-azure-sentinel/main/Logstash-Configuration/etc/logstash/conf.d/03-filter.conf -PO /etc/logstash/conf.d/
sudo wget https://raw.githubusercontent.com/noodlemctwoodle/pfsense-azure-sentinel/main/Logstash-Configuration/etc/logstash/conf.d/05-apps.conf -PO /etc/logstash/conf.d/
sudo wget https://raw.githubusercontent.com/noodlemctwoodle/pfsense-azure-sentinel/main/Logstash-Configuration/etc/logstash/conf.d/20-interfaces.conf -PO /etc/logstash/conf.d/
sudo wget https://raw.githubusercontent.com/noodlemctwoodle/pfsense-azure-sentinel/main/Logstash-Configuration/etc/logstash/conf.d/30-geoip.conf -PO /etc/logstash/conf.d/
sudo wget https://raw.githubusercontent.com/noodlemctwoodle/pfsense-azure-sentinel/main/Logstash-Configuration/etc/logstash/conf.d/35-rules-desc.conf -PO /etc/logstash/conf.d/
sudo wget https://raw.githubusercontent.com/noodlemctwoodle/pfsense-azure-sentinel/main/Logstash-Configuration/etc/logstash/conf.d/37-enhanced_user_agent.conf -PO /etc/logstash/conf.d/
sudo wget https://raw.githubusercontent.com/noodlemctwoodle/pfsense-azure-sentinel/main/Logstash-Configuration/etc/logstash/conf.d/38-enhanced_url.conf -PO /etc/logstash/conf.d/
sudo wget https://raw.githubusercontent.com/noodlemctwoodle/pfsense-azure-sentinel/main/Logstash-Configuration/etc/logstash/conf.d/45-cleanup.conf -PO /etc/logstash/conf.d/
sudo wget https://raw.githubusercontent.com/noodlemctwoodle/pfsense-azure-sentinel/main/Logstash-Configuration/etc/logstash/conf.d/49-enhanced_private.conf -PO /etc/logstash/conf.d/
sudo wget https://raw.githubusercontent.com/noodlemctwoodle/pfsense-azure-sentinel/main/Logstash-Configuration/etc/logstash/conf.d/50-outputs.conf -PO /etc/logstash/conf.d/
```

Agora baixe os padrões gronk

```
sudo wget https://raw.githubusercontent.com/noodlemctwoodle/pfsense-azure-sentinel/main/Logstash-Configuration/etc/logstash/conf.d/patterns/pfelk.grok -P /etc/logstash/conf.d/patterns/
sudo wget https://raw.githubusercontent.com/noodlemctwoodle/pfsense-azure-sentinel/main/Logstash-Configuration/etc/logstash/conf.d/patterns/openvpn.grok -P /etc/logstash/conf.d/patterns/
```

E os seguintes arquivos de configuração.
```
sudo wget https://raw.githubusercontent.com/noodlemctwoodle/pfsense-azure-sentinel/main/Logstash-Configuration/etc/logstash/conf.d/36-ports-desc.conf -P /etc/logstash/conf.d/
```

E os bancos de dados..
```
sudo wget https://raw.githubusercontent.com/noodlemctwoodle/pfsense-azure-sentinel/main/Logstash-Configuration/etc/logstash/conf.d/databases/rule-names.csv -P /etc/logstash/conf.d/databases/
sudo wget https://raw.githubusercontent.com/noodlemctwoodle/pfsense-azure-sentinel/main/Logstash-Configuration/etc/logstash/conf.d/databases/service-names-port-numbers.csv -P /etc/logstash/conf.d/databases/
sudo wget https://raw.githubusercontent.com/noodlemctwoodle/pfsense-azure-sentinel/main/Logstash-Configuration/etc/logstash/conf.d/databases/private-hostnames.csv -P /etc/logstash/conf.d/databases/
```

NAgora que seus templates e banco de dados estão criados, vamos exportar suas regras do pfSense.

No pfSense, vá para Diagnósticos > Prompt de Comando e digite o seguinte comando para exportar as regras:
```
pfctl -vv -sr | grep label | sed -r 's/@([[:digit:]]+).*(label "|label "USER_RULE: )(.*)".*/"\1","\3"/g' | sort -V -u | awk 'NR==1{$0="\"Rule\",\"Label\""RS$0}7'
```

gora salve esses dados e crie o arquivo `/etc/logstash/conf.d/databases/rule-names.csv` com este conteúdo, isso lhe dará todas as regras e descrições do pfSense.

O último passo é editar /etc/logstash/conf.d/20-interfaces.conf e mudar o nome da interface para a interface correta. No meu caso, mudei de `igb2` para `eth0`:

```
    ### Change interface as desired ###
    if [interface][name] =~ /^eth0$/ {
      mutate {
        add_field => { "[interface][alias]" => "WAN" }
        add_field => { "[network][name]" => "FiOS" }
      }
    }
```

Você está quase lá! O último passo é reiniciar o logstash e iniciar um tcpdump para verificar as conexões na porta do logstash 5140
```
sudo service logstash stop
sudo service logstash start
sudo tcpdump -A -ni any port 5140 -v
```

Mantenha este terminal aberto para poder verificar se está recebendo pacotes assim que terminar a configuração do pfSense

### Criando uma regra de firewall para permitir o tráfego de entrada para o Logstash
Estamos quase lá. Agora você precisa permitir conexões de entrada na porta 5140, para que seu pfSense possa relatar logs para o Logstash.

No Portal Azure, vá para sua VM > Networking e adicione a seguinte Regra:

![Regra de Firewall de Rede do Syslog](/images/syslog-azure2.png)

Esteja ciente de que você também deve restringir o IP de origem para garantir que apenas seu pfSense possa enviar logs para o logstash. Estou omitindo o endereço IP aqui por razões óbvias.

## Configurando pfSense
O último e derradeiro passo desta configuração é permitir que o pfSense envie os logs para o seu servidor Syslog.

No pfSense, navegue até Status -> System Logs -> Settings.

General Logging Options.

Show log entries in reverse order. (newest entries on top)
General Logging Options > Log firewall default blocks. (optional)

1. Log packets matched from the default block rules in the ruleset.
2. Log packets matched from the default pass rules put in the ruleset.
3. Log packets blocked by 'Block Bogon Networks' rules.
4. Log packets blocked by 'Block Private Networks' rules.
5. Log errors from the web server process.

Remote Logging Options:

1. check "Send log messages to remote syslog server".
2. Select a specific interface to use for forwarding. (Optional)
3. Select IPv4 for IP Protocol.
4. Enter the Logstash server local IP into the field Remote log servers with port 5140. (e.g. 192.168.1.50:5140)
5. Under "Remote Syslog Contents" check "Everything".

## Configurando Log Analytics Workspace

Faça login no Portal Azure e vá para as configurações do `Log Analytics workspace`.
Selecione `Gerenciamento de Agentes` e anote o `ID do Workspace` e a `Chave Primária`. Se você ainda não tem um Log Analytics Workspace, agora é a hora de criar um!
Instale o plugin Logstash LogAnalytics da Microsoft usando o seguinte comando:
```
sudo /usr/share/logstash/bin/logstash-plugin install microsoft-logstash-output-azure-loganalytics
```

Edite a configuração do Logstash e adicione o id e a chave do seu workspace ao /etc/logstash/conf.d/50-outputs.conf.
```
output {
    microsoft-logstash-output-azure-loganalytics {
        workspace_id => "<WORKSPACE ID>" # <your workspace id>
        workspace_key => "<Primary Key>" # <your workspace key>
        custom_log_table_name => "<Name of Log>"
        }
    }
```

Reinicie o Logstash

```
sudo systemctl enable logstash
sudo systemctl start logstash
```

Dentro de alguns minutos, você deverá começar a ver eventos chegando ao Log Analytics

## Última Etapa: Sentinel!
Se você ainda não o fez, adicione o Microsoft Sentinel ao seu espaço de trabalho do Log Analytics. Depois de alguns minutos, você começará a ver os eventos chegando ao Sentinel

![Captura de Tela do Sentinel](/images/sentinel1.png)

Como você pode ver, tenho uma regra chamada "IoT Devices Trying to do something strange". Para ilustrar algumas capacidades básicas, mostrarei abaixo como criar uma consulta KQL simples para detectar se meus dispositivos IoT, que estão na minha VLAN IoT, tentam alcançar minha VLAN Confiável.


### Primeira Regra: Detectando movimentação lateral
No Sentinel, navegue até Logs e crie uma nova busca semelhante a esta:

```SQL
// Blocked IoT devices trying to communicate to Trusted VLAN
pfSenseLogstash_CL
| where TimeGenerated > ago(1m)
| where interface_alias_s == "mvneta0.50"
| extend event_created_t = TimeGenerated
| project TimeGenerated, source_ip_s, source_port_s, destination_ip_s, destination_port_s,
    network_direction_s, event_action_s, event_reason_s, rule_description_s, destination_service_s, network_transport_s
```

Esta consulta procura por alertas ocorrendo em mvneta0.50, que é minha VLAN IoT. Então, no final do dia, se meus dispositivos IOT estiverem fazendo algo que violaria uma de minhas regras de firewall, preciso verificar o que aconteceu.

![Sentinel Screenshot](/images/sentinel2.png)

Depois de testar esta consulta, você pode ir em `New Alert rule` > `Create Azure Sentinel Alert`

Preencher o nome, descrição e severidade da regra e seguir para `Set rule logic`

![Sentinel Screenshot](/images/sentinel_rule1.png)

Sentinel dá várias opções para enriquecer seus alertas, mapas e automatizar respostas. Para esse exemplo nós só vamos criar uma simples regra, então clique em `Next` ate chegar na tab `Review and create`.

Essa tab te dá a oportunidade de rever todas as configurações. Estando tudo certo clique em `Create`.

![Sentinel Screenshot](/images/sentinel_rule2.png)


Voila! Agora você tem sua primeira regra criada e o pfSense está reportando eventos para o Sentinel.

Na próxima vez eu vou mostrar como criar Dashboards e Alertas para começar a expandir seu deployment.

