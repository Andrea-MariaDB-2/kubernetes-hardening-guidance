# 日志审计

日志记录了集群中的活动。审计日志是必要的，这不仅是为了确保服务按预期运行和配置，也是为了确保系统的安全。系统性的审计要求对安全设置进行一致和彻底的检查，以帮助识别潜在威胁。Kubernetes 能够捕获集群操作的审计日志，并监控基本的 CPU 和内存使用信息；然而，它并没有提供深入的监控或警报服务。

**关键点**

- 在创建时建立 Pod 基线，以便能够识别异常活动。
- 在主机层面、应用层面和云端（如果适用）进行日志记录。
- 整合现有的网络安全工具，进行综合扫描、监控、警报和分析。
- 设置本地日志存储，以防止在通信失败的情况下丢失。

## 日志

在 Kubernetes 中运行应用程序的系统管理员应该为其环境建立一个有效的日志、监控和警报系统。仅仅记录 Kubernetes 事件还不足以了解系统上发生的行动的全貌。还应在主机级、应用级和云上（如果适用）进行日志记录。而且，这些日志可以与任何外部认证和系统日志相关联，以提供整个环境所采取的行动的完整视图，供安全审计员和事件响应者使用。

在 Kubernetes 环境中，管理员应监控 / 记录以下内容：

- API 请求历史
- 性能指标
- 部署情况
- 资源消耗
- 操作系统调用
- 协议、权限变化
- 网络流量

当一个 Pod 被创建或更新时，管理员应该捕获网络通信、响应时间、请求、资源消耗和任何其他相关指标的详细日志以建立一个基线。正如上一节所详述的，匿名账户应被禁用，但日志策略仍应记录匿名账户采取的行动，以确定异常活动。

应定期审计 RBAC 策略配置，并在组织的系统管理员发生变化时进行审计。这样做可以确保访问控制的调整符合基于角色的访问控制部分中概述的 RBAC 策略加固指导。

审计应包括将当前日志与正常活动的基线测量进行比较，以确定任何日志指标和事件的重大变化。系统管理员应调查重大变动——例如，应用程序使用的变化或恶意程序的安装，如密码器，以确定根本原因。应该对内部和外部流量日志进行审计，以确保对连接的所有预期的安全限制已被正确配置，并按预期运行。管理员还可以在系统发展过程中使用这些审计，以确定何时不再需要外部访问并可以限制。

日志可以导向外部日志服务，以确保集群外的安全专业人员的可用使用它们，尽可能接近实时地识别异常情况，并在发生损害时保护日志不被删除。如果使用这种方法，日志应该在传输过程中用 TLS 1.2 或 1.3 进行加密，以确保网络行为者无法在传输过程中访问日志并获得关于环境的宝贵信息。在利用外部日志服务器时，要采取的另一项预防措施是在 Kubernetes 内配置日志转发器，只对外部存储进行追加访问。这有助于保护外部存储的日志不被删除或被集群内日志覆盖。

## Kubernetes 原生审计日志配置

> Kubernetes 的审计功能默认是禁用的，所以如果没有写审计策略，就不会有任何记录。

`kube-apiserver` 驻留在 Kubernetes 控制平面上，作为前端，处理集群的内部和外部请求。每个请求，无论是由用户、应用程序还是控制平面产生的，在其执行的每个阶段都会产生一个审计事件。当审计事件注册时，`kube-apiserver` 检查审计策略文件和适用规则。如果存在这样的规则，服务器会在第一个匹配的规则所定义的级别上记录该事件。Kubernetes 的内置审计功能默认是不启用的，所以如果没有写审计策略，就不会有任何记录。

集群管理员必须写一个审计策略 YAML 文件，以建立规则，并指定所需的审计级别，以记录每种类型的审计事件。然后，这个审计策略文件被传递给 `kube-apiserver`，并加上适当的标志。一个规则要被认为是有效的，必须指定四个审计级别中的一个：`none`、`Meatadataa`、`Request` 或 `RequestResponse`。[**附录 L：审计策略**](http://localhost:4000/appendix/l.html)展示了一个审计策略文件的内容，该文件记录了 `RequestResponse` 级别的所有事件。[**附录 M**](http://localhost:4000/appendix/m.html) 向 `kube-apiserver` 提交审计策略文件的标志示例显示了 `kube-apiserver` 配置文件的位置，并提供了审计策略文件可以被传递给 `kube-apiserver` 的标志示例。附录 M 还提供了如何挂载卷和在必要时配置主机路径的指导。

`kube-apiserver` 包括可配置的日志和 webhook 后端，用于审计日志。日志后端将指定的审计事件写入日志文件，webhook 后端可以被配置为将文件发送到外部 HTTP API。附录 M 中的例子中设置的 `--audit-log-path` 和 `--audit-log-maxage` 标志是可以用来配置日志后端的两个例子，它将审计事件写到一个文件中。`log-path` 标志是启用日志的最小配置，也是日志后端唯一需要的配置。这些日志文件的默认格式是 JSON，尽管必要时也可以改变。日志后端的其他配置选项可以在 Kubernetes 文档中找到。

为了将审计日志推送给组织的 SIEM 平台，可以通过提交给 `kube-apiserver` 的 YAML 文件手动配置 webhook 后端。webhook 配置文件以及如何将该文件传递给 `kube-apiserver` 可以在[**附录 N：webhook 配置**](http://localhost:4000/appendix/n.html)的示例中查看。关于如何在 `kube-apiserver` 中为 webhook 后端设置的配置选项的详尽列表，可以在 Kubernetes 文档中找到。

### 工作节点和容器的日志记录

在 Kubernetes 架构中，有很多方法可以配置日志功能。在日志管理的内置方法中，每个节点上的 kubelet 负责管理日志。它根据其对单个文件长度、存储时间和存储容量的策略，在本地存储和轮转日志文件。这些日志是由 kubelet 控制的，可以从命令行访问。下面的命令打印了一个 Pod 中的容器的日志。

```sh
kubectl logs [-f] [-p] POD [-c CONTAINER]
```

如果要对日志进行流式处理，可以使用 `-f` 标志；如果存在并需要来自容器先前实例的日志，可以使用 `-p` 标志；如果 Pod 中有多个容器，可以使用 `-c` 标志来指定一个容器。如果发生错误导致容器、Pod 或节点死亡，Kubernetes 中的本地日志解决方案并没有提供一种方法来保存存储在失败对象中的日志。NSA 和 CISA 建议配置一个远程日志解决方案，以便在一个节点失败时保存日志。

远程记录的选项包括：

| 远程日志选项                                                 | 使用的理由                                                   | 配置实施                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 在每个节点上运行一个日志代理，将日志推送到后端               | 赋予节点暴露日志或将日志推送到后端的能力，在发生故障的情况下将其保存在节点之外。 | 配置一个 Pod 中的独立容器作为日志代理运行，让它访问节点的应用日志文件，并配置它将日志转发到组织的 SIEM。 |
| 在每个 Pod 中使用一个 sidecar 容器，将日志推送到一个输出流中 | 用于将日志推送到独立的输出流。当应用程序容器写入不同格式的多个日志文件时，这可能是一个有用的选项。 | 为每种日志类型配置 sidecar 容器，并用于将这些日志文件重定向到它们各自的输出流，在那里它们可以被 kubelet 处理。然后，节点级的日志代理可以将这些日志转发给 SIEM 或其他后端。 |
| 在每个 Pod 中使用一个日志代理 sidecar，将日志推送到后端      | 当需要比节点级日志代理所能提供的更多灵活性时。               | 为每个 Pod 配置，将日志直接推送到后端。这是连接第三方日志代理和后端的常用方法。 |
| 从应用程序中直接向后端推送日志                               | 捕获应用程序的日志。Kubernetes 没有内置的机制直接来暴露或推送日志到后端。 | 各组织将需要在其应用程序中建立这一功能，或附加一个有信誉的第三方工具来实现这一功能。 |

Sidecar 容器与其他容器一起在 Pod 中运行，可以被配置为将日志流向日志文件或日志后端。Sidecar 容器也可以被配置为作为另一个标准功能容器的流量代理，它被打包和部署。

为了确保这些日志代理在工作节点之间的连续性，通常将它们作为 DaemonSet 运行。为这种方法配置 DaemonSet，可以确保每个节点上都有一份日志代理的副本，而且对日志代理所做的任何改变在集群中都是一致的。

### Seccomp: 审计模式

除了上述的节点和容器日志外，记录系统调用也是非常有益的。在 Kubernetes 中审计容器系统调用的一种方法是使用安全计算模式（seccomp）工具。这个工具默认是禁用的，但可以用来限制容器的系统调用能力，从而降低内核的攻击面。Seccomp 还可以通过使用审计配置文件记录正在进行的调用。

自定义 seccomp 配置文件用于定义哪些系统调用是允许的，以及未指定调用的默认动作。为了在 Pod 中启用自定义 seccomp 配置文件，Kubernetes 管理员可以将他们的 seccomp 配置文件 JSON 文件写入到 `/var/lib/kubelet/seccomp/` 目录，并将 `seccompProfile` 添加到 Pod 的 `securityContext`。自定义的 `seccompProfile` 还应该包括两个字段。`Type: Localhost` 和 `localhostProfile: myseccomppolicy.json`。记录所有的系统调用可以帮助管理员了解标准操作需要哪些系统调用，使他们能够进一步限制 seccomp 配置文件而不失去系统功能。

### SYSLOG

Kubernetes 默认将 kubelet 日志和容器运行时日志写入 journald，如果该服务可用的话。如果组织希望对默认情况下不使用的系统使用 syslog 工具，或者从整个集群收集日志并将其转发到 syslog 服务器或其他日志存储和聚合平台，他们可以手动配置该功能。Syslog 协议定义了一个日志信息格式化标准。Syslog 消息包括一个头——由时间戳、主机名、应用程序名称和进程 ID（PID）组成，以及一个以明文书写的消息。Syslog 服务，如 syslog-ng® 和 rsyslog，能够以统一的格式收集和汇总整个系统的日志。许多 Linux 操作系统默认使用 rsyslog 或 journald——一个事件日志守护程序，它优化了日志存储并通过 journalctl 输出 syslog 格式的日志。在运行某些 Linux 发行版的节点上，syslog 工具默认在操作系统层面记录事件。运行这些 Linux 发行版的容器，默认也会使用 syslog 收集日志。由 syslog 工具收集的日志存储在每个适用的节点或容器的本地文件系统中，除非配置了一个日志聚合平台来收集它们。

## SIEM 平台

安全信息和事件管理（SIEM）软件从整个组织的网络中收集日志。SIEM 软件将防火墙日志、应用程序日志等汇集在一起；将它们解析出来，提供一个集中的平台，分析人员可以从这个平台上监控系统安全。SIEM 工具在功能上有差异。一般来说，这些平台提供日志收集、威胁检测和警报功能。有些包括机器学习功能，可以更好地预测系统行为并帮助减少错误警报。在其环境中使用这些平台的组织可以将它们与 Kubernetes 集成，以更好地监测和保护集群。用于管理 Kubernetes 环境中的日志的开源平台是作为 SIEM 平台的替代品存在的。

容器化环境在节点、Pod、容器和服务之间有许多相互依赖的关系。在这些环境中，Pod 和容器不断地在不同的节点上被关闭和重启。这给传统的 SIEM 带来了额外的挑战，它们通常使用 IP 地址来关联日志。即使是下一代的 SIEM 平台也不一定适合复杂的 Kubernetes 环境。然而，随着 Kubernetes 成为最广泛使用的容器编排平台，许多开发 SIEM 工具的组织已经开发了专门用于 Kubernetes 环境的产品变化，为这些容器化环境提供全面的监控解决方案。管理员应该了解他们平台的能力，并确保他们的日志充分捕捉到环境，以支持未来的事件响应。

## 警报

Kubernetes 本身并不支持警报功能；然而，一些具有警报功能的监控工具与 Kubernetes 兼容。如果 Kubernetes 管理员选择配置一个警报工具在 Kubernetes 环境中工作，有几个指标是管理员应该监控和配置警报的。

可能触发警报的案例包括但不限于：

- 环境中的任何机器上的磁盘空间都很低。
- 记录卷上的可用存储空间正在减少。
- 外部日志服务脱机。
- 一个以 root 权限运行的 Pod 或应用程序。
- 一个账户对他们没有权限的资源提出的请求。
- 一个正在使用或获得特权的匿名账户。
- Pod 或工作节点的 IP 地址被列为 Pod 创建请求的源 ID。
- 异常的系统调用或失败的 API 调用。
- 用户 / 管理员的行为不正常（即在不寻常的时间或从不寻常的地点），以及
- 显著偏离标准操作指标基线。

当存储不足时发出警报，可以帮助避免因资源有限而导致的性能问题和日志丢失，并帮助识别恶意的加密劫持企图。可以调查有特权的 Pod 执行案例，以确定管理员是否犯了一个错误，一个真实的用例需要升级特权，或者一个恶意行为者部署了一个有特权的 Pod。可疑的 Pod 创建源 IP 地址可能表明，恶意的网络行为者已经突破了容器并试图创建一个恶意的 Pod。

将 Kubernetes 与企业现有的 SIEM 平台整合，特别是那些具有机器学习 / 大数据功能的平台，可以帮助识别审计日志中的违规行为并减少错误警报。如果配置这样的工具与 Kubernetes 一起工作，它应该被配置为这些情况和任何其他适用于用例的情况被配置为触发警报。

当疑似入侵发生时，能够自动采取行动的系统有可能被配置为在管理员对警报作出反应时采取步骤以减轻损害。在 Pod IP 被列为 Pod 创建请求的源 ID 的情况下，一个可以实施的缓解措施是自动驱逐 Pod，以保持应用程序的可用性，但暂时停止对集群的任何损害。这样做将允许一个干净的 Pod 版本被重新安排到一个节点上。然后，调查人员可以检查日志，以确定是否发生了漏洞，如果是的话，调查恶意行为者是如何执行潜在威胁的，以便可以部署一个补丁。

## 服务网格

服务网格是一个平台，通过允许将这些通信逻辑编码到服务网格中，而不是在每个微服务中，来简化应用程序中的微服务通信。将这种通信逻辑编码到各个微服务中是很难扩展的，当故障发生时很难调试，而且很难保证安全。使用服务网格可以简化开发人员的工作。服务网格可以：

- 当一个服务中断时，重新定向流量。
- 收集性能指标以优化通信。
- 允许管理服务与服务之间的通信加密。
- 收集服务间通信的日志。
- 从每个服务中收集日志。
- 帮助开发者诊断微服务或通信机制的问题和故障。

服务网格还可以帮助将服务迁移到混合或多云环境。虽然服务网格不是必须的，但它们是一种高度适合 Kubernetes 环境的选择。托管的 Kubernetes 服务通常包括他们自己的服务网格。然而，其他几个平台也是可用的，如果需要的话，是可以高度定制的。其中一些包括一个生成和轮换证书的证书颁发机构，允许服务之间进行安全的 TLS 认证。管理员应该考虑使用服务网格来加强 Kubernetes 集群的安全性。

![图5：集群利用服务网格，将日志与网络安全结合起来](images/f5.jpg)

## 容错性

应制定容错策略，以确保日志服务的可用性。这些策略可以根据具体的 Kubernetes 用例而有所不同。一个可以实施的策略是，如果在存储容量超标的情况下，绝对有必要允许新的日志覆盖最旧的日志文件。

如果日志被发送到外部服务，应该建立一种机制，以便在发生通信中断或外部服务故障时将日志存储在本地。一旦与外部服务的通信恢复，应制定策略，将本地存储的日志推送到外部服务器上。

## 工具

Kubernetes 不包括广泛的审计功能。然而，该系统的构建是可扩展的，允许用户自由开发自己的定制解决方案，或选择适合自己需求的现有附加组件。最常见的解决方案之一是添加额外的审计后端服务，它可以使用 Kubernetes 记录的信息，并为用户执行额外的功能，如扩展搜索参数、数据映射功能和警报功能。已经使用 SIEM 平台的企业可以将 Kubernetes 与这些现有的功能进行整合。

开源监控工具，如 Cloud Native Computing Foundation 的 Prometheus®、Grafana Labs 的 Grafana® 和 Elasticsearch 的 Elastic Stack (ELK)®—— 可用于进行事件监控、运行威胁分析、管理警报，以及收集资源隔离参数、历史使用情况和运行容器的网络统计数据。在审计访问控制和权限配置时，扫描工具可以通过协助识别 RBAC 中的风险权限配置而发挥作用。NSA 和 CISA 鼓励在现有环境中使用入侵检测系统（IDS）的组织考虑将该服务也整合到他们的 Kubernetes 环境中。这种整合将使企业能够监测并有可能杀死有异常行为迹象的容器，从而使容器能够从最初的干净镜像中重新启动。许多云服务提供商也为那些希望得到更多管理和可扩展解决方案的人提供容器监控服务。