---
title: "Show HN: SmallDocs – Markdown without the frustrationsShow HN: SmallDocs – 没有烦恼的 Markdown"
source: "https://news.ycombinator.com/item?id=47777633"
author:
  - "[[FailMore]]"
published: 2026-04-15
created: 2026-04-19
description: "Show HN: SmallDocs – Markdown without the frustrationsShow HN: SmallDocs – 没有烦恼的 Markdown - by FailMore on Hacker News"
tags:
  - "clippings"
---
[item?id=47777633](https://news.ycombinator.com/item?id=47777633)

Hi HN, I’d like to introduce you to SmallDocs ([https://sdocs.dev](https://sdocs.dev/)). SDocs is a CLI + webapp to instantly and 100% privately elegantly preview and share markdown files. (Code: [https://github.com/espressoplease/SDocs](https://github.com/espressoplease/SDocs))  
Hi HN，我想向大家介绍 SmallDocs（https://sdocs.dev）。SDocs 是一个 CLI+Web 应用，可以即时、100%私密且优雅地预览和分享 Markdown 文件。（代码：https://github.com/espressoplease/SDocs）

The more we work with command line based agents the more \`.md\` files are part of our daily lives. Their output is great for agents to produce, but a little bit frustrating for humans: Markdown files are slightly annoying to read/preview and fiddly to share/receive. SDocs was built to resolve these pain points.  
随着我们越来越多地使用基于命令行的代理，\`.md\` 文件在我们的日常生活中扮演着越来越重要的角色。它们的输出非常适合代理生成，但对人类来说有点令人沮丧：Markdown 文件在阅读/预览时有点烦人，在分享/接收时也有些繁琐。SDocs 旨在解决这些问题。

If you \`sdoc path/to/file.md\` (after \`npm i -g sdocs-dev\`) it instantly opens in the browser for you to preview (with our hopefully-nice-to-look-at default styling) and you can immediately share the url.  
如果你在安装了 \`npm i -g sdocs-dev\` 后输入 \`sdoc path/to/file.md\`，它将立即在浏览器中打开供你预览（带有我们希望看起来不错的默认样式），你可以立即分享该 URL。

The \`.md\` files our agents produce contain some of the most sensitive information we have (about codebases, unresolved bugs, production logs, etc.). For this reason 100% privacy is an essential component of SDocs.  
我们的代理生成的 \`.md\` 文件包含我们最敏感的一些信息（关于代码库、未解决的错误、生产日志等）。因此，100% 的隐私是 SDocs 的一个重要组成部分。

To achieve this SDoc urls contain your markdown document's content in compressed base64 in the url fragment (the bit after the \`#\`):  
要实现这一点，SDoc 网址的 URL 片段（\`#\`后面的部分）包含您 Markdown 文档内容的压缩 base64 编码：

[https://sdocs.dev/#md=GzcFAMT...(this](https://sdocs.dev/#md=GzcFAMT...\(this) is the contents of your document)...  
https://sdocs.dev/#md=GzcFAMT...(这是您文档的内容)...

The cool thing about the url fragment is that it is never sent to the server (see [https://developer.mozilla.org/en-US/docs/Web/URI/Reference/F...](https://developer.mozilla.org/en-US/docs/Web/URI/Reference/Fragment): "The fragment is not sent to the server when the URI is requested; it is processed by the client").  
URL 片段的有趣之处在于它永远不会发送到服务器（参考 https://developer.mozilla.org/en-US/docs/Web/URI/Reference/F...： "当请求 URI 时，片段不会被发送到服务器；它由客户端处理")。

The sdocs.dev webapp is purely a client side decoding and rendering engine for the content stored in the url fragment. This means the contents of your document stays with you and those you choose to share it with, the SDocs server doesn't access it. (Feel free to inspect/get your agent to inspect our code to confirm this!)  
sdocs.dev 网页应用纯粹是一个客户端解码和渲染引擎，用于处理 URL 片段中存储的内容。这意味着您的文档内容始终保留在您和您选择与之分享的人那里，SDoc 服务器无法访问它。（您可以自由检查/让您的代理检查我们的代码来确认这一点！）

Because \`.md\` files might play a big role in the future of work, SDocs wants to push the boundaries of styling and rendering interesting content in markdown files. There is much more to do, but to start with you can add complex styling and render charts visually. The SDocs root (which renders \`sdoc.md\` with our default styles) has pictures and links to some adventurous examples. \`sdoc schema\` and \`sdoc charts\` provides detailed information for you or your agent about how how make the most of SDocs formatting.  
因为\`.md\`文件在未来工作中可能扮演重要角色，SDocs 希望突破 Markdown 文件中样式和渲染有趣内容的界限。还有很多工作要做，但首先你可以添加复杂的样式并可视化地渲染图表。SDocs 根目录（使用默认样式渲染\`sdoc.md\`）包含一些冒险性示例的图片和链接。\`sdoc schema\`和\`sdoc charts\`为你或你的代理提供了关于如何充分利用 SDocs 格式的详细信息。

If you share a SDocs URL, your styles travel with it because they are added as YAML Front Matter - [https://jekyllrb.com/docs/front-matter/](https://jekyllrb.com/docs/front-matter/) - to the markdown file. E.g.:  
如果你分享一个 SDocs URL，你的样式会随之一起传播，因为它们被作为 YAML Front Matter（https://jekyllrb.com/docs/front-matter/）添加到 Markdown 文件中。例如：

```
---

   styles:

     fontFamily: Lora

     baseFontSize: 17

     ...

   ---
```
At work, we've been putting this project to the test. My team and I have found SDocs to be particularly useful for sharing agent debugging reports and getting easily copyable content out of Claude (e.g. a series of bash commands that need to be ran).  
在工作中，我们一直在测试这个项目。我和我的团队发现 SDocs 特别适用于分享代理调试报告，并能轻松地从 Claude 中获取可复制的内容（例如需要运行的 bash 命令序列）。

To encourage our agents to use SDocs we add a few lines about them in our root "agent files" (e.g. ~/.claude/CLAUDE.md or ~/.codex/AGENTS.md). When you use the cli for the first time there is an optional setup phase to do this for you.  
为了鼓励我们的代理使用 SDocs，我们在根目录的"代理文件"中添加了一些关于它们的说明（例如：~/.claude/CLAUDE.md 或~/.codex/AGENTS.md）。当你第一次使用 CLI 时，有一个可选的设置阶段可以为你完成这些操作。

I'm of course very interested in feedback and open to pull requests if you want to add features to SDocs.  
我对反馈非常感兴趣，如果你想要为 SDocs 添加功能，我欢迎你提交 pull 请求。

Thank you for taking a look!  
感谢你查看！

---

## Comments

> **franga2000** · [2026-04-19](https://news.ycombinator.com/item?id=47822640)
> 
> I can't believe I'm saying this, but this should be an Electron app. Or Tauri or whatever.  
> 我简直不敢相信自己在说这个，但这应该是一个 Electron 应用。或者 Tauri，或者任何其他类似的东西。
> 
> Seriously, it's a really nice Markdown app, but the "launch a CLI to urlencode your file" flow is such a messy way of doing this. Just open the file like any other app. Sure, the web version is convenient for demonstrating to people or one-off use, but it's no way to work day to day.  
> 说真的，它是一个相当不错的 Markdown 应用，但"启动 CLI 来对文件进行 urlencode"的流程真是做这件事的一种非常混乱的方式。就像其他任何应用一样打开文件。当然，网页版对于向他人展示或一次性使用很方便，但根本不适合日常使用。
> 
> As for the motivation... "Fiddly to send/receive"?? Just send the file like you would any other. Don't you have to send *other files*? So you already have a way of doing this. Just do that. Bonus points for being able to easily receive an edited file, make diffs, etc., as well as the person on the other side being able to use whatever viewer/editor they prefer, not the one you pushed onto them.  
> 至于动机... "发送/接收很麻烦"？？就像发送其他文件一样发送文件。难道你不用发送其他文件吗？所以你已经有了一种做这件事的方式。就那样做。如果能够轻松地接收编辑过的文件、生成差异等，并且对方可以使用他们喜欢的任何查看器/编辑器，而不是你强加给他们的，那就更棒了。
> 
> How is sending a GIGANTIC link any better? If your file is nearly as long as the shit my LLMs write, you'll reach the chat character limit on most platforms, even though the file itself is well within file upload limits. And now there's a link shortener to solve this problem, which just defeats the purpose of it being offline and independent of a cloud service.  
> 发送一个巨大的链接有什么更好的地方？如果你的文件几乎和我的 LLMs 写的东西一样长，你会在大多数平台上达到聊天字符限制，尽管文件本身完全在文件上传限制之内。现在有了链接缩短工具来解决这个问题，这反而违背了它离线且独立于云服务的目的。

> **petersumskas** · [2026-04-19](https://news.ycombinator.com/item?id=47821948)
> 
> \> Markdown files are slightly annoying to read/preview  
> \> Markdown 文件在阅读/预览时有点烦人
> 
> Maybe I’ve missed the intentions of markdown, but the ability to easily read the plain text version has always been the killer feature.  
> 或许我错过了 markdown 的初衷，但能够轻松阅读纯文本版本始终是它的杀手级功能。
> 
> Rendering as html is a nice bonus.  
> 渲染为 HTML 是一种不错的附加功能。
> 
> I understand there are plenty of useful things to say “but what about…” to, like inline images, and I use them. But they still detract from what differentiated markdown in the first place.  
> 我明白有很多有用的东西可以说“但……怎么办”，比如内联图片，我确实在使用它们。但它们仍然削弱了 markdown 最初与众不同之处。
> 
> The more of that you add, the more it could have been any document format.  
> 你添加得越多，它就越可能变成任何一种文档格式。
> 
> > **FailMore** · [2026-04-19](https://news.ycombinator.com/item?id=47822293)
> > 
> > I feel like things have changed as the main interface for code has (for some) become an agent running in the cli. I feel like we (certainly I) check my code editor way less frequently than before. Because of that (for me) easily reading/rendering Markdown files has become more of a pain than it used to be.  
> > 我觉得情况有所改变，因为对于一些人来说，代码的主界面变成了在命令行界面中运行的代理。我觉得我们（至少是我）检查代码编辑器的频率远不如以前。正因为如此（对我来说），轻松阅读/渲染 Markdown 文件比以前更麻烦了。
> > 
> > > **franga2000** · [2026-04-19](https://news.ycombinator.com/item?id=47822607)
> > > 
> > > If you problem is that you don't have a text editor open...just open it to read the file? When I click on a Markdown file, it opens in my code editor with the preview pane already open. Then I close it when I'm done. How is that different from any other file on a computer?  
> > > 如果你的问题是没打开文本编辑器……那就打开它来阅读文件？当我点击一个 Markdown 文件时，它会在我的代码编辑器中打开，预览窗格已经打开。然后在我完成后关闭它。这与电脑上的任何其他文件有什么不同？
> > > 
> > > If your problem is that you don't want to use a text editor, there are many markdown viewers out there, both dedicated (MarkLite), as part of a larger tool (Obsidian) or even in an office suite (LibreOffice Writer).  
> > > 如果你的问题是不想使用文本编辑器，有很多 Markdown 查看器，既有专门的（MarkLite），作为更大工具的一部分（Obsidian），甚至办公套件中也有（LibreOffice Writer）。
> > > 
> > > If your problem is that you don't want fo leave the terminal, there are many command line markdown "renderers", at least as far as that is even technically possible (glow is markdown-specific, bat is more of a general fancy text file viewer).  
> > > 如果你的问题是我不想离开终端，有很多命令行 Markdown "渲染器"，至少在技术上是可能的（glow 是专门针对 Markdown 的，bat 更像是一个通用的花哨文本文件查看器）。
> > > 
> > > I fail to see how any of these problems are even partially solved by a web app and a CLI tool that launches it, let alone any better than the existing solutions.  
> > > 我无法理解这些问题是如何通过一个网页应用及其启动工具得到部分解决的，更不用说比现有解决方案更好了。

> **FailMore** · [2026-04-17](https://news.ycombinator.com/item?id=47807630)
> 
> A little update: I added privacy-focused optional shorter URLs to SDocs.  
> 一个小更新：我为 SDocs 添加了注重隐私的可选短链接。
> 
> You can read more about the implementation here: [https://sdocs.dev/#sec=short-links](https://sdocs.dev/#sec=short-links)  
> 更多关于实现的信息请在这里阅读：https://sdocs.dev/#sec=short-links
> 
> Briefly:  简而言之：
> 
> ```
> https://sdocs.dev/s/{short id}#k={encryption key}
>                       └────┬───┘   └───────┬──────┘
>                            │                │
>                       sent to           never leaves
>                        server           your browser
> ```
> We encrypt your document client side. The encrypted document is sent to the server with an id to save it against. The encryption key stays client side in the URL fragment. (And - probably very obviously - the encryption key is required to make the sever stored text readable again).  
> 我们在客户端加密您的文档。加密后的文档会连同 ID 一起发送到服务器进行存储。加密密钥保留在 URL 的片段中。（而且——可能非常明显地——需要加密密钥才能将服务器存储的文本再次读取出来）。
> 
> You can test this by opening your browser's developer tools, switch to the Network tab, click Generate next to the "Short URL" heading, and inspecting the request body. You will see a base64-encoded blob of random bytes, not your document.  
> 您可以通过打开浏览器的开发者工具，切换到网络标签页，点击"短链接"标题旁边的"生成"按钮，然后检查请求体来测试这一点。您会看到一串随机字节的 base64 编码数据，而不是您的文档。

> **big\_toast** · [2026-04-16](https://news.ycombinator.com/item?id=47795933)
> 
> URL data sites are always very cool to me. The offline service worker part is great.  
> URL 数据网站对我来说总是非常酷。离线服务工作者部分很棒。
> 
> The analytics\[1\] is incredible. Thank you for sharing (and explaining)! I love this implementation.  
> 分析\[1\]太棒了。感谢分享（和解释）！我喜欢这个实现。
> 
> I'm a little confused about the privacy mention. Maybe the fragment data isn't passed but that's not a particularly strong guarantee. The javascript still has access so privacy is just a promise as far as I can tell.  
> 我对隐私方面的说明有点困惑。也许片段数据没有被传递，但这并不是一个特别强的保证。JavaScript 仍然可以访问，所以在我看来，隐私只是一个承诺。
> 
> Am I misunderstanding something and is there a stronger mechanism in browsers preserving the fragment data's isolation? Or is there some way to prove a url is running a github repo without modification?  
> 我是不是有什么误解，浏览器中是否有更强的机制来保护片段数据的隔离？或者有没有什么方法可以证明一个 URL 正在运行一个未经修改的 GitHub 仓库？
> 
> \[1\]:[https://sdocs.dev/analytics](https://sdocs.dev/analytics)
> 
> > **FailMore** · [2026-04-16](https://news.ycombinator.com/item?id=47798707)
> > 
> > Thanks for the kind words re the analytics!  
> > 感谢您对分析功能的美好评价！
> > 
> > You are right re privacy. It is possible to go from url hash -> parse -> server (that’s not what SDocs does to be clear).  
> > 你说得对关于隐私问题。从 URL 哈希到解析再到服务器是可能的（当然，SDocs 不会这样做，这一点要明确）。
> > 
> > I’ve been thinking about how to prove our privacy mechanism. The idea I have in my head at the moment is to have 2+ established coding agents review the code after every merge to the codebase and to provide a signal (maybe visible in the footer) that, according to them it is secure and the check was made after the latest merge. Maybe overkill?! Or maybe a new way to “prove” things?? If you have other ideas please let me know.  
> > 我一直在思考如何证明我们的隐私机制。目前我想到的方法是，在每次代码合并后，让两个或更多的认证编码代理审查代码，并提供一个信号（可能显示在页脚），表明他们认为这是安全的，并且检查是在最新的合并后进行的。这可能有点过度？或者也许是一种新的“证明”事物的方式？如果你有其他想法，请告诉我。
> > 
> > > **adelks** · [2026-04-19](https://news.ycombinator.com/item?id=47821521)
> > > 
> > > How about simply making the website an app and have it load your makedown file with a button and file browser. Just like e.g. [https://app.diagrams.net/](https://app.diagrams.net/)  
> > > 或者，我们只是把网站做成一个应用程序，通过按钮和文件浏览器加载你的 Markdown 文件。就像例如 https://app.diagrams.net/ 一样。
> > > 
> > > And I believe you can then tell the browser that you need no network communication at that point. And a user can double check that.  
> > > 而且我相信你可以在那个时刻告诉浏览器不需要网络通信。用户也可以双重检查这一点。
> > > 
> > > **big\_toast** · [2026-04-16](https://news.ycombinator.com/item?id=47799097)
> > > 
> > > No, I don't have any good ideas. Just hoping someone else does, or that I'm missing something.  
> > > 不，我没有什么好点子。只是希望其他人有，或者我错过了什么。
> > > 
> > > I think it's in the hands of browser vendors.  
> > > 我认为这取决于浏览器供应商。
> > > 
> > > The agent review a la socket.dev probably doesn't address all the gaps. I think you're already doing about as much as you reasonably can.  
> > > socket.dev 那样的代理审查可能无法解决所有问题。我认为你已经尽力了。
> > > 
> > > > **FailMore** · [2026-04-16](https://news.ycombinator.com/item?id=47799641)
> > > > 
> > > > Thanks. The question has made me wonder about the value of some sort of real time verification service.  
> > > > 谢谢。这个问题让我开始思考实时验证服务的价值。
> > > > 
> > > > > **Nevermark** · [2026-04-19](https://news.ycombinator.com/item?id=47821120)
> > > > > 
> > > > > If it's possible to isolate that part of the code, and essentially freeze it for long periods. At least people would know it wasn't being tweaked under them all the time.  
> > > > > 如果能够将那部分代码隔离出来，并长时间冻结它。至少人们会知道它没有被一直暗中修改。
> > > > > 
> > > > > That is my half of a bad idea.  
> > > > > 这就是我那糟糕想法的一半。
> > > > > 
> > > > > > **FailMore** · [2026-04-19](https://news.ycombinator.com/item?id=47822303)
> > > > > > 
> > > > > > I have something coming out soon (just working on it). Your client (browser) has hashing algos built into it. So the browser can run a hash of all the front end assets it serves. Every commit merged into main will cause a hash of all the public files to be generated. We will allow you to compare the hashes of the front end files in your browser with the hashes from the public GH project. Interested to know what you think...  
> > > > > > 我很快就要发布一个东西（正在开发中）。你的客户端（浏览器）内置了哈希算法。所以浏览器可以运行它所服务的所有前端资产的哈希值。每次合并到主分支的提交都会生成所有公共文件的哈希值。我们将允许你比较浏览器中的前端文件哈希值与公共 GH 项目的哈希值。想了解你对这个的看法...
> 
> > **edgardurand** · [2026-04-19](https://news.ycombinator.com/item?id=47821285)
> > 
> > For the "prove the server doesn't touch the data" problem — the realistic path today is probably reproducible builds + published bundle hashes.  
> > 对于"证明服务器未触碰数据"的问题——如今现实的路径可能是可重复构建+已发布的捆绑哈希。
> > 
> > ```
> > Concretely: the sdocs.dev JS bundle should be byte-for-byte reproducible                                                                                                         
> >   from a clean checkout at a given commit. You publish { gitSha, bundleSha256 }
> >   on the landing. Users (or agents) can compute the hash of what their browser                                                                                                     
> >   actually loaded (DevTools → Sources → Save As → sha256) and compare.                                                                                                             
> >    
> >   That closes the "we swapped the JS after deploy" gap. It doesn't close                                                                                                           
> >   "we swapped it between the verification moment and now" — SRI for SPA
> >   entrypoints is still not really a thing. That layer is on browser vendors.                                                                                                       
> >                                                                                                                                                                                    
> >   The "two agents review every merge" idea upthread is creative, but I worry                                                                                                       
> >   that once the check is automated people stop reading what's actually                                                                                                             
> >   verified. A dumb published hash is harder to fake without getting caught.                                                                                                        
> >                                                                                                                                                                                    
> >   (FWIW, working on a similar trust problem from the other end — a CLI + phone                                                                                                     
> >   app that relays AI agent I/O between a dev's machine and their phone                                                                                                             
> >   [codeagent-mobile.com]. "Your code never leaves your machine" is easy to                                                                                                         
> >   say, genuinely hard to prove.)
> > ```
> > 
> > > **FailMore** · [2026-04-19](https://news.ycombinator.com/item?id=47822308)
> > > 
> > > That's basically exactly what I'm working on now actually. We will let you compare all the publicly served files with their hashes on github  
> > > 基本上就是我目前正在着手的工作。我们将让你们在 github 上比较所有公开提供的文件及其哈希值。
> > > 
> > > **big\_toast** · [2026-04-19](https://news.ycombinator.com/item?id=47821393)
> > > 
> > > Ya. I could imagine a browser extension performing some form of verification loop for simpler webpages. Maybe too niche.  
> > > 是的。我可以想象一个浏览器扩展为简单网页执行某种形式的验证循环。可能过于小众了。

> **fredericgalline** · [2026-04-18](https://news.ycombinator.com/item?id=47814454)
> 
> Nice implementation — the URL fragment trick for privacy is clever.  
> 实现很棒——用于隐私的 URL 片段技巧很巧妙。
> 
> Related pattern I've leaned into heavily: treating .md files as structured state the agent reads back, not just output. YAML frontmatter parsed as fields (status, dependencies, ids), prose only in the body. Turns them from "throwaway outputs" into state the filesystem enforces across sessions — a new session can't silently drift what was decided in the previous one.  
> 我深入探索的一种相关模式：将.md 文件视为代理读取的结构化状态，而不仅仅是输出。YAML 前体解析为字段（状态、依赖项、ID），正文仅包含散文。将它们从"一次性输出"转变为文件系统跨会话强制的状态——新会话不能无声地偏离前一个会话的决定。
> 
> Your styling-via-frontmatter is the same mechanism applied to presentation. Have you thought about a read mode that exposes the frontmatter as structured data, for agents that consume sdoc URLs downstream?  
> 你的通过前体进行样式设置与呈现的机制是相同的。你是否考虑过一种读取模式，可以以前体为结构化数据的形式展示，供下游消费 sdoc URL 的代理使用？
> 
> > **FailMore** · [2026-04-19](https://news.ycombinator.com/item?id=47822317)
> > 
> > I think the next thing I want to do (but not sure how to implement yet) is to make it easy for your agent to go from SDocs url to content. I don't know if that's via curl or a \`sdoc\` command, or some other way... That could include the styling Front Matter / the agent could specify it.  
> > 我认为我想做的下一件事（但还不确定如何实现）是让你的代理能够轻松地从 SDoc URL 跳转到内容。我不知道这是通过 curl 还是\`sdoc\`命令，或以其他方式...这可能包括样式前体/代理可以指定它。
> > 
> > At the moment the most efficient way to get sdocs content into an agent is to copy the actual content. But I think that's not too beautiful.  
> > 目前将 sdoc 内容导入代理的最有效方式是复制实际内容。但我认为这不太优雅。

> **throwaway81523** · [2026-04-19](https://news.ycombinator.com/item?id=47821858)
> 
> Soon... there are 15 competing standards.  
> 很快...就会有 15 种竞争标准。

> **pdyc** · [2026-04-16](https://news.ycombinator.com/item?id=47791589)
> 
> i also used fragment technique for sharing html snippets but url's became very long, i had to implement optional url shortener after users complained. Unfortunately that meant server interaction.  
> 我也曾使用片段技术来分享 HTML 代码片段，但 URL 变得非常长，我不得不在用户投诉后实现可选的 URL 缩短器。不幸的是，这意味着需要服务器交互。
> 
> [https://easyanalytica.com/tools/html-playground/](https://easyanalytica.com/tools/html-playground/)
> 
> > **FailMore** · [2026-04-17](https://news.ycombinator.com/item?id=47807637)
> > 
> > (I left a stand alone comment, but:) A little update: I added privacy-focused optional shorter URLs to SDocs.  
> > （我留下了一个独立的评论，但：）一个小更新：我为 SDocs 添加了注重隐私的可选缩短 URL。
> > 
> > You can read more about the implementation here: [https://sdocs.dev/#sec=short-links](https://sdocs.dev/#sec=short-links)  
> > 更多关于实现的信息请在这里阅读：https://sdocs.dev/#sec=short-links
> > 
> > Briefly:  简而言之：
> > 
> > ```
> > https://sdocs.dev/s/{short id}#k={encryption key}
> >                       └────┬───┘   └───────┬──────┘
> >                            │                │
> >                       sent to           never leaves
> >                        server           your browser
> > ```
> > We encrypt your document client side. The encrypted document is sent to the server with an id to save it against. The encryption key stays client side in the URL fragment. (And - probably very obviously - the encryption key is required to make the sever stored text readable again).  
> > 我们在客户端加密您的文档。加密后的文档会连同 ID 一起发送到服务器进行存储。加密密钥保留在 URL 的片段中。（而且——可能非常明显地——需要加密密钥才能将服务器存储的文本再次读取出来）。
> > 
> > You can test this by opening your browser's developer tools, switch to the Network tab, click Generate next to the "Short URL" heading, and inspecting the request body. You will see a base64-encoded blob of random bytes, not your document.  
> > 您可以通过打开浏览器的开发者工具，切换到网络标签页，点击"短链接"标题旁边的"生成"按钮，然后检查请求体来测试这一点。您会看到一串随机字节的 base64 编码数据，而不是您的文档。
> > 
> > **FailMore** · [2026-04-16](https://news.ycombinator.com/item?id=47791690)
> > 
> > Really nice implementation by the way.  
> > 顺便说一句，实现得真的很棒。
> > 
> > Re URL length: Yes... I have a feeling it could become an issue. I was wondering if a browser extension might give users the ability to have shorter urls without losing privacy... but haven't looked into it deeply/don't know if it would be possible (browser extensions are decent bridges between the local machine and the browser, so maybe some sort of decryption key could be used to allow for more compressed urls...)  
> > 关于 URL 长度：是的……我感觉这可能成为一个问题。我在想是否可以通过浏览器扩展让用户在不牺牲隐私的情况下拥有更短的 URL……但还没深入调查，也不知道是否可行（浏览器扩展是本地计算机和浏览器之间不错的桥梁，所以也许可以使用某种解密密钥来允许更紧凑的 URL……）
> > 
> > > **pdyc** · [2026-04-16](https://news.ycombinator.com/item?id=47792162)
> > > 
> > > i doubt it would be possible, it boils down to compression problem compressing x amount of content to y bits, since content is unpredictable it cannot be done without having intermediary to store it.  
> > > 我怀疑这不太可能实现，关键在于压缩问题，将 x amount 的内容压缩成 y bits，由于内容是不可预测的，如果没有中间存储介质就无法完成。
> 
> > **mystickphoenix** · [2026-04-16](https://news.ycombinator.com/item?id=47793009)
> > 
> > For this use-case, maybe compression and then encoding would get more data into the URL before you hit a limit (or before users complain)?  
> > 对于这个用例，也许先压缩再编码，可以在达到限制（或用户开始抱怨之前）将更多数据放入 URL？
> > 
> > I.e. .md -> gzip -> base64  
> > 例如：.md -> gzip -> base64

> **beckford** · [2026-04-19](https://news.ycombinator.com/item?id=47822140)
> 
> Using fragments for secure data has been discussed before on hn: [https://news.ycombinator.com/item?id=23036515](https://news.ycombinator.com/item?id=23036515). Tldr: it may not go directly to the server (unless you are using a buggy browser or web client) but the fragment is captured in several places.  
> 使用片段进行安全数据传输之前在 hn 上讨论过：https://news.ycombinator.com/item?id=23036515。简而言之：它可能不会直接发送到服务器（除非你使用的是有问题的浏览器或网络客户端），但片段会在多个地方被捕获。

> **stealthy\_** · [2026-04-16](https://news.ycombinator.com/item?id=47790288)
> 
> Nice, I've also built something like this we use internally. Will it reduce token consumption as well?  
> 挺好的，我也内部构建了类似的东西。这能减少 token 消耗吗？
> 
> > **FailMore** · [2026-04-16](https://news.ycombinator.com/item?id=47790326)
> > 
> > Thanks. Re tokens reduction: not that I’m aware of. Would you mind explaining how it might? That could be a cool feature to add  
> > 谢谢。关于 token 减少：据我所知并没有。你介意解释一下它可能的工作原理吗？这可能会是一个很酷的功能

> **Arij\_Aziz** · [2026-04-18](https://news.ycombinator.com/item?id=47814913)
> 
> This is a neat tool. I always had to manually copypaste longs texts into notepad and convert it into md format. Obvisouly i couldn't parse complex sites with lots of images or those that had weird editing. this will be useful  
> 这是一个很棒的工具。我以前总是需要手动将长文本复制粘贴到记事本中并将其转换为 md 格式。显然我无法解析包含大量图片或编辑奇怪的复杂网站。这将很有用
> 
> > **FailMore** · [2026-04-18](https://news.ycombinator.com/item?id=47815194)
> > 
> > Thank you. If you use an AI agent you might be able to tell it to curl the target website, extract the content into a markdown file and then sdoc it. It might have some interesting ideas with images (using the hosted URLs or hosting them yourself somehow)  
> > 谢谢。如果你使用 AI 代理，你可能会告诉它 curl 目标网站，将内容提取到 markdown 文件中然后 sdoc。它在处理图片方面可能会有一些有趣的想法（使用托管 URL 或以某种方式自己托管它们）

> **moaning** · [2026-04-16](https://news.ycombinator.com/item?id=47790018)
> 
> Markdown style editing looks very easy and convenient  
> Markdown 风格的编辑看起来非常简单方便
> 
> > **FailMore** · [2026-04-16](https://news.ycombinator.com/item?id=47790227)
> > 
> > Thanks! One potential use case I have for it is being able to make "branded" markdown if you need to share something with a client/public facing.  
> > 谢谢！我有一个潜在的用途是，如果你需要与客户或公开分享内容，可以制作“品牌化”的 Markdown。

> **moeadham** · [2026-04-15](https://news.ycombinator.com/item?id=47778009)
> 
> I had not heard of url fragments before. Is there a size cap?  
> 我之前没听说过 URL 片段。有没有大小限制？
> 
> > **FailMore** · [2026-04-15](https://news.ycombinator.com/item?id=47780616)
> > 
> > Ish, but the cap is the length of url that the browser can handle. For desktop chrome it's 2MB, but for mobile Safari its 80KB.  
> > 是的，但限制是浏览器能处理的 URL 长度。对于桌面版 Chrome 是 2MB，但对于移动版 Safari 是 80KB。
> > 
> > The compression algo SDocs uses reduces the size of your markdown file by ~10x, so 80KB is still ~800KB of markdown, so fairly beefy.  
> > SDocs 使用的压缩算法将您的 markdown 文件大小压缩至原来的约十分之一，因此 80KB 仍然相当于约 800KB 的 markdown，相当可观。
> > 
> > > **tcfhgj** · [2026-04-19](https://news.ycombinator.com/item?id=47820914)
> > > 
> > > It's 2^16=65,536 bytes for Firefox  
> > > 对于 Firefox 来说，它是 2^16=65,536 字节
> 
> > **vivid242** · [2026-04-16](https://news.ycombinator.com/item?id=47791141)
> > 
> > Hadn’t heard of it either - very smart, could open lots of other privacy-friendliness-improved „client-based web“ apps  
> > 我也没听说过这个 - 非常聪明，可以打开很多其他隐私性改进的“基于客户端的网页”应用
> > 
> > > **FailMore** · [2026-04-16](https://news.ycombinator.com/item?id=47791166)
> > > 
> > > TYVM. Yeah, I am curious to explore moving into other file formats like CSVs.  
> > > 谢谢。是的，我对探索转换到其他文件格式（如 CSV）很感兴趣。

> **pbronez** · [2026-04-17](https://news.ycombinator.com/item?id=47805038)
> 
> Cool project. Heads up - there’s a commercial company with a very similar name that might decide to hassle you about it:  
> 很棒的项目。注意一下 - 有一个商业公司名称非常相似，可能会因此找你的麻烦：
> 
> [https://www.sdocs.com/](https://www.sdocs.com/)
> 
> > **FailMore** · [2026-04-17](https://news.ycombinator.com/item?id=47805361)
> > 
> > Thanks + thanks for the heads up. I will see what happens. It's a domain-name war out there!  
> > 感谢提醒。我会看看会发生什么。现在到处都是域名之争！
> > 
> > > **deepfriedbits** · [2026-04-19](https://news.ycombinator.com/item?id=47821961)
> > > 
> > > In the spirit of r/IllegallySmolCats, perhaps SmolDocs is a possible option.  
> > > 以 r/IllegallySmolCats 的精神为例，或许 SmolDocs 是一个可行的选择。