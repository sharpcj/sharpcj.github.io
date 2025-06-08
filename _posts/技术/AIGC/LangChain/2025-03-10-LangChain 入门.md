---
layout: post
title: "LangChain入门学习"
date:  2025-03-10 22:58:12 +0800
categories: ["技术", "AIGC", "LangChain"]
tag: ["AIGC", "LangChain"]
---

## 前言
未来已来。2022年11月30日，OpenAI 发布了 ChatGPT,这是 AI 时代重要的一次划时代的产品，到如今，AI 已经开始渗透到普通人生活中的点点滴滴，各种 AI 应用产品层出不穷，其中以 LLM 为基础的应用产品尤为广泛。为了能够快速定制开发 LLM 应用，诞生了很多开发框架，其中以 LangChain 最为有名。作为互联网从业人员，我觉得学习和使用 LangChain 框架很有必要，可以帮助自己理解 LLM 大模型，提升自己的工作效率。从本文开始，接下来系列文章将记录一些学习心得。

## 一、认识 LangChain
### 1.1 LangChain 的概念

认识 LangChain
1.1 LangChain 的概念
LangChain 是一个开源框架，旨在简化基于大型语言模型（LLM）的应用程序开发。其核心设计理念是通过模块化组件和链式任务管理，将 LLM 与外部工具、数据源和服务集成，构建复杂的 AI 应用（如智能问答系统、自动化工作流等）。它使得应用程序能够：
- 具有上下文感知能力：将语言模型连接到上下文来源（提示指令，少量的示例，需要回应的内容等）
- 具有推理能力：依赖语言模型进行推理（根据提供的上下文如何回答，采取什么行动等）

关键特性包括：

- ​模型接口统一化：支持多种 LLM（如 OpenAI、DeepSeek、Llama 等）和嵌入模型，开发者可灵活切换底层模型。
- ​链（Chains）​：通过组合多个组件（如模型调用、数据处理、工具调用）形成端到端任务流水线，例如 RAG（检索增强生成）流程。
- ​代理（Agents）​：赋予 LLM 动态决策能力，根据用户需求调用外部 API、数据库或工具，实现自动化操作。
- ​记忆（Memory）​：支持短期对话上下文和长期用户偏好存储，增强多轮交互的连贯性。

这个框架由几个部分组成：
- **LangChain 库**：Python 和 JavaScript 库。包含了各种组件的接口和集成，一个基本的运行时，用于将这些组件组合成链和代理，以及现成的链和代理的实现。
- **LangChain 模板**：一系列易于部署的参考架构，用于各种任务。
- **LangServe**：一个用于将 LangChain 链部署为 REST API 的库。
- **LangSmith**：一个开发者平台，让你可以调试、测试、评估和监控基于任何 LLM 框架构建的链，并且与 LangChain 无缝集成。

### 1.2 LangChain 的价值
LangChain 的核心价值在于降低 LLM 应用开发门槛并扩展模型能力边界：
- ​简化开发流程：通过预置组件（如提示模板、文档加载器）减少重复代码，例如快速搭建基于本地知识库的问答系统。
- ​灵活集成外部资源：支持与 70+ LLM 和 700+ 第三方工具（如向量数据库、API）交互，解决模型“数据孤岛”问题。
- ​支持复杂任务编排：通过链和代理实现多步骤推理（如“解析用户意图→检索数据→生成回答→记录日志”）。
- ​活跃生态与工具链：衍生工具 LangSmith（监控）、LangGraph（多代理协作）等形成完整 LLMOps 体系。

### 1.3 其它 LLM 应用开发框架
除了 LangChain 以外，还有很多其它 LLM 应用开发框架，这里以 LlamaIndex、 Flowise AI 作一个对比：
- LangChain 以灵活性和扩展性见长，适合复杂任务编排与多工具集成场景
- LlamaIndex 是一个专注于 RAG 应用程序构建的开源框架，在 RAG 应用开发领域更具有优势。
- Flowise AI 是一个低代码开发框架，适合不动编程的用户快速构建 LLM 应用。

## 二、安装 LangChain
安装方法有很多，这里以 pip 为例：

```
pip install langchain
```

这将安装LangChain的最基本要求。 LangChain的很多价值在于将其与各种模型提供商、数据存储等进行集成。 默认情况下，进行此类集成所需的依赖项并未安装。您需要单独安装特定集成的依赖项。

**LangChain Core**
langchain-core包含LangChain生态系统使用的基础抽象，以及LangChain表达式语言。它由langchain自动安装，但也可以单独使用。安装方法如下：

```
pip install langchain-core
```

**LangChain community**
langchain-community包含第三方集成。它由langchain自动安装，但也可以单独使用。安装方法如下：

```
pip install langchain-community
```

**LangChain experimental**
langchain-experimental包含实验性的LangChain代码，用于研究和实验用途。 安装方法如下：

```
pip install langchain-experimental
```

**LangServe**
LangServe帮助开发者将LangChain可运行文件和链作为REST API部署。 LangServe由LangChain CLI自动安装。 如果不使用LangChain CLI，安装方法如下：

```
pip install "langserve[all]"
```

用于客户端和服务器依赖项。或者 `pip install "langserve[client]"` 用于客户端代码，和 `pip install "langserve[server]"` 用于服务器代码。

**LangChain CLI**
LangChain CLI对于处理LangChain模板和其他LangServe项目非常有用。 安装方法如下：

```
pip install langchain-cli
```

**LangSmith SDK**
LangSmith SDK由LangChain自动安装。 如果不使用LangChain，安装方法如下：

```
pip install langsmith
```

## 三、第一个 LangChain 应用示例
单纯学习概念很是很枯燥无味的，先整个例子跑起来了，可能会更有趣味。

### 3.1 环境设置
LangChain 通过需要与一个或者多个模型提供者、数据存储、API 等进行集成，如果是访问在线的 LLM 服务，一般需要进行 API 秘钥配置，可以通过环境变量配置，也可以在初始化时通过参数设置。本人学习是在本地进行部署的大模型，无需配置 API 秘钥，具体用法下面会提到。

这里首先在本地部署大模型，前面有一篇文章讲过使用 LLM Studio 在本地部署大模型，作为开发，这里选择了使用 ollama 部署 DeepSeek 大模型。

### 3.2 第一个应用程序

直接上代码：

```
from langchain_openai import ChatOpenAI

def test_openai():
    # 指向 Ollama 本地 API 地址
    llm = ChatOpenAI(
        openai_api_base="http://localhost:11434/v1",  # Ollama 默认端口
        openai_api_key="NA",  # 本地无需密钥，但需占位符
        model_name="deepseek-r1:14b"  # 对应 Ollama 已下载的模型名称，如 qwen2.5:latest
    )

    # 调用模型
    result = llm.invoke("写一首关于春天的诗")
    print(result.content)
```

简单说明一下代码：
1. 首先通过 ChatOpenAI 构造出一个大模型实例对象，必须的参数有模型名称，访问地址，以及 api_key，相关的代码都有注释说明。
2. 调用大模型对象的 `invoke` 方法，参数传入提示词。
3. 打印出返回值的 `content`。

输出结果如下：

```
<think>
好的，用户让我写一首关于春天的诗。首先，我需要确定主题是春天，所以我要想到春天的特点，比如温暖、花开、绿草、鸟鸣等等。

接下来，考虑诗的形式。可能选择四句的绝句，结构简单，容易表达春天的美好。押韵也很重要，通常选平声韵，让读起来顺口。

然后，每个句子的内容安排。第一句可以描述天气的变化，比如“暖风轻”来表现春风拂面的感觉。第二句描绘大地苏醒，万物复苏，“大地醒”这样的词简洁明了。

第三句需要具体一些，比如花开的情景，用“桃红李白”来代表各种花卉盛开的颜色和种类。最后一句可以加入动物的元素，增强生机勃勃的画面感，比如“喜鹊唱春耕”，表现春天不仅是视觉上的美丽，更是农耕的开始。

赏析部分要解释诗中的意象和情感。第一句点出春风温暖，第二句展现大地生机，第三句用色彩丰富画面，第四句以声音收尾，整体表达对春天的喜爱和赞美。

最后，检查押韵是否合适，句子结构是否流畅。确保每句都紧扣主题，没有偏离春天的意境。这样整首诗就完成了。
</think>

《七绝·咏春》
暖风轻抚柳含烟，
大地苏醒展新颜。
桃红李白争相艳，
喜鹊枝头唱春天。

赏析：这首作品描绘了春天的美丽景象，通过“暖风轻”、“柳含烟”等意象生动展现了春风和煦、万物复苏的自然美景。诗中“桃红李白”一句更是以鲜明的色彩对比，形象地表现了花的繁盛与艳丽。末句以“喜鹊唱春天”作结，寓意着春天带给人们的喜悦与生机，在声色交融中表达了对春天的热爱之情。
```

由于使用的 DeepSeek 模型，所以连大模型的思考内容也一并输出出来了。

### 3.3 ChatOpenAI 和 ChatOllama
上面的例子中使用了 ChatOpenAI 创建大模型实例，来自于 `from langchain_openai import ChatOpenAI` 这个包。
另外还有使用 OpenAI 创建大模型实例的，输入是字符串，输出也是字符串。两者的区别，后面一片文章再说。

由于我是本地部署的大模型，在 `from langchain_ollama import ChatOllama` 这个包中还有一个 `ChatOllama` 可以用来创建大模型对象。它与 ChatOpenAI 的区别：
1. ChatOpenAI 需要有一个参数 `openai_api_key`, 本地模型可以随便传一个，但是必须传一个占位。ChatOllama 没有这个参数。
2. ChatOpenAI 严格遵循 OpenAI 的 `/v1/chat/completions` 接口标准,所以它的 `openai_api_base` 参数为 "http://localhost:11434/v1"。而使用 ChatOllama 时，它有一个参数 `base_url`, 如果传入和 ChatOpenAI 一样，则会报错 404， 因为 ollama 本地服务地址为 "http://localhost:11434", 无需 `/v1`, 它只能传入 "http://localhost:11434" 或者 `None`。 ChatOllama 也可以不传这个参数，默认值就是 `None`，因为它默认调用本地 11434 端口的 Ollama 服务。当然如果你是用的其它服务如 LM Studio，则需明确指定。

总结一下：
1. OpenAI 适用于生成连贯文本、代码补全等单轮任务, ChatOpenAI支持多轮对话上下文管理，适合聊天机器人、客服系统等场景。 OpenAI 能干的活，ChatOpenAI 都能干。所以绝大多数情况下，使用 ChatOpenAI 就够了。
2. **ChatOpenAI 适用于依赖云端服务、追求高性能的标准化任务（如客服机器人、代码生成）。而 ChatOllama 适用于对隐私敏感、需本地化运行模型的场景（如医疗数据处理、企业私有知识库）。**

## 后记
本文简单介绍了 LangChain 的概念，演示了使用 LangChain 和 LLM 的基本交互，后续文章会详细介绍更多知识点。
最后吟诗一首：

```
《诉衷情·银屏未暗》
银屏未暗乱码纷，数据锁重门。
夜阑键碎声切，寒透薄衫身。
云海卷，客星沦，竞浮沉。
倦指频移，敲碎冰心，倦眼空瞋。
——怎一个卷字堪题？
```