---
title: "all_about_transformers"
date: 2025-01-06T00:09:23+08:00
draft: true
series:
    - llm2ai
---

ChatGPT 带火了 OpenAI，LLM 也开启了 AI 新时代。

LLM 的核心技术是 Transformer 架构，本文就来聊聊作为基石的 Transformer 架构, 洞悉神奇 AI 的玄机。


## 1. 横空出世

在 Transformer 架构问世前，NLP 领域的主流模型是 RNN（循环神经网络） 和 CNN（卷积神经网络）。

这些模型各有所长，但还是有些问题。

RNN 就像一个记忆力很强的读者，它会从第一个词开始逐个阅读，每读一个词都会更新自己的“记忆”，然后用这些记忆来理解当前读到的词。

但是它的“记忆”会随着时间推移而衰减，意味着它在处理长文本时记忆会丢失。此外，RNN 无法并行处理序列数据，因此顺序计算导致效率低。

而 CNN 则更擅长处理具有局部模式的数据，比如图像或短文本，所以它也不适合处理长文本。

2017 年基于 **多头注意力机制** 的 Transformer 架构横空出世，此后经过不断演变完善，不仅可以 **并行处理** 序列数据，而且可以 **捕捉长距离依赖关系**，终于长文本不再是难题。

各模型对比如下:

| 模型 | 优点 | 缺点 |
| --- | --- | --- |
| RNN (LSTM, GRU) | 处理序列数据，捕捉时间依赖关系 | 处理速度慢，难以并行处理，长距离依赖效果差，容易梯度消失或爆炸 |
| CNN (用于NLP) | 并行处理能力强，局部特征提取能力强 | 难以捕捉长距离依赖关系，全局上下文理解能力弱 |
| Transformer | 并行处理能力强，长距离依赖效果好，可解释性强，适合大规模训练 | 计算量较大，对于短序列的处理可能不如RNN |

Transformer 架构原理图如下：

![transformer](https://raw.githubusercontent.com/YabZhang/hugo-blog-source/main/assets/images/transformer.png)


## 2. 多头注意力机制

Transformer 架构的核心是多头注意力机制，它可以让模型在处理序列数据时，关注到序列中的重要部分。它的核心思想是计算词与词之间的相关性，并根据相关性调整词的权重。

这类似于我们去看一幅画，可以迅速地捕捉到图像中的形象，而不是逐个像素去分析；当我们快速浏览文章时，会把注意力放在关键词上，而不会平等地去关注每一个词。

注意力机制原理：

TODO


处理过程可视化：

TODO


## 3. 推理与训练

## 4. 前沿发展


Materials:

1. 一文读懂Transformer模型 https://mp.weixin.qq.com/s/NhFnfphMKvYN6Z1ai2ZvcA
2. Transformers for Machine Learning A Deep Dive
3. 
