---
layout: post
title: 让我给 AI Coding 来个小考
tags:
  - AI
---

最近刚好有个比较复杂的问题，可以对比下几个 AI 的表现，今天就给大家分享一下。

去年手写了一个 Kotlin multiplatform 的 SocketIO 库：[kmp-socketio](https://github.com/HackWebRTC/kmp-socketio)，使用 Ktor 里的 WebSocket，实现了多平台统一的 SocketIO 实现。（_当时手写代码的过程还是非常快乐的_）

这个项目有充分的单元测试（很大一部分来自 Java 版项目的用例），之前发现有个用例偶现失败，我收集到了这个用例正常通过时的完整日志，以及失败时的完整日志。老实说代码写了一年了，我一时半会儿也看不出是什么问题，所以先尝试问了 Kimi Code，根据回答我判断大概是个多线程/事件竞争的问题，但它的回答我不是非常认可，有些地方说不通。于是我根据它的半吊子回答，自己再好好看了日志和代码，终于自己把问题给梳理清楚了。

既然这个问题比较复杂，Kimi Code 做得不好，我正好就试试看，其他 AI Coding 工具效果怎么样，我测试了这几个：
1. Kimi Code (K2.5)
2. OpenCode + K2.5
3. OpenClaw + K2.5
4. Trae + K2.5
5. Trae + GPT-5.2-Codex
6. Claude Code + Opus 4.6

因为我买了 Kimi Code 会员，所以就想着测测 K2.5 在不同工具下表现如何，Trae + GPT 则是看看换成更强的模型后 Trae 效果如何。最后，当世最强的 Claude Code + Opus 4.6 我今天也终于算是体验上了。

_因为问题涉及一些 socketio 的实现细节，我就把问题详细说明放到附录部分了，感兴趣的朋友可以[跳转查看](/2026/02/27/AI-Coding-Evaluation/index.html#%E9%99%84%E5%BD%95%E9%97%AE%E9%A2%98%E8%AF%A6%E7%BB%86%E8%AF%B4%E6%98%8E)_。

## 工具配置篇

先来简单分享下工具配置里我遇到的坑。

### Oh My OpenCode 配置使用 Kimi for coding K2.5

上一篇我分享了 OpenCode 相关的内容，但我发现配置 Oh My OpenCode 使用 Kimi for coding K2.5 还有点费劲，让它自己配置也折腾（怒骂）了一会儿才勉强搞成，所以这里简单说下。

首先，OpenCode 安装后确实直接就能用它的免费模型，也能在里面让它安装 Oh My OpenCode，但安装下来后 `~/.config/opencode/oh-my-opencode.json` 里配置各个 agent 使用的模型是 `opencode/glm-4.7-free`，但实际上这个模型压根不能用（不过 OpenCode 会回退到可用的 Big Pickle 模型，所以也还能继续用）。使用 Big Pickle 的正确配置值是 `opencode/big-pickle`。

然后要使用 Kimi for coding K2.5，按照官方文档使用 `/connect` 命令是不行的，需要修改 `~/.config/opencode/opencode.json`，增加这段：

```json
  "provider": {
    "kimi-for-coding": {
      "name": "Kimi For Coding",
      "npm": "@ai-sdk/anthropic",
      "options": {
        "baseURL": "https://api.kimi.com/coding/v1",
	      "apiKey": "sk-kimi-XXXXXX"
      },
      "models": {
        "k2p5": {
          "name": "Kimi K2.5",
          "reasoning": true,
          "attachment": false,
          "limit": {
            "context": 262144,
            "output": 32768
          },
          "modalities": {
            "input": ["text", "image", "video"],
            "output": ["text"]
          },
          "options": {
            "interleaved": {
              "field": "reasoning_content"
            }
          }
        }
      }
    }
  }
```

然后在 oh-my-opencode.json 把模型取值配置为 `kimi-for-coding/k2p5`。

### OpenClaw + K2.5 + Slack

我安装 OpenClaw 的步骤比较顺利，就是按照官方说明执行 `curl -fsSL https://openclaw.ai/install.sh | bash`，然后按照它自己的提示一步步来就行。用 Slack 比用飞书要简单一些，Slack 应用创建的步骤[按照这篇 blog 操作](https://heoffice.com/web/openclaw%E8%BF%9E%E6%8E%A5slack%E9%85%8D%E7%BD%AE%E5%AE%8C%E6%95%B4%E6%95%99%E7%A8%8B/)的（这个网站 https 证书过期了，忽略下警告就行）。

_其他模型、channel，后面再尝试_。

### Claude Code 和 Anthropic 充值

因为 Anthropic 没有对中国用户提供服务，所以我就搞了个新加坡的 VPS，直接在 VPS 上安装使用 Claude Code，而不是搭梯子在国内用。账号注册就是用的 Gmail 登录，不过全程访问都搭了新加坡的梯子，充值则是在闲鱼上找了个口碑极好的卖家代充。

## AI 考试篇

在我自己梳理清楚问题、找到根因后（详见[附录部分](/2026/02/27/AI-Coding-Evaluation/index.html#%E9%99%84%E5%BD%95%E9%97%AE%E9%A2%98%E8%AF%A6%E7%BB%86%E8%AF%B4%E6%98%8E)），我重新整理了下给 AI 的提问，如下：

> 帮我排查一下 kmp-socketio 这个项目一个偶现的 ut 失败问题，其中 good.log 是 ut case 通过时的完整日志，err.log 则是这个 case 失败时打印的日志。
>
> 我需要你细致分析日志里面关于 transport 实例、行为的过程，Transport polling@ 和 Transport websocket@ 开头的日志，@ 后面跟着的是 transport 实例的 hash 值，不同的 hash 值表明是不同的实例，请严谨地分析日志内容，以及代码里打印对应日志的逻辑，帮我彻底梳理确认整个测试用例的执行过程，以及 transport 状态的变化。
>
> 注意只做分析，不要进行修复。

直接原因很清楚，writeBuffer 是空却依然来了 drain 事件，关键问题是为什么会这样？

### Kimi Code (K2.5)

[Kimi Code 就没讲明白](https://github.com/HackWebRTC/kmp-socketio/blob/main/docs/ut-case-analysis/kimi-code-result-bad.txt)（甚至都没给出一个哪怕错误的明确结论）：

> 根本原因：EngineSocket.close() 和 probe() 的 upgrade 流程之间存在竞争条件。当 socket 正在关闭（有未发送数据等待 drain）时，如果 probe transport 成功完成，会触发升级流程。此时：
> 
> 1. upgrading 被设为 true
> 2. 原 polling transport 被 pause
> 3. 新 WebSocket transport 被设置并触发 flush
> 4. 这导致 onDrain 在错误的时机被调用，writeBuffer 和 prevBufferLen 状态不一致
> 5. 最终 ArrayDeque.removeFirst() 在空队列上抛出 NoSuchElementException

### OpenCode + K2.5

同样的 K2.5 模型，换成 [OpenCode（且有 Oh My OpenCode 加持），依然不行](https://raw.githubusercontent.com/HackWebRTC/kmp-socketio/refs/heads/main/docs/ut-case-analysis/opencode-kimi-result-bad.md)：说了跟没说一样，如说。

> 这是一个并发时序竞态问题：
> - 触发条件: 在 WebSocket 升级过程中（从 polling 切换到 websocket 时），如果发生 socket 关闭
> - 崩溃原因: onDrain 回调尝试从已被清空的 writeBuffer 移除元素
> - 偶现原因: 依赖于 IO 线程与工作线程的执行时序，只在特定时间窗口内触发

### OpenClaw + K2.5

[OpenClaw 的表现](https://github.com/HackWebRTC/kmp-socketio/blob/main/docs/ut-case-analysis/openclaw-kimi-result-good.md)其实也没太大差别：

> 这是一个**竞态条件 (Race Condition)** 导致的偶现问题：
>
> 1. **触发条件**：
>    - Socket 在 websocket probe 进行中时关闭
>    - WebSocket probe 在 socket 关闭后才完成
>    - Upgrade 流程在 close 等待期间完成
> 
> 2. **根本原因**：
>    - `onDrain` 方法假设 `writeBuffer` 中的元素数量与 `len` 参数匹配
>    - 但 transport 切换期间，异步事件可能导致 `onDrain` 被调用时 `writeBuffer` 已被清空
>    - `setTransport` 为新 transport 注册监听器，但可能接收到来自旧 transport 的异步事件
> 
> 3. **为何是偶现**：
>    - 取决于 websocket probe 完成和 socket 关闭的时序
>    - 如果 probe 在关闭前被中断，不会触发问题
>    - 如果 probe 在关闭后完成，就会触发异常
> 
> 4. **问题本质**：
>    - Transport 生命周期管理和事件处理的竞态条件
>    - 异步事件（`EVENT_DRAIN`）与同步状态（`writeBuffer`）不同步

不过我追问了下：但是在 EngineSocket.kt 的 probe 函数里，setTransport 应该是在 Upgrade 包对应的 `EVENT_DRAIN` 事件真的收到之后，才会执行的啊，也不用管 WebSocket.doSend 和 `EVENT_DRAIN` 触发是否异步，这个先后顺序是保证了的啊，为什么 `EVENT_DRAIN` 来了之后，还有 `EVENT_DRAIN` 呢？

然后它就开始不停猜测、推翻、再猜测，最终居然给出了正确的分析结果：

> probe 中发送了两个包：先 Ping，收到 Pong 后再 Upgrade
>
> 两次 doSend 是异步的：都在 ioScope.launch 中执行
>
> 完成顺序不确定：虽然 Ping 先发送，但 Upgrade 可能先完成（网络/调度原因）
>
> **两个独立的 emit**：setTransport 在第一个 emit 中执行，但第二个 emit 触发时新监听器已存在

这一点是出乎意料的，Kimi Code 和 Trae 我也追问过同样的问题，可是依然说不出个所以然来。

### Trae + K2.5

[Trae + K2.5 也不行](https://github.com/HackWebRTC/kmp-socketio/blob/main/docs/ut-case-analysis/trae-kimi-result-bad.md)：

> 这是一个**竞态条件（Race Condition）**问题，发生在以下场景：
> 
> 1. **触发条件**：第一次连接关闭时，websocket probe 恰好成功
> 2. **升级流程启动**：`probe` 方法中的 `onTransportOpen` 回调被触发，开始 transport 升级
> 3. **关闭流程与升级流程并发**：
>    - 关闭流程：调用 `close()` → `closeAction()` → 清空 writeBuffer
>    - 升级流程：发送 upgrade packet → 等待 drain 事件 → 切换 transport
> 4. **时序问题**：upgrade packet 发送完成（触发 drain 事件）发生在 writeBuffer 被清空之后

### Trae + GPT-5.2-Codex

[Trae + GPT-5.2-Codex 依然不行](https://github.com/HackWebRTC/kmp-socketio/blob/main/docs/ut-case-analysis/trae-gpt-5.2-codex-result-bad.md)：

> EngineSocket 把“transport drain 事件”当成了“自己 writeBuffer 的 drain”，但在 upgrade/close 交错时，transport 可能 drain 的是并不来自 writeBuffer 的包（Upgrade probe 包），从而导致 onDrain 对空队列执行 remove。

### Claude Code + Opus 4.6

[Claude Code + Opus 4.6 就厉害](https://raw.githubusercontent.com/HackWebRTC/kmp-socketio/refs/heads/main/docs/ut-case-analysis/cc-opus-result-good.md)了，提完问题开始默不作声地干了十几分钟，但最后给出的分析精准无比！

> 存在两层竞态条件：
> 
> 竞态 1：Close-drain vs Probe-pong
> - EngineSocket.close() 注册 drain handler，其中检查 upgrading 决定走哪条路径
> - probe() 中收到 pong 会设置 upgrading = true
> - 如果 drain 先于 pong → upgrading=false → 直接关闭（good.log）
> - 如果 pong 先于 drain → upgrading=true → 走 upgrade 路径，最终调用 setTransport(websocket)
> 
> 竞态 2：2probe-drain vs Upgrade-drain
> - websocket@260110639 的 doSend 被调用了两次：一次发 2probe，一次发 Upgrade
> - 每次 doSend 都在 ioScope.launch 中异步发送数据，完成后通过 scope.launch 发射 EVENT_DRAIN
> - 这两个 ioScope.launch 的完成顺序不确定
> - 如果 Upgrade 的 drain 先触发：upgradeHandler 消费了 once listener → setTransport 注册了新的持久 drain listener
> - 然后 2probe 的 drain 延迟触发：hit 到 setTransport 注册的新 listener → onDrain(1) 在空 writeBuffer 上操作 → 崩溃

不过贵也确实是贵，就分析这一个问题，就把 5h 限额用了一大半。细糠虽好，奈何太贵。

## 总结展望

1. 国内的模型和工具在面对复杂问题时，现在还力有不逮，当世最强的模型和工具确实让人瑟瑟发抖了。
2. 工具的差异不是很明显，应该主要还是模型的差异。
3. 不过差距大概也就半年，再过半年，国内的模型和工具也许就能达到 Claude Code + Opus 4.6 现在的水平了，到时再来重考下。

时不我待！

## 附录：问题详细说明

[偶现失败的 reconnectManually 用例](https://github.com/HackWebRTC/kmp-socketio/blob/23153d55f797e0ae68858494e047a0c5a3b9c2e7/kmp-socketio/src/jvmTest/java/io/socket/client/ConnectionTest.java#L354-L380)逻辑：
1. 创建 socket 并连接
2. 第一次 CONNECT 回调 → socket.close()
3. DISCONNECT 回调 → 注册新的 CONNECT 监听 → socket.open() (手动重连)
4. 第二次 CONNECT 回调 → socket.close()，向队列 offer("done")
5. values.take() 阻塞等待 "done"

偶现的失败问题堆栈：

```bash
Exception in thread "DefaultDispatcher-worker-10 @coroutine#1018" java.util.NoSuchElementException: ArrayDeque is empty.
	at kotlin.collections.ArrayDeque.removeFirst(ArrayDeque.kt:146)
	at com.piasy.kmp.socketio.engineio.EngineSocket.onDrain(EngineSocket.kt:280)
	at com.piasy.kmp.socketio.engineio.EngineSocket.access$onDrain(EngineSocket.kt:17)
	at com.piasy.kmp.socketio.engineio.EngineSocket$setTransport$1.call(EngineSocket.kt:246)
	at com.piasy.kmp.socketio.emitter.Emitter.emit$lambda$0$0(Emitter.kt:128)
	at com.piasy.kmp.socketio.emitter.Emitter.emit(Emitter.kt:136)
	at com.piasy.kmp.socketio.engineio.transports.WebSocket$doSend$2$3.invokeSuspend(WebSocket.kt:172)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:34)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:100)
	at kotlinx.coroutines.internal.LimitedDispatcher$Worker.run(LimitedDispatcher.kt:124)
	at kotlinx.coroutines.scheduling.TaskImpl.run(Tasks.kt:89)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:586)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:829)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:717)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:704)
	Suppressed: kotlinx.coroutines.internal.DiagnosticCoroutineContextException: [CoroutineId(1018), "coroutine#1018":StandaloneCoroutine{Cancelling}@21fcd395, siowkr]
```

[对应的代码](https://github.com/HackWebRTC/kmp-socketio/blob/23153d55f797e0ae68858494e047a0c5a3b9c2e7/kmp-socketio/src/commonMain/kotlin/com/piasy/kmp/socketio/engineio/EngineSocket.kt#L277-L281)：

```kotlin
private fun onDrain(len: Int) {
    Logging.debug(TAG) { "onDrain: prevBufferLen $prevBufferLen, writeBuffer.size ${writeBuffer.size}, len $len" }
    for (i in 1..len) {
        writeBuffer.removeAt(0)
    }
    ...
}
```

然后就没有第二次 CONNECT 回调了，values.take() 就会一直阻塞导致测试用例超时失败了。

首先介绍下核心类结构：

```bash
+---------+                       +---------+
| Socket  |-----------------------| Manager |
+---------+       holds `io`      +---------+
                                      | manages `nsps`
                                      |
                                      | holds `engine`
                                      |
                               +---------------+
                               | EngineSocket  |
                               +---------------+
                                      |
                                      | holds `transport`
                                      |
                                  +-----------+
                                  | Transport |
                                  +-----------+
                                      |
                        +-------------+-------------+
                        |                           |
                  +-------------+             +-----------+
                  | PollingXHR  |             | WebSocket |
                  +-------------+             +-----------+
```

再简单介绍下 SocketIO 的连接和关闭过程。

连接：
1. 先使用 http polling 的方式建立连接；
2. 连接建立后会发送 Connect 包；
3. 再 poll 到 server 返回的 Connect 包，就是连接成功了，会发出 CONNECT 回调；

同时建连过程还会触发连接升级，从 polling 升级到 websocket，升级过程为：
1. 建立 websocket 连接；
2. 连接成功后发送 Ping 包；
3. 收到 server 返回的 Pong 包后，发送 Upgrade 包；
4. Upgrade 包发送成功后，EngineSocket 就会把 transport 从 polling 替换为 websocket；

关闭：
1. 先发送 Disconnect 包，不用等发送完成就会发出 DISCONNECT 回调；
2. Manager 就会解除对 EngineSocket 实例的引用，下次再 open 就是创建新的 EngineSocket 实例了；
2. 但老的 EngineSocket/Transport 实例还会继续运转，polling/websocket 要等 Disconnect 包发送完成，polling 还要再额外发送一个 Close 包，完事之后才算关闭完成；

最后就是这个 onDrain 机制了：EngineSocket 会把正在发送中的包都缓存到 writeBuffer 里，等底下的 transport 发送完成（EVENT_DRAIN）后删除对应数量的包。这样重连就不会丢失包。出问题的地方就是 writeBuffer 为空了，但还来了个 len 为 1 的 onDrain，所以就触发异常了。

连接升级过程有这么一段[关键代码](https://github.com/HackWebRTC/kmp-socketio/blob/23153d55f797e0ae68858494e047a0c5a3b9c2e7/kmp-socketio/src/commonMain/kotlin/com/piasy/kmp/socketio/engineio/EngineSocket.kt#L444-L454)：

```kotlin
transport.once(EVENT_DRAIN, object : Listener {
    override fun call(vararg args: Any) {
        Logging.info(TAG, "upgrade packet send success")
        emit(EVENT_UPGRADE, transport)
        setTransport(transport)
        cleaned = true
        upgrading = false
        flush()
    }
})
transport.send(listOf(EngineIOPacket.Upgrade))
```

先等升级的 websocket 发送的 Upgrade 包发送完成（`EVENT_DRAIN`），才会把 EngineSocket 的 transport 切换到 websocket，再之后的 `EVENT_DRAIN` 才会触发 EngineSocket 的 onDrain。**这是所有失败的 AI 犯错的地方，他们都知道是 ws 的 drain 触发的问题，但明明是 drain 了才会 setTransport，怎么 setTransport 之后还会有 drain？**

实际上问题出在 Ping 包和 Upgrade 包的 drain 乱序了，或者说 Ping 包的 drain 来晚了：先发 Ping，然后发 Upgrade（此时就已经注册了上面的监听），然后不管是哪个包的 drain 先来，都会触发 setTransport，后来的 drain 就都会触发问题了。

详细的分析，可以见[这个文档](https://github.com/HackWebRTC/kmp-socketio/blob/main/docs/ut-case-analysis/kmp-socketio-ut-case-analysis.txt)。
