
## 内容概要

本文主要分析 WeiboMesh 在改造过程中对历史积累的一些考量以及适配，还有我个人对面向未来架构的思考。

从 SideCar 模式初具雏形的 2016 年末开始，WeiboMesh 就已经在生产环境逐步被摸索和打磨，并被大范围验证，这也是为什么 WeiboMesh 是目前最接地气的一个 Service Mesh 实现的原因所在。

它本身源自于微博内部对整体服务化的迫切需求，而面对微博巨大的流量和各业务线千差万别的异构服务现状，WeiboMesh 走出了一条适合自己也适合像微博这种在服务化进程中有着沉重历史包袱的团队。

## WeiboMesh 的独到之处

WeiboMesh 实现了当前 Service Mesh 的事实规范，在架构中抽象的一层实现了 Service Mesh 的数据面板和控制面板，二者很好地解决了请求的可靠传输和统一服务治理问题。

那 WeiboMesh 同其他 Service Mesh 实现相比又有哪些独到之处呢？我们下面从面临的问题和解决的思路两方面来简要分析。

伴随着微服务化和云化技术的普及，服务间的通信方式被重新定义，数据的可靠传输以及统一规范化的服务治理是大家必须面对的问题。如何应对这些问题？

微博多年来积累了一套基于 Motan RPC 和混合云的完备服务化体系，基于我们面临的问题和当时的现状考虑，解决思路很简单，通过 RPC 跨语言来完成数据的可靠传输及服务间的依赖调用，同时复用我们强大的 Client Side 服务治理体系来实现服务的统一规范化治理，以保障服务的高可用。

但是在演进的过程中发现跨语言 RPC 的支持其实没有那么大的困难，反倒是在各种语言的 Client 端都要实现 Client Side 的服务治理体系，难度超乎想象。于是我们削薄了集成了复杂治理功能的 RPC Client，将这些通用的服务治理功能抽象到了 WeiboMesh 中统一实现。

<img src="https://static001.geekbang.org/resource/image/ce/0a/ceaab06e118f9ac24abfde9b4711160a.png" alt="" />

这样一来，我们兼顾了微博平台原有的 RPC 服务体系，服务治理功能得到很好的复用，Client 对依赖的服务通过 Mesh 层本地调用实现，我们在 Mesh 层支持了常见的协议比如 HTTP、gRPC 等，让不同的语言选择自己喜欢的方式来调用。

而且这样有个好处就是我们去除了中间的层层转发，因为业务对自己所依赖的服务是明确的，你只需要在 Mesh 的配置文件里添加相关的依赖，服务发现、调用、治理等事情都交给 Mesh 层统一完成就好。

而 Mesh（Client Agent） 层又与所依赖的 Service 对端的 Mesh （Server Agent）通过 Motan RPC 完成请求调用，像我们以往擅长的那样。这种巧妙的 Motan-Client 调用实现就是我认为 WeiboMesh 独到的地方。

## Motan-Client

那这个削薄了 Motan-Client 是一定必要的吗？答案是否定的，我们奉行“适合自己的才是最好的”这样的原则。如果要让 WeiboMesh 支持其他 Mesh 实现的调用方式也很简单，只需要在框架层面对 Motan-Agent 进行简单封装即可，所以不一定需要这个 Client，但我们的场景下，这种方式是最适合的。

一方面主要是出于成本的考量，因为基于微博现状，我们平台内部更多的服务都是通过 RPC 服务来提供的，而对外提供的服务是 RESTful 接口，这样就会存在同样一个服务需要同时维护两个表现层，维护成本极高。

而 RESTful 接口的请求链路长，尤其是在我们升级为混合云架构以后，经常有请求在公私有云间穿梭的情况出现，取一条微博列表动辄上百 K 的数据，带着繁重的 HTTP 头，经过层层转发，性能损耗严重，带宽资源浪费，而且问题排查也很困难。

我们希望改变这种现状，统一只提供一种服务，面对 RPC 和 HTTP，我们选择了前者。

精简的自定义协议，高度可编程和可定制化，另外 RPC 实现点对点的通信，去除了之前的层层转发，性能、服务可用性保障都有了质的飞跃，问题排查也更明确、简单。

为了更低的成本把原来的 RESTful 接口改造为 RPC 服务，我们需要一个更精简的 Motan-Client 来替换之前的各种语言所使用的 HTTP-Client。

另一方面，基于架构的考虑，虽然我们也可以做到不需要这个 Motan-Client，因为这个精简的 Client 目前最大的作用就是用来发 Motan 协议的请求，我们完全可以在 Mesh 层来支持 HTTP 协议，这样之前使用 RESTful 接口的业务方就可以不改一行代码，而全部交由 Mesh 层处理。

但是我们只解决了眼下跨语言服务对业务服务依赖的问题，那资源层面呢？大家可以想象一下以往你的代码里面有多少各种各样的 Client ？

Memecache Client、Redis Client 各种语言都有多种实现，选择使用哪个 Client 是个问题，使用过程中的一些问题排查、检测等也都是问题。

如果都统一服务化呢？关注微博技术的同学可能知道，我们内部有 CacheService，我们不去纠结你要用哪种，你申请完直接用就是了。所以我们需要一个通用的 Client 来统一业务对各种服务的依赖调用，这就引出下一个话题，面向未来的架构。

## 面向未来 / 服务的架构

服务化和微服务之所以深得人心，是因为它给我们解决了很多之前头疼的问题，它完美适应了云计算时代的开发方式，改善了工程师的开发体验。

我们提倡面向服务的架构，把一切依赖的数据接口、资源都服务化后，未来在维护、运营方面就能极大的统一化、规范化，从而降低成本、提高效率，这也就是面向未来的架构。

我们希望能用一个 Client 来完成所有的服务、资源的依赖调用，而在 Mesh 层对这些资源、服务统一进行治理、认证、监控、遥测。把以往散落在各种业务代码里面那些与业务无关的通用逻辑都提到 Mesh 层来统一处理。这是我们面向未来的泛服务化构想，我们也在往这个方向上积极探索。

Motan-OpenResty 就是我们对泛服务化构想的一个探索之一，对于业务依赖接口的服务化，我们可以通过对 Mesh 进行注册和发现来解决，请求是直接的、点对点的，但是对于资源，比如对 Memcache 的一致性 Hash，Memcache 不会把自己注册到注册中心，Mesh 作为业务无关的中间件不应该去关注你是一致性 Hash 还是用的 Crc32 取模 Hash，这就需要有资源抽象的中间件，作为资源的 edge 代理。

OpenResty 作为 Nginx 最强大的衍生版本，提供了对四、七层友好的扩展能力，Motan-OpenResty 基于 OpenResty Stream-Lua 模块开发，实现了对 Motan 协议的高效封装与代理。我们只需要基于此实现相关的资源逻辑以及资源协议转换就能轻松完成对资源的服务化。这就是泛服务化的妙趣所在。

## 结束语

到此，我们本系列的几个议题就讨论完了，由于个人能力有限，如果有不妥当的地方，还望指正.我们从微博单体服务到 WeiboMesh 的演进，从整体感观上给大家描绘了一个 WeiboMesh 的路线图，不管你是处于单体服务、服务化、微服务、容器化等任何阶段，希望这些探讨能与你产生共鸣，最好是能对解决你的问题有所帮助。

我们同样对 WeiboMesh 几个核心的点分别进行了探讨，比如跨语言服务化的关键技术、WeiboMesh 对 Service Mesh 规范的理解、双向（Server Agent、Client Agent）代理的角色和功能、Motan-Client 在 WeiboMesh 中的关键作用和泛服务化的理念等。

我们也放眼业界，通过实例分析对比了 WeiboMesh 和 Istio 在处理请求过程中的一些区别，希望大家能通过这些议题的探讨对 Service Mesh 能有更深入的认识和了解，最好能直接上手体验下，也期待大家更多的关注 WeiboMesh ，让我们能得到更多社区的反馈，帮助我们做得更好。

希望大家一起努力,共同进步，感谢各位!

阅读过程中你有哪些问题和感悟？欢迎留言与大家交流，我们也会请作者解答你的疑问。
