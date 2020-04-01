---
layout: post
title: OWT Server 集群部署和扩缩容
tags:
    - 实时多媒体
    - WebRTC
    - OWT
---

一转眼 [OWT Server 快速入门](/2019/04/14/OWT-Server-Quick-Start/index.html)已经快一年了，最近终于遇到了单台机器无法支撑用户规模的情况，原本我乐观地认为 OWT 自动扩缩容是一件很简单的事情，但实际上这事一点也不简单。

## 单台机器性能

扩缩容的第一步就是要测量单台机器的性能，为此我按照实际使用场景进行了测试：一个房间两台设备，纯 SFU 模式，视频推流参数为 1280x720 2.5Mbps。

第一次测试是在 AWS m4.xlarge 实例（4 核 16G）上进行，CPU 占用以 webrtc-agent 为主，单核占用 10~20%（波动较大主要是客户端网络不稳定，无法持续观测），内存占用则小于 1%，至于带宽，两人房间有两路上传两路下载，各 5Mbps。照此估计，单台机器只能承载 20 个两人房间，而 20 个房间上下行带宽都只需要 100Mbps，小意思。

由于服务器性能瓶颈是 CPU，而且基本都在 webrtc-agent 上，所以我们可以把 webrtc-agent 单独部署，机器也可以选用 CPU 更强劲的 c5.2xlarge 实例（8 核 16G）。成功部署后，我进行了第二次测试，操作保持一致，此时 CPU 占用单核只有不到 6% 了，c 系列果然比 m 系列有显著优势。照此保守估计，单台机器至少可以承载 112 个两人房间（总体 CPU 占用率估计会到 85%）。112 个房间，上下行带宽需要 560Mbps，对于 c5.2xlarge 的高达 10Gbps 带宽，可以说是小菜一碟了。

[SRS 开源项目的作者忘篱在他的 MacBook Pro 上基于 Docker 对 OWT Server 做了一些性能测试](https://github.com/winlinvip/owt-docker#performance)，结果和 m4.xlarge 比较接近，大家也可以进行参考。

在我分享了我的测试结果后，忘篱指出：

> 一个房间占 6%，不代表三个房间占 18%，这里有个线性能力的问题，几百个线程的，要是能线性了，那也就真的奇迹再现。

但笔记本打开浏览器 tab 或多台手机测试，很容易就触发设备接入网带宽瓶颈，2.5Mbps 推流，实际上两人房间开到第三个就已经歇菜了。所以我就再开了两台 c5.2xlarge 的服务器运行 Selenium grids，用 KITE 来做测试。

下面分享几组测试结果，对 webrtc-agent 所在的机器，我使用了 htop 观察 CPU 占用情况，使用了 iftop 观察带宽情况。

### 2 人房间

| 房间数 | 推流确实有 2.5Mbps 时  | 推流只有 1Mbps 左右时 |
| ----- | -------------------- | ------------------ |
| 1     | 6~8%                 | 3~5%               |
| 3     | 8~10%                | 3~6%               |
| 5     | 8~10%                | 4~6%               |
| 10    | 9~11%                | 4~7%               |

表格里的百分比，是指单个 webrtc-agent 进程占用单个 CPU 核的比例。OWT web demo 里设置的推流码率始终都是 2.5Mbps，但有时所有客户端的推流码率确实都能到 2.5Mbps，有时却都只能到 1Mbps 左右，「推流确实有 2.5Mbps 时」和「推流只有 1Mbps 左右时」这两列指的就是这两种情况下，服务器的 CPU 占用情况。

具体的，1 个 2 人房间，「推流确实有 2.5Mbps 时」就是 5Mbps 的上下行带宽（服务器的带宽），此时只有一个 webrtc-agent 进程，占用单核 6~8% 的 CPU，「推流只有 1Mbps 左右时」就是 2Mbps 的上下行带宽，此时还是只有一个 webrtc-agent 进程，占用单核 3~5% 的 CPU；10 个这样的房间，「推流只有 1Mbps 左右时」就是 50Mbps 的上下行带宽，有 10 个 webrtc-agent 进程，每个进程占用单核 9~11% 的 CPU，总共就是 90~110% 的 CPU，而总可用 CPU 为 800%，「推流只有 1Mbps 左右时」每个进程占用单核 4~7% 的 CPU，总共就是 40~70% 的 CPU，总可用 CPU 为 800%。

测试过程中遇到了几个问题：

+ AWS c5.2xlarge 实例（8 核 16G）跑 Selenium grids，5 个 2 人房间就能把 CPU 占满，确实比较耗资源，难以大规模，为此我不得不再启动了一台 EC2，专门运行 node，[docker-compose.yaml 见 GitHub](https://github.com/HackWebRTC/KITE/blob/owt-test/docker-compose-node.yaml)；
+ 推流分辨率会从 320x180 开始，有时上调得比较慢，而我使用的视频文件，后半段编码基本都不会超过 1Mbps，所以如果一分钟内分辨率没有调到 720p，那就测不到最高负载了；
+ 10 个房间同时连，总有一些房间连不上，而「非 webrtc-agent 机器」只有一两秒 CPU 处于满载状态；每个 runner 都间隔 1.2s 启动（原来只有同房间的 runner 间隔 2s 启动），终于能都连上了；

### 3 人房间

| 房间数 | 推流确实有 2.5Mbps 时  | 推流只有 1Mbps 左右时 |
| ----- | -------------------- | ------------------ |
| 1     | 15~17%               | -                  |
| 3     | 18~20%               | 14~16%             |
| 5     | 19~22%               | 14~17%             |

测试 1 个房间时，推流码率始终都在 2.5Mbps 左右，看来码率不是文件的问题，是运行时状态的问题。

### 4 人房间

| 房间数 | 推流确实有 2.5Mbps 时  | 推流只有 1Mbps 左右时 |
| ----- | -------------------- | ------------------ |
| 1     | 27~31%               | -                  |
| 3     | 32~35%               | 24~26%             |
| 5     | 33~37%               | 24~27%             |

5 个房间的测了四次，都没成功，要么码率太低，要么视频 freeze/blank。

---

根据「2 人房间」的测试结果，如果单房间占单核 CPU 的占用率随着房间数的增长是线性的（不一定准确），到 20 个房间时，每个房间将占用最多 14% 的单核 CPU，30 房间时最多 17%，40 房间时最多 20%，此时总占用率将达到 800%，所以一台 c5.2xlarge 机器只能支持 40 个 2 人房间。

当然这只是推算，但真要测到这么大的并发，KITE 显然是比较吃力的，我在考虑为[基于 Kotlin multiplatform 的多平台 WebRTC SDK](/2019/12/05/Kmpp-WebRTC-SDK/index.html) 增加 Linux 版本，并且实现本地文件推流且不渲染，这样既能在服务器上运行，也无需编解码，减少资源消耗，进而减少压测所需资源，敬请期待。

## 集群部署

测试出来机器的性能后，下一步就是 webrtc agent 的集群部署了，这一步我遇见了不少 OWT Server 的问题。

### RabbitMQ Server

首先就是新部署的机器无法连接到 RabbitMQ Server，无论怎么修改 RabbitMQ Server 的配置，默认 guest 用户死活连不上，最后只能新建一个用户，命令如下：

```bash
rabbitmqctl add_user owt owt
rabbitmqctl set_user_tags owt administrator
rabbitmqctl set_permissions -p / owt ".*" ".*" ".*"
```

然后，我们修改 `agent.toml`：

```toml
[rabbit]
host = "172.16.25.194" #default: "localhost"
port = 5672 #default: 5672
login = "owt"
password = "owt"
```

host 填 RabbitMQ Server 所在服务器的局域网 IP 地址。同时，我们需要修改 RabbitMQ Server 所在服务器的防火墙配置，开放 TCP 5672 端口，但要注意不要开放给公网访问，即允许的源 IP 不要设置为 `0.0.0.0/0`，可以设置为机房 VPC 的局域网子网，比如 `172.16.0.0/16`。

如此，webrtc agent 就能连上 RabbitMQ Server 了。但当我测试时，却发现不通，查看 webrtc agent node 的日志，发现 node 没连上 RabbitMQ Server。经过一番加日志排查，发现是 node 启动时 RabbitMQ config 的 password 变成了空字符串，看代码发现是 agent 主进程连接成功之后，`amqp_client.js` 里的代码，把 config 中的 password 字段给删了：

```javascript
var conn = amqp.createConnection(options);
var connected = false;
conn.on('ready', function() {
    delete options.password;
    // ...
}
```

实际上 agent 的 `index.js` 里似乎考虑了这个问题，并做了 deep copy：

```javascript
var init_manager = () => {
  var reuseNode = !(myPurpose === 'audio'
    || myPurpose === 'video'
    || myPurpose === 'analytics'
    || myPurpose === 'conference'
    || myPurpose === 'sip');
  var consumeNodeByRoom = !(myPurpose === 'audio' || myPurpose === 'video' || myPurpose === 'analytics');

  var spawnOptions = {
    cmd: 'node',
    config: Object.assign({}, config)
  };
  spawnOptions.config.purpose = myPurpose;
  // ...
}
```

就是 `config: Object.assign({}, config)` 这一行。只可惜当这一行代码执行时，已经太晚了，`init_manager` 是在主进程连接 RabbitMQ Server 成功并注册之后，才会被调用，此时 password 已经被删掉了。

改成这样就好了：

```javascript
var rabbit_config_copy = Object.assign({}, config.rabbit);
var init_manager = () => {
  var reuseNode = !(myPurpose === 'audio'
    || myPurpose === 'video'
    || myPurpose === 'analytics'
    || myPurpose === 'conference'
    || myPurpose === 'sip');
  var consumeNodeByRoom = !(myPurpose === 'audio' || myPurpose === 'video' || myPurpose === 'analytics');

  var spawnOptions = {
    cmd: 'node',
    config: Object.assign({}, config)
  };
  spawnOptions.config.purpose = myPurpose;
  spawnOptions.config.rabbit = rabbit_config_copy;
  // ...
}
```

至此，webrtc agent 独立部署就通了。

### Internal IO

但在我继续测试时，偶然发现有多台 webrtc agent 机器时，似乎有时房间内的流无法互通，经过一番探索，最终定位到是一个房间里的不同用户，被分配到不同机器的 webrtc agent 时，就必现不通。查看 webrtc agent node 日志，发现有如下错误：

```bash
ERROR: owt.RawTransport - TCP wrote data error: Bad file descriptor
```

查看代码最终推测是跨机器传数据的 Internal IO 连接建立失败，查看 webrtc agent 的配置文件，发现是这么配置的：

```toml
[internal]
#The IP address used for internal-cluster media spreading. Will use the IP got from the 'network_interface' item if 'ip_address' is not specified or equal to "".
ip_address = "" #default: ""

#The network interface used for internal-cluster media spreading. The first enumerated network interface in the system will be adopted if this item is not specified.
# network_interface = "eth0" # default: undefined

# The internal listening port range, only works for TCP now
maxport = 0 #default: 0
minport = 0 #default: 0
```

而如果端口是 0 的话，那就会随机监听一个端口，那显然就会被防火墙拦住了，所以这里修改一下 `maxport` 和 `minport`，并在防火墙设置中开放对应的 TCP 端口范围即可。这里也不必开放给公网，只需开放给机房 VPC 的局域网子网即可。

至此，webrtc agent 集群部署算是彻底通了。

### 负载监控

实际上这个问题是在尝试扩缩容的时候发现的。我发现即便用 `dd if=/dev/zero of=/dev/null` 把 webrtc agent 所在的机器 CPU 打满，并且已经有新的机器注册到 cluster manager 了，但新的连接仍然调度到了老的机器上。

经过请教、加日志、看代码，最终发现 webrtc agent 监控的负载压根不是 CPU 占用率，而是网络带宽，而且这个逻辑是写死在代码里的，无法配置。

对于固定带宽的机器，使用带宽作为负载依据是合理的，但对于按流量付费的机器，经过前面的测试分析，我们已经知道瓶颈是 CPU，所以要用 CPU 作为负载依据才合适。为此就需要修改 agent 的 `configLoader.js` 了，原本代码长这样：

```javascript
config.cluster.worker.load.item = {
    name: 'network',
    interf: 'lo',
    max_scale: config.cluster.network_max_scale || 1000
};
```

改成如下即可实现可配置：

```javascript
config.cluster.worker.load.item = {};
config.cluster.worker.load.item.name = config.cluster.load_type || 'cpu';
if (config.cluster.worker.load.item.name === 'network') {
    config.cluster.worker.load.interf = 'lo';
    config.cluster.worker.load.max_scale = config.cluster.network_max_scale || 1000;
}
```

配置文件里如果没配置，那就默认用 CPU，如果要配置，可以这么写：

```toml
[cluster]
name = "owt-cluster"

#The number of times to retry joining if the first try fails.
join_retry = 60 #default: 60

#The interval of reporting the work load
report_load_interval = 1000 #default: 1000, unit: millisecond

#The load type, e.g. cpu, disk, network
load_type = "cpu"
#The max load under which this worker can take new tasks.
max_load = 0.85 #default: 0.85
```

---

上面提到的修改，都[已经提交 PR，欢迎关注](https://github.com/open-webrtc-toolkit/owt-server/pull/455)。

## 扩缩容

OWT 本身的坑就已经趟平了，接下来到了趟 AWS 坑的时候了。

我原本以为 AWS 的 auto scaling group 可以满足需求，但实际测试，它的策略如下：

+ 创建时可以指定「维持 CPU 占用率」（比如 85%）、最大机器数、最小机器数、期望机器数；
+ 运行时只要有一台机器超过 85%，就会不停启新机器，直到达到期望机器数；
+ 而机器总数量一旦超过期望机器数（比如现有机器任一台 CPU 超过 85%，就会触发新起一台机器，此时就超过一台了），就会随机关掉机器，以维持在期望机器数；

这种策略对于短连接 API 倒是很合适，但长连接就不行了。先不管「只要有一台机器超过 85%，就会不停启新机器」会造成浪费，单凭「随机关掉机器」就可以否掉这个方案了，只要机器还有活跃连接，那就不能关。

其实对于「只要有一台机器超过 85% 就不停启新机器会造成浪费」的问题，倒也不是无解，修改 cluster manager 对 webrtc agent 的调度策略，可以一定程度缓解这个问题。目前默认的调度策略是 last-used，即一直使用一台机器，直到负载超过 max load，修改为 least-used，可以让多台机器的负载基本一致。不过这也有个坏处，如果同一个房间的连接调度到了不同的机器上，那就需要通过 Internal IO 进行数据传输，会增加一定的延迟，也会增加一点开销。

现有轮子不可用，就得考虑如何造轮子了，我初步设想的方案为：

+ webrtc agent 调度策略保持 last-used 不变；
+ 由 cluster manager 负责决策（或提供负载信息和控制接口，由独立程序决策）何时启新机器、何时关老机器；
+ 一个比较简单的决策策略为：当所有机器平均 CPU 占用率超过启动阈值时，启动新机器，低于关闭阈值时，关闭老机器；
+ 新机器可以直接启动，老机器不能直接关闭，当决定关闭老机器时，不再向它调度新的连接，等它的所有连接都断开后，再关闭之；

如此，需要扩展 cluster manager，并调用 AWS 等云商的 SDK，进行机器的启停。这个轮子做起来需要一点时间，在此之前，可以先手动进行扩缩容操作，加个 CPU 报警即可，虽然不完美，但也能转起来。
