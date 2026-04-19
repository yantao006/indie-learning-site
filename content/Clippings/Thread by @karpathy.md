---
title: "Thread by @karpathy"
source: "https://x.com/karpathy/status/2039805659525644595"
author:
  - "[[@karpathy]]"
published: 2026-04-03
created: 2026-04-19
description: "LLM Knowledge BasesLLM 知识库Something I'm finding very useful recently: using LLMs to build personal knowledge bases for various topics of res"
tags:
  - "clippings"
---
LLM Knowledge Bases  
LLM 知识库  
  
Something I'm finding very useful recently: using LLMs to build personal knowledge bases for various topics of research interest. In this way, a large fraction of my recent token throughput is going less into manipulating code, and more into manipulating knowledge (stored as markdown and images). The latest LLMs are quite good at it. So:  
最近我发现一个非常有用的方法：使用 LLMs 来构建关于各种研究兴趣主题的个人知识库。通过这种方式，我最近大部分的 token 吞吐量不再用于操作代码，而是用于操作知识（以 markdown 和图像形式存储）。最新的 LLMs 在这方面表现得相当出色。所以：  
  
Data ingest: I index source documents (articles, papers, repos, datasets, images, etc.) into a raw/ directory, then I use an LLM to incrementally "compile" a wiki, which is just a collection of .md files in a directory structure. The wiki includes summaries of all the data in raw/, backlinks, and then it categorizes data into concepts, writes articles for them, and links them all. To convert web articles into .md files I like to use the Obsidian Web Clipper extension, and then I also use a hotkey to download all the related images to local so that my LLM can easily reference them.  
数据摄取： 我将源文档（文章、论文、代码库、数据集、图像等）索引到 raw/目录中，然后使用 LLM 逐步“编译”一个维基，它只是一个目录结构中的.md 文件集合。这个维基包含 raw/中所有数据的摘要、反向链接，然后它将数据分类到概念中，为它们撰写文章，并将它们全部链接起来。为了将网络文章转换为.md 文件，我喜欢使用 Obsidian Web Clipper 插件，然后我还使用快捷键将所有相关图像下载到本地，这样我的 LLM 就可以轻松地引用它们。  
  
IDE: I use Obsidian as the IDE "frontend" where I can view the raw data, the the compiled wiki, and the derived visualizations. Important to note that the LLM writes and maintains all of the data of the wiki, I rarely touch it directly. I've played with a few Obsidian plugins to render and view data in other ways (e.g. Marp for slides).  
IDE: 我使用 Obsidian 作为 IDE 的"前端"，在那里我可以查看原始数据、编译后的维基以及派生的可视化。需要注意的是，LLM 负责编写和维护维基的所有数据，我很少直接操作它。我尝试过几个 Obsidian 插件，以其他方式渲染和查看数据（例如，使用 Marp 制作幻灯片）。  
  
Q&A: Where things get interesting is that once your wiki is big enough (e.g. mine on some recent research is ~100 articles and ~400K words), you can ask your LLM agent all kinds of complex questions against the wiki, and it will go off, research the answers, etc. I thought I had to reach for fancy RAG, but the LLM has been pretty good about auto-maintaining index files and brief summaries of all the documents and it reads all the important related data fairly easily at this ~small scale.  
问答： 有趣之处在于，一旦你的维基足够大（例如我最近关于某项研究的维基有~100 篇文章和~40 万字），你就可以向你的 LLM 代理提出各种复杂的问题，它会去研究答案等等。我曾以为需要使用复杂的 RAG，但 LLM 在自动维护索引文件和所有文档的简要摘要方面表现得相当好，并且在这个~小规模上它能相当容易地读取所有重要相关数据。  
  
Output: Instead of getting answers in text/terminal, I like to have it render markdown files for me, or slide shows (Marp format), or matplotlib images, all of which I then view again in Obsidian. You can imagine many other visual output formats depending on the query. Often, I end up "filing" the outputs back into the wiki to enhance it for further queries. So my own explorations and queries always "add up" in the knowledge base.  
输出： 与其在文本/终端中获取答案，我更喜欢让它渲染 Markdown 文件给我，或者幻灯片（Marp 格式），或者 matplotlib 图像，我之后都会在 Obsidian 中再次查看这些内容。你可以想象根据查询会有许多其他的视觉输出格式。通常，我会把输出结果“归档”回维基中，以增强它，供后续查询使用。所以我的探索和查询总是“累积”在知识库中。  
  
Linting: I've run some LLM "health checks" over the wiki to e.g. find inconsistent data, impute missing data (with web searchers), find interesting connections for new article candidates, etc., to incrementally clean up the wiki and enhance its overall data integrity. The LLMs are quite good at suggesting further questions to ask and look into.  
代码检查： 我运行了一些 LLM “健康检查”来检查维基，例如查找不一致的数据、用网络搜索器填充缺失数据、为新文章候选者寻找有趣的关联等，以逐步清理维基并增强其整体数据完整性。LLM 在建议进一步的问题和深入探究方面表现得相当出色。  
  
Extra tools: I find myself developing additional tools to process the data, e.g. I vibe coded a small and naive search engine over the wiki, which I both use directly (in a web ui), but more often I want to hand it off to an LLM via CLI as a tool for larger queries.  
额外工具： 我发现自己在开发额外的工具来处理数据，例如我 vibe 编码了一个小型且简单的维基百科搜索引擎，我既直接使用它（在网页界面中），但更经常我想通过命令行界面将其交给 LLM 作为更大查询的工具。  
  
Further explorations: As the repo grows, the natural desire is to also think about synthetic data generation + finetuning to have your LLM "know" the data in its weights instead of just context windows.  
进一步探索： 随着仓库的增长，自然地想要考虑合成数据生成+微调，以便让 LLM "在权重中知道"数据而不是仅仅在上下文窗口中。  
  
TLDR: raw data from a given number of sources is collected, then compiled by an LLM into a .md wiki, then operated on by various CLIs by the LLM to do Q&A and to incrementally enhance the wiki, and all of it viewable in Obsidian. You rarely ever write or edit the wiki manually, it's the domain of the LLM. I think there is room here for an incredible new product instead of a hacky collection of scripts.  
TLDR：从给定数量的来源收集原始数据，然后由 LLM 编译成一个 .md 维基百科，然后由 LLM 通过各种命令行界面进行操作以进行问答和逐步增强维基百科，所有这些都可以在 Obsidian 中查看。你很少手动编写或编辑维基百科，这是 LLM 的领域。我认为这里有机会开发一款令人难以置信的新产品，而不是一堆杂乱的脚本。