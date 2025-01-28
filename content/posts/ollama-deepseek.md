---
title: "OLlama Deepseek"
date: 2025-01-28T17:36:42-05:00
draft: false
toc: true
---

There's a lot of chatter about DeepSeek R1 and how it is disrupting the [tech markets this week](https://www.theverge.com/ai-artificial-intelligence/598846/deepseek-big-tech-ai-industry-nvidia-impac) (Jan 27, 2025), as well as concerns about how [DeepSeek collects your data](https://www.wired.com/story/deepseek-ai-china-privacy-data/). After all, their privacy policy explicitly states that it stores collected information on servers located in the People’s Republic of China. This includes user inputs such as chat messages, device information, and interaction data. Critics are concerned about the potential for content censorship and the misuse of collected data by the Chinese government. 

With all that in mind, given that DeepSeek is an open-source model, you can run it locally on your machine, test it out, and avoid some of the data privacy issues. Here's a quick guide on how I installed it on my M1 MacBook Air.

## Installing Ollama
[Ollama](https://ollama.com) is a platform that allows users to run and interact with AI models locally on their own devices, without requiring cloud access. It's free and runs under the MIT license.

You can download Ollama from its official website: [https://ollama.com/download](https://ollama.com/download)

Once downloaded, follow the installation instructions for your operating system.

## Downloading the DeepSeek R1 Model

Ollama offers an extensive repository of models you can download and run locally on your device. Check it out: [https://ollama.com/library](https://ollama.com/library)

Remember, larger models provide more sophisticated AI capabilities but require more computational power. Since I have limited hardware, I decided to go with the **8B version**. Just open your terminal and run:

```bash
ollama run deepseek-r1:8b
```

![ollama run deepseek-r1:8b](/images/ollama_run.png)

Once the download is complete, you're good to go. Ollama will provide a prompt where you can query the model and see its reasoning:

![ollama run deepseek-r1:8b](/images/ollama_query1.png)

That's all! I recommend starting with a smaller model to test whether you have enough compute power before moving to a bigger version.

## Using Chatbox for a Better Interface

While Ollama provides a basic terminal interface to query the model, there are other graphical interfaces that improve the user experience. One great alternative is **[Chatbox AI](https://chatboxai.app/en)**, which supports images, documents, and the creation of custom Copilots.

You can download Chatbox AI here: [https://chatboxai.app/en](https://chatboxai.app/en)

### Setting Up Chatbox AI with Ollama
1. Open **Chatbox AI**.
2. Navigate to **Settings**.
3. Under **Model Provider**, select **Ollama**.
4. Choose the **DeepSeek R1 model** you downloaded.
5. Adjust additional settings like **Temperature** (which controls response randomness) as needed.

![Chatbox AI settings](/images/chatbox_ai.png)

Now, you can play with it:

![Chatbox AI Chat](/images/chatbox_ai_2.png)


## Conclusion

Running DeepSeek R1 locally using Ollama is a great way to explore this AI model while maintaining data privacy. 

If you're new to AI models, I recommend starting with a smaller model to test your system's performance before scaling up. Whether you're using a model for research, development, or curiosity, running it locally ensures that you stay in control of your data. 

Ah! Take the opportunity to also test Microsoft’s [phi-4](https://ollama.com/library/phi4) this small model is powerful and delivers excellent performance.

Try it out !