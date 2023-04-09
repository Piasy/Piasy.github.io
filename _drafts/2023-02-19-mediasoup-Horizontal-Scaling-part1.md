---
layout: post
title: mediasoup 水平扩缩容（一）
tags:
    - 实时多媒体
    - WebRTC
    - mediasoup
---

咱们[书接上回](/2023/02/18/mediasoup-Quick-Start/index.html)，聊聊 mediasoup 的水平扩缩容。

## 基本思路

在和朋友请教 mediasoup 扩缩容的时候，首先被提到的就是 PipeTransport，确实网上搜出来的很多资料也提到了 PipeTransport，但其实如果仔细阅读 [mediasoup 官方的 Scalability 页面](https://mediasoup.org/documentation/v3/scalability/)：

> Multiple and Separate mediasoup Routers
>
> ...
>
> If higher capability is required, the application backend should run mediasoup in multiple hosts and distribute “rooms” across them.

我们发现关键点其实是把房间分布到不同的机器上去。

当然也会有 PipeTransport 的应用场景，那就是超大房间（以及官方页面里提到的 One-To-Many Broadcasting，但这其实也就是超大房间的一种形式）：只有在一个房间的流才需要互通（或者更复杂的多房间互通的业务逻辑），一台机器扛不住一个房间的负载时才需要用 PipeTransport 连通多台机器。

那其实可以通过动态域名的方式实现扩缩容，用不着 PipeTransport，除非是搞多机房就近接入，能提高客户端的接入质量，否则如果 PipeTransport 连通的多台机器就在同一个机房，那就确实没必要了，反而是让运行时的情况变得更复杂。

具体的，这个调度器可以监控现有机器的负载情况，根据配置的水位，决定是否需要启动或者释放机器。然后业务 API server 向这个调度器获取 mediasoup 的信令 url，调度器再根据负载情况，给每个房间分配一个信令 url（其实也就是 mediasoup worker）。

释放机器还需要有个排空的过程，就是决定要释放之后，不能直接释放，先进行一个标记，不再分配新的房间过来，等现有房间全部销毁之后，再释放机器。需要增加机器时，优先从处于排空状态的机器中挑一台，取消标记，继续分配新的房间，没有排空状态的机器时，才启动新的机器。

机器的启动/释放、域名的添加/移除、DNS 的配置，都可以调用云商的 API 来实现。而且域名可以把时间戳放进去，这样都不用担心 DNS 缓存的问题。

机器列表、负载情况，都可以存在数据库中（比如 redis），那这个调度器本身是无状态的，都可以用类似于 AWS 的 lambda 功能实现，redis 和 lambda 的扩缩容，云商都已经提供了，所以整个系统就都是可水平伸缩的了。

超大房间场景，还是需要用到 PipeTransport，但就不是房间粒度的调度，而是参与者粒度的调度。

其实思路也差不多，就是业务 API server 在每个用户入会/下会时，去向调度器获取 mediasoup 的信令 url，调度器根据负载情况，决定是复用现有机器，还是创建新的机器，只不过新启动的机器，还需要和现有的机器建立 PipeTransport 连接，具体连接哪些机器，则根据房间里的各个流都在哪些机器上来确定。而且调度策略应该优先于尽量把用户都集中在少量的机器上，这样 PipeTransport 就不用太多。

总结下来，我们需要做这几件事：

+ mediasoup 服务容器化；
+ worker 负载情况上报，可以请求基于 lambda 的 API 服务；
+ 调度器逻辑实现，是否支持超大房间倒也不需要做成配置，逻辑中根据房间人数来决定是否使用 PipeTransport 即可；

## 容器化

...

限于篇幅，第一部分我们就到这里，对 mediasoup 的改造、调度器的实现以及压测的内容，大家敬请期待 :)
