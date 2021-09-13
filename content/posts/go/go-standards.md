---
title: "上帝视角看 “Go 项目标准布局” 之争"
date: 2021-09-13T23:34:23+08:00
images:
tags: 
  - go
---

大家好，我是煎鱼。

前段时间 Go 语言社区有一件事情引爆了热议，那就是 `golang-standards/project-layout` 项目的 “Go 项目的标准布局” 之争。

没想到，五一假期，认真一看，这个 issues 已经提出将近一个月了，仍然在热议阶段，我想，咱们需要好好的聊聊这个话题。

煎鱼带你了解下的前因后果，再分享我的看法和业务真实情况。

背景
--

### 问题发生地

在 GitHub 上有一个项目 Spaghetti（github.com/adonovan/spaghetti），是 Go 软件包的一个依赖性分析工具。

该项目的目录结构如下：

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/11b7f74aa3944e509c89bcbfc51a2c99~tplv-k3u1fbpfcp-zoom-1.image)

看上去并不复杂，代码量不多，文件平铺也不超过一屏，就是一个布局比较简单的项目。

有一位老哥提出了一个 PR，明确的期望该项目按照 `golang-standards/project-layout` 项目给出的 “标准” 布局来调整。：

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e0da59130ce94ee190d51491396d6919~tplv-k3u1fbpfcp-zoom-1.image)

我猜测该项目可能是因为把 Go、HTML、JS、PNG 和 go.mod 文件等摆在了一起，引起了该同学的一丝丝纠结，觉得比较乱？

### “标准布局“ 长什么样子

在 `golang-standards/project-layout` 项目中，其自称：

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/55511ceb43cb48c59d0ce302783eec59~tplv-k3u1fbpfcp-zoom-1.image)

项目的组织名也是 "golang-standards"，其提供了一个基本的 Go 项目布局，精简展示如下：

```
project-layout
├── api
├── cmd
├── configs
├── docs
├── go.mod
├── init
├── internal
├── pkg
├── scripts
├── vendor
├── ...

```

*   /cmd：项目主要的应用程序。
    
*   /internal：私有的应用程序代码库，这些是不希望被其他人导入的代码。
    

*   应用程序实际的代码可以放在 /internal/app 目录（如：internal/app/myapp）。
    
*   应用程序的共享代码放在 /internal/pkg 目录（如：internal/pkg/myprivlib）中。
    

*   /pkg：外部应用程序可以使用的库代码（如：/pkg/mypubliclib）。其他项目将会导入这些库来保证项目可以正常运行。
    
*   /vendor：应用程序的依赖关系，可通过执行 `go mod vendor` 执行得到。
    
*   /configs：配置文件模板或默认配置。
    
*   /init：系统初始化（systemd、upstart、sysv）和进程管理（runit、supervisord）配置。
    
*   /scripts:：用于执行各种构建，安装，分析等操作的脚本。
    

更具体的布局介绍，大家可以参见 project-layout 项目的 README，其基本把方方面面的目录都考虑到了（人多力量大）。

由于内容过于长，因此就不一一展示了。

### Russ Cox 现身原因

不过很巧，该项目的作者是前 Google 员工，是 `gopl.io` 项目（5.1k stars）的作者。

在仅仅过去 23 分钟后，作为 GoTeam Leader 的 Russ Cox（@rsc）就现身，并提出新的 issue 表达出了反对意见：

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f72fcd999116477193c22430eb33c22f~tplv-k3u1fbpfcp-zoom-1.image)

在 `golang-standards/project-layout` 项目的 README 中有明确指出这不是官方的标准，有如下声称：

> > it is a set of common historical and emerging project layout patterns in the Go ecosystem.

Russ Cox 主要是对声称 "这是一套 Go 生态系统中常见的历史和新兴的项目布局模式" 这一说法表示了 “不准确” 的意见。

例如：Go 生态系统中的绝大多数包都不会将可导入的包放在 pkg 子目录中。更广泛地说，这里描述的只是非常复杂的工程项目，而 Go 的仓库往往要简单得多。

另外，不幸的是，这套项目布局在组织名字上被称作 "golang-standards"（Golang 标准） 提出来，实际上并非真的是官方标准，有误导的情况存在。

### Russ Cox 反对原因

在了解 project-layout 项目所提供的 “标准“ 项目布局和 Russ Cox 提出 issues 的背景后。

我们进一步了解 Russ Cox 认为**这不对**的根本考虑。project-layout 这个项目有两个问题：

*   它声称是 Go 标准（Go standards）的主办方，但实际上并非如此，因为这些标准绝非 Go 官方标准。
    
*   它提出的项目布局标准过于复杂，不是一个合理的标准。
    

Go 项目布局的标准是什么
-------------

提出这个 issues 后，出现了一大堆人追问 Russ Cox，到底何为 Go 项目的布局标准？

Russ Cox 给出了正式回应，一个可导入的 Go repo 的最小标准布局是：

*   在你的根目录下放一个 LICENSE 文件。
    
*   在你的根目录下放一个 go.mod 文件。
    
*   将 Go 代码放在 repo 中，放在根目录中，或者按照你认为合适的方式组织成一个目录树。
    

就这样了，这就是 "标准"，没有那么复杂。不需要像 project-layout 项目一样的布局。像是 Go 官方的 `golang.org/x` 仓库打破了 project-layout 所说的这些 "规则 "中的每一条。

Go 提案
-----

在经历了长时间的口水战后，已经有人在 Go 官方仓库提出希望释出相关的提案（proposal）：

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ada96a8647e04b38a340995ccf764156~tplv-k3u1fbpfcp-zoom-1.image)

猜测可能会有如下几种可能：

*   GitHub 项目 golang-standards/project-layout 愿意更名，不再自称 ”golang-standards“，不过可能性比较低，因为已经已多人提出，但作者没什么表示。
    
*   Go 官方正式提供 Go 标准项目布局的说明。
    
*   Go 官方不做约束，仅做表态，可能输出文章。
    

后续大家继续关注该提案，就可以知道发展了，传送门：issues #45861。

按照惯例，我猜测第三种可能性最大，因为很难有人可以提供所有开发者认可的标准，每个事业部、团队的喜好都可能有所不同。

总结
--

实际上，任何东西自称 “XX 标准”，在名气大后，都会带来一些问题。就像本文提到的 golang-standards/project-layout 项目一样。

换位思考一下，若你是某个项目的 Leader，某一天你的同事，被人拿着 “标准” 来建议修改时，说这是这个项目的 “标准”，会不会很奇妙？

无独有偶，我有一个朋友，他们公司早年只有一套 DDD 标准，本想统一。结果后面每一个介入 DDD 的业务同学，都认为前人不标准，每个人都自创了一套 DDD 标准。

总是会有小伙伴**想让定义绝对的 “标准”，又或是 “最佳实践”**。其实是难以定义的，最好的就是能够一个团队内形成基本共识，这里面牵扯到的不单单只有技术...

你对此有什么看法呢，**欢迎在评论区留言和大家一起交流**！