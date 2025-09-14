---
title: "Flashing Sonoff S31 com o Flipper Zero (Sem Adaptador USB Serial!)"
date: 2025-09-14T16:47:29-04:00
draft: false
toc: true
tags: ["home-assistant", "tasmota", "flipper-zero", "iot", "diy"]
---

Um guia passo a passo de como fazer **Flashing** no **Sonoff S31** com o firmware **Tasmota**, usando apenas o **Flipper Zero** (sem adaptador USB-Serial e sem necessidade de solda).

---

Eu estava tendo problemas com meus antigos dispositivos **Wemo**. Eles viviam caindo da rede e eu queria me livrar da dependência da nuvem. Depois de algumas pesquisas, encontrei o **Sonoff S31** — uma tomada inteligente disponível em duas versões: Zigbee e Wi-Fi.  

O que realmente me chamou a atenção foi a versão Wi-Fi: ela pode receber firmwares customizados como **Tasmota** ou **ESPHome**, tornando-se totalmente local e perfeita para minha integração com o **Home Assistant**. Vendido! 🚀  

Mas ainda restava uma grande questão:  
*Como fazer o Flash desse dispositivo sem um adaptador USB-Serial?*  

Olhando para a mesa, vi meu **Flipper Zero**. Será que eu poderia usar os pinos GPIO dele para fazer o Flash do Sonoff? Desafio aceito. 

---

## Abrindo o Sonoff S31

Abrir o dispositivo é bem simples. Use uma faca ou chave de fenda para remover a tampa cinza e depois retire três parafusos. Isso vai revelar a placa interna:

![Sonoff aberto](/images/sonoff_1.png)

Na placa, você verá os pinos usados para o Flash. Para o nosso caso, só precisamos de **VCC, RX, TX e GND**:

![Pinos Sonoff](/images/sonoff_2.png)

---

## Conectando o Flipper Zero

O Flipper Zero oferece tanto saída de **5V** quanto de **3,3V**.  
⚠️ **Atenção:** o Sonoff S31 trabalha com **3,3V**, então use o pino correto.

No Flipper Zero, conecte:

- **3V3 → VCC**  
- **TX → RX**  
- **RX → TX**  
- **GND → GND**

Repare na inversão de TX/RX: o TX do Flipper deve ir no RX do Sonoff e vice-versa.

Aqui está como ficou minha montagem:

![Fiação Flipper](/images/flipper_1.png)

Para evitar solda, usei **garras de teste (test hook clips)** ([link Amazon](https://www.amazon.com/dp/B08M5GNY47)) em vez de pinos fixos. Assim, não precisei modificar permanentemente a placa.

---

## Configurando o Flipper Zero como USB-UART Bridge

O próximo passo é expor o Flipper como adaptador USB-Serial:

1. No Flipper, vá em **GPIO → USB-UART Bridge**  
2. Isso cria uma porta COM no seu computador que será usada para fazer o Flash do firmware.

![USB UART](/images/flipper_2.png)

Com tudo conectado, segure o **botão de energia** do Sonoff enquanto liga o dispositivo — isso ativa o **modo de boot**.

![Boot Mode](/images/flipper_3.png)

---

## Fazendo Flash do Tasmota

Agora vem a parte divertida: fazer o Flash. Acesse o [instalador web do Tasmota](https://tasmota.github.io/install).

1. Clique em **Connect**  
2. Selecione o dispositivo USB do Flipper  
3. Escolha a versão **Tasmota (English)**  
4. Aguarde enquanto o dispositivo finaliza de escrever o novo Firmware

![Selecionar Tasmota](/images/tasmota_1.png)  
![Escolher Dispositivo](/images/tasmota_2.png)  
![Instalando](/images/tasmota_3.png)

---

## Conclusão

E pronto! O Sonoff S31 agora está rodando Tasmota. Você pode se conectar ao Wi-Fi criado por ele, configurar o dispositivo e integrá-lo diretamente ao **Home Assistant** — sem depender de serviços de nuvem.  

Sem dongle USB, sem solda, sem complicação. Apenas **Flipper Zero + Sonoff S31 + Tasmota = Tomada inteligente totalmente local**.  

Aproveite suas novas tomadas offline!