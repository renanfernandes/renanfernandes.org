---
title: "Controle Total do DeepSeek R1: Passo a Passo para Instalação Local"
date: 2025-01-28T17:36:42-05:00
draft: false
toc: true
---

Essa semana uma startup Chinesa movimentou o mercado de tecnologia com o DeekSeek R1, um novo modelo que está revolucionando os [mercados essa semana](https://www.theverge.com/ai-artificial-intelligence/598846/deepseek-big-tech-ai-industry-nvidia-impac) (27 de janeiro de 2025), e causando preocupações sobre como o [DeepSeek coleta seus dados](https://www.wired.com/story/deepseek-ai-china-privacy-data/). Afinal, sua política de privacidade afirma explicitamente que as informações coletadas são armazenadas em servidores localizados na República Popular da China. Isso inclui entradas dos usuários, como mensagens de chat, informações do dispositivo e dados de interação. Críticos estão preocupados com o potencial de censura de conteúdo e o uso indevido de dados coletados pelo governo chinês.

Levando isso em conta, dado que o DeepSeek é um modelo de código aberto, você pode executá-lo localmente no seu computador, testá-lo e evitar algumas dessas questões de privacidade de dados. Aqui está um guia rápido sobre como instalei o modelo no meu MacBook Air M1.

## Instalando o Ollama
[Ollama](https://ollama.com) é uma plataforma que permite aos usuários executar e interagir com modelos de IA localmente em seus próprios dispositivos, sem a necessidade de acesso à nuvem. É gratuito e licenciado sob a licença MIT.

Você pode baixar o Ollama no site oficial: [https://ollama.com/download](https://ollama.com/download)

Depois de baixado, siga as instruções de instalação para o seu sistema operacional.

## Baixando o Modelo DeepSeek R1

O Ollama oferece um repositório extenso de modelos que você pode baixar e executar localmente no seu dispositivo. Confira: [https://ollama.com/library](https://ollama.com/library)

Lembre-se de que modelos maiores oferecem capacidades mais sofisticadas de IA, mas exigem maior poder computacional. Como tenho hardware limitado, optei pela versão **8B**. Basta abrir seu terminal e executar:

```bash
ollama run deepseek-r1:8b
```

![ollama run deepseek-r1:8b](/images/ollama_run.png)

Depois que o download for concluído, você estará pronto para usar. O Ollama fornecerá um prompt onde você pode consultar o modelo e ver seu raciocínio:

![ollama run deepseek-r1:8b](/images/ollama_query1.png)

É isso! Recomendo começar com um modelo menor para verificar se você tem poder de processamento suficiente antes de passar para uma versão maior.

## Usando o Chatbox para uma Interface Melhor

Embora o Ollama ofereça uma interface básica no terminal para consultar o modelo, existem outras interfaces gráficas que melhoram a experiência do usuário. Uma ótima alternativa é o **[Chatbox AI](https://chatboxai.app/en)**, que suporta imagens, documentos e a criação de copilotos personalizados.

Você pode baixar o Chatbox AI aqui: [https://chatboxai.app/en](https://chatboxai.app/en)

### Configurando o Chatbox AI com o Ollama
1. Abra o **Chatbox AI**.  
2. Navegue até **Configurações**.  
3. Em **Provedor de Modelo**, selecione **Ollama**.  
4. Escolha o modelo **DeepSeek R1** que você baixou.  
5. Ajuste configurações adicionais como **Temperatura** (que controla a aleatoriedade das respostas), se necessário.

![Configurações Chatbox AI](/images/chatbox_ai.png)

Agora você pode explorar e interagir com o modelo:

![Chatbox AI Chat](/images/chatbox_ai_2.png)

## Conclusão

Executar o DeepSeek R1 localmente usando o Ollama é uma ótima maneira de explorar este modelo de IA e manter a sua privacidade. 

Se você é novo em modelos de IA, recomendo começar com uma versão menor para testar o desempenho do seu sistema antes de escalar para uma versão maior. Seja para pesquisa, desenvolvimento ou curiosidade, rodar o modelo localmente garante que você tenha controle total sobre seus dados.  

Ah! Aproveite para testar também o [phi-4](https://ollama.com/library/phi4) da Microsoft, esse pequeno modelo é poderoso e com uma performance excelente

Experimente e me conte como foi!

