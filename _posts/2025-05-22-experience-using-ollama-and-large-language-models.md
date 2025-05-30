---
layout: post
title: "Experience using ollama and large language models"
---

记录一下最近两个月，体验 ollama 本地部署/运行 LLM 的过程和思考；

# run deepseek/qwen3/llama3... locally (本地运行 deepseek,qwen3,llama3 等大语言模型)
The easiest way to run llm locally: [Ollama](https://ollama.com/). It is already too famous, here just copy its classic introduction:

[Ollama](https://ollama.com/) is a tool designed to simplify the process of running open-source large language models (LLMs) directly on your computer.
It acts as a local model manager and runtime, handling everything from downloading the model files to setting up a local environment
where you can interact with them.

# install [Ollama](https://ollama.com/) on Linux(Fedora-41)
see: https://ollama.com/download/linux
```
curl -fsSL https://ollama.com/install.sh | sh
```

# download/run models(deepseek,qwen3...)
from ollama site. see: https://ollama.com/search
```
ollama pull deepseek-r1:32b   #see: https://ollama.com/library/deepseek-r1, https://ollama.com/library/deepseek-r1:32b
ollama pull qwen3:32b   #see: https://ollama.com/library/qwen3, https://ollama.com/library/qwen3:32b
```

from 3rd-party site. e.g: https://huggingface.co/  
*Note: Now ollama only support models with GGUF format, you should find the GGUF version to pull/run*
```
ollama pull huggingface.co/Qwen/Qwen3-32B-GGUF
ollama pull huggingface.co/QuantFactory/Qwen2-Boundless-GGUF
```

if the model is not GGUF, ollama will give you error msg like:
```
$ ollama pull huggingface.co/ystemsrx/Qwen2-Boundless
pulling manifest 
Error: pull model manifest: 400: {"error":"Repository is not GGUF or is not compatible with llama.cpp"}
```

# run models
```
ollama ls
ollama run $the-model-name-get-from-ollama-ls   #if the model has not been local, ollama will pull it and run
```

# AI effects
[zh] 对于推理性问题，大模型的输出/反馈确实令人惊艳；包括一些确定性比较强的代码生成，效果也不错。

但是对一时效性比较强的新闻信息无能为力，这方面还是需要搜索引擎来完成，或者借助 MCP server 因该可以比较好的完成这类任务；

还有提问一些影视文学作品的原文片段、桥段、细节，大模型也表现很差，一方面是版权的限制，一方面样可能是文学性的内容关联性，推理性差
而 AI 模型的原理是找文字之间的相关性、概率；所以这类问题 AI 给出的结果/答案看起来大多是 一本正经的胡说八道。它好像不会说我不知道
而是找一个它能找到的相关性最大的结果 给提问者。 

也许经过不断的与用户对话学习、微调 能达到很好的效果，，但是本地训练对硬件要求比较高，我目前还没有尝试对模型进行微调，
等实践过后，再补充相关内容。

[en] For reasoning problems, the output/feedback of the large model is indeed amazing; including some code generation with strong certainty, the effect is also good.

However, it is powerless for news information with strong timeliness. This still requires search engines to complete, or with the help of MCP server,
this kind of task should be completed relatively well;

There are also questions about original text clips, plots, and details of film and television literary works. The big model also performs poorly.
On the one hand, it is due to copyright restrictions, and on the other hand, it may be the relevance of literary content and poor reasoning.
The principle of the LLM is to find the correlation and probability between words; so the results/answers given by AI for such questions mostly look like serious nonsense.
It doesn't seem to say I don't know, But to find the most relevant result it can find and give it to the questioner.

Perhaps after continuous dialogue with users, learning, and fine-tuning, it can achieve good results, but local training has high hardware requirements.
I have not tried to fine-tune the model yet. After practice, I will add relevant content


# MCP
[zh] MCP 协议刚刚开源不久，五月初发布的 qwen3 模型声明已经支持 MCP，但是目前好像还没有成熟、易于跟 ollama 一起工作的 MCP 管理工具。
MCP server 方面，也没有看到效果很好的易于本地部署的 cli or search-engine MCP server，再等等吧，应该很快就会出现。
很多应用提供商应该会基于 MCP 发布更好的 AI 智能产品；

[en] The MCP protocol has just been open sourced. The qwen3 model released in early May claims to support MCP,
but there seems to be no mature MCP management tool that is easy to work with ollama.

As for the MCP server, there is no cli or search-engine MCP server that is easy to deploy locally. Wait a little longer, it should appear soon.
Many application providers should release better AI/smart products based on MCP.

# Will AI replace human? (AI 会代替人工吗？)
[zh] 我认为不会，可能训练、微调工具足够完善易用后，人类可以利用 AI 根据个人习惯，生成很好的AI个人助理，方便更好的工作生活。  

顺便说一句: 工业革命产生的便利、导致很多人身体机能缺少锻炼；我有点担心 AI 会不会也导致人类的大脑的机能也产生某种退化

还有一点很重要：人类的记忆是立体多维度的，包括五官（视觉、听觉、嗅觉、味觉、触觉）还有情绪记忆，而语言模型LLM 只有语言/文字，
只是计算这几单词后面出现的单词的概率

[en] I don’t think so. Maybe after training and fine-tuning the tools to be perfect and easy to use,
humans can use AI to generate good AI personal assistants based on personal habits, which will facilitate better work and life.

By the way: The convenience brought by the Industrial Revolution has led to a lack of physical exercise for many people;
I am a little worried that AI will also cause some kind of degeneration of the human brain function.

Another important thing is that human memory is three-dimensional/multi-dimensional, including the five senses (vision, hearing, human perception, taste, touch) and emotional memory,
while the language model LLM only has language/text. Just calculate the probability of the words that appear after the previous few(or lots-of) words
