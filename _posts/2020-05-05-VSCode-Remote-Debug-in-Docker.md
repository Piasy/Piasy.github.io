---
layout: post
title: VSCode 远程调试 Docker 里的 OWT Server
tags:
    - OWT
---

不久前在群里听说了 VSCode 远程调试很厉害，但一直没试过，其实我认为，对于服务端程序，靠单步调试分析问题应该不是首选，主要还是得通过日志分析。不过如果是为了熟悉代码流程，那还是很有帮助的。之前我是通过静态看代码，再结合少量加日志，理清了音视频数据的完整转发、合成流程，但这次当我试图梳理 RTCP 流程时，代码看起来始终不得要领，于是想着干脆单步起来吧。直接上 gdb 命令行调试，肯定也可以，但现在都有了挖掘机，何必坚持用铲子呢 :)

我首先想尝试的是 SSH 远程调试，因为之前在群里看到的也是说的这种方式，不过实际上我的测试版本 OWT Server 是运行在 macOS 上的 docker 里的，而在 macOS 上基本不能通过 ssh 连接到 docker 实例。我再一搜「macos vscode remote debug c++ in docker」，发现 VSCode 对 docker 有直接的支持，这就很棒了。

摸索了几个小时之后，终于成功调试起来了我已有 docker 镜像里的 OWT Server，在这里分享给大家：

+ 启动 docker 镜像时，增加 `--cap-add=SYS_PTRACE --security-opt seccomp=unconfined` 选项，比如：

    ```bash
    # 原来的启动命令
    docker run --rm -v ~/src:/root/src \
      -p 4000:4000 -p 8080:8080 -p 3000:3000 \
      -p 3001:3001 -p 3002:3002 -p 3003:3003 \
      -p 3004:3004 --expose=20000-20050/udp -it \
      piasy/rtc-dev:v1.2 /bin/bash
    # 增加后的启动命令
    docker run --cap-add=SYS_PTRACE \
      --security-opt seccomp=unconfined \
      --rm -v ~/src:/root/src \
      -p 4000:4000 -p 8080:8080 -p 3000:3000 \
      -p 3001:3001 -p 3002:3002 -p 3003:3003 \
      -p 3004:3004 --expose=20000-20050/udp -it \
      piasy/rtc-dev:v1.2 /bin/bash
    ```

    不加的话，attach 时会报错 `ptrace: Operation not permitted`；

+ 按照[忘篱大神的教程，编译 debug 版 OWT Server，并修改相关超时配置](https://github.com/winlinvip/owt-docker#debug)：

    ```bash
    # 编译OWT时使用特殊的参数，完整编译步骤可以参考
    # https://blog.piasy.com/2019/04/14/OWT-Server-Quick-Start/index.html
    ./scripts/build.js -t mcu --check --debug

    # vi dist/webrtc_agent/agent.toml +3
    [agent]
    maxProcesses = 1 #default: 13
    prerunProcesses = 1 #default: 2

    # vi dist/webrtc_agent/nodeManager.js +125
    child.check_alive_interval = setInterval(function() {
    }, 3000000);

    # vi dist/webrtc_agent/amqp_client.js +10
    var TIMEOUT = 2000000;
    ```

+ 使用 `init-all.sh` 和 `start-all.sh` 脚本，把 OWT Server 跑起来；
+ 在 macOS 上打开 VSCode，安装 `Remote - Containers` 扩展；
+ 安装完 `Remote - Containers` 扩展后，VSCode 左下角会有「Remote Host」的图标：

    ![「Remote Host」的图标](https://imgs.piasy.com/2020-05-05-vscode-remote-host-icon.png)

    点击之后选择「Remote-Containers: Attach to Running Container...」，然后选择在跑着 OWT Server 的实例，之后 VSCode 会打开新的窗口，在新的窗口里，通过下面的路径来打开实例里的 OWT Server 项目：

    ![打开 docker 里的项目](https://imgs.piasy.com/2020-05-05-vscode-remote-open-folder-1.png)

    注意，点击左下角的「Remote Host」图标触发的「Remote-Containers: Open Folder/Workspace/Repository in Container」都不好使；

+ 然后我们需要在 docker 里安装 C++ 扩展，步骤如下：

    ![在 docker 里安装扩展](https://imgs.piasy.com/2020-05-05-vscode-remote-install-extension.png)

    搜出来之后，点击安装即可，应该会提示 reload window，因为我截图时已经安装过了，所以图片右下方可以看到已经安装了的这个扩展；注意打开项目、安装扩展，只需要手动操作一次，VSCode 会记住我们对同一个 docker 镜像（只要是同一镜像，不同次运行实例名字不同没关系）的操作，下次 attach 时，会自动打开项目、安装扩展；

+ 现在我们打开的是在 docker 实例里的 OWT Server 项目，所有的操作（修改文件，安装扩展）都是在修改 docker 实例的状态；我们需要在项目根目录，创建 `.vscode/launch.json` 文件，内容如下：

    ```json
    {
        "version": "0.2.0",
        "configurations": [
            {
                "name": "Launch Main",
                "type": "cppdbg",
                "request": "attach",
                "program": "/usr/local/lib/nodejs/node-v8.15.1-linux-x64/bin/node",
                "processId": "4810",
                "MIMode": "gdb",
                "setupCommands": [
                    {
                        "description": "Enable pretty-printing for gdb",
                        "text": "-enable-pretty-printing",
                        "ignoreFailures": true
                    }
                ]
            }
        ]
    }
    ```

    其中 `program` 为实际 node 程序的绝对路径，`processId` 为 webrtc-agent 工作进程的 pid，可通过如下命令获取：`ps aux|grep webrtc|grep workingNode|awk '{print $2}'`，记得每次重新启动 OWT Server 都要更新 `processId` 字段的值；

+ 好了，现在万事俱备，只欠 F5 了，那我们就按下 F5 键，即可开始单步调试啦！当然，别忘了在编辑器里添加断点 :)

    ![docker 远程调试](https://imgs.piasy.com/2020-05-05-vscode-remote-debug.png)

_我还没有深度进行调试，不过目前已经发现离开房间时，webrtc-agent 工作进程会挂掉，后面若有更多发现，我会在这里继续补充_。

Happy debugging :)
