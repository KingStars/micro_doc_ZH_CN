# Micro文档

概述：Micro是一个简化分布式开发的微服务生态系统。它为开发分布式应用程序提供了基本的构建模块。这篇Micro文档做为采用Micro的参考指南。

## 介绍
Micro是一个微服务生态系统。目标是简化分布式系统开发。

技术正在迅速发展。现在云计算能够给我们几乎是无限的scale能力，但是采用现有工具来使用scale能力仍然是很困难的。Micro试图去解决这个问题，开发人员首先关注。

Micro的核心是简单易用，任何人都可以轻松开始编写微服务。随着您扩展到数百种服务，Micro将提供管理微服务环境所需的基本工具

## 开始使用
如果你想开始写微服务，直接去[go-micro](https://github.com/micro/go-micro)仓库。

## 概述

提供的主要软件是[Micro](https://github.com/micro/micro)，一个微服务工具包。

该工具包由以下组件组成：

- **Go Micro** - 用于在Go中编写微服务的插件式RPC框架。它提供了用于服务发现，客户端负载平衡，编码，同步和异步通信库。

- **API** - 提供并将HTTP请求路由到相应微服务的API网关。它充当单个入口点，可以用作反向代理或将HTTP请求转换为RPC。

- **Sidecar** - 一种对语言透明的RPC代理，具有go-micro作为HTTP端点的所有功能。虽然Go是构建微服务的伟大语言，但您也可能希望使用其他语言，因此Sidecar提供了一种将其他应用程序集成到Micro世界的方法。

- **Web** - 用于Micro Web应用程序的仪表板和反向代理。我们认为应该基于微服务建立web应用，因此被视为微服务领域的一等公民。它的行为非常像API反向代理，但也包括对web sockets的支持。

- **CLI** - 一个直接的命令行界面来与你的微服务进行交互。它还使您可以利用Sidecar作为代理，您可能不想直接连接到服务注册表。

- **Bot** - Hubot风格的bot，位于您的微服务平台中，可以通过Slack，HipChat，XMPP等进行交互。它通过消息传递提供CLI的功能。可以添加其他命令来自动执行常见的操作任务。

*注意：Go-micro是一个独立的库，可以独立于其他工具包使用。*

## 运行时
该工具包是可插入式并运行时不感知。在笔记本电脑基于docker，使用kubernetes上运行micro或者AWS等等。

![](overview.png)

## 了解更多
浏览此文档以了解更多信息，查看下面的资源或尝试一些[示例](https://github.com/micro/examples)。

## 资源
- 阅读[博客](https://micro.mu/blog/)，深入了解Micro和更广泛的微服务理念。
- 在Golang UK Conf 2016上观看Micro简化微服务的[视频](https://www.youtube.com/watch?v=xspaDovwk34)。
- 查看演讲台上各种演示的[幻灯片](https://speakerdeck.com/asim)。

## 赞助商
Micro开源开发是由![](sixt_logo.png)赞助
