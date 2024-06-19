# nftables简介

## 什么是nftables

nftables 是 Linux 内核的一部分，用作构建、管理和应用数据包过滤规则的框架。它**是 iptables 的后继者**，提供了**更现代、更高效的方式来管理网络数据包的过滤和路由。**

nftables 的作用**不仅限于网络层**（OSI 模型的第三层），它实际上在 OSI 模型中的多个层次上都可以发挥作用。具体来说，nftables 可以处理从网络层（例如，IP 地址过滤）到传输层（例如，TCP/UDP 端口过滤）的规则，甚至可以处理某些应用层的规则（通过深度包检测）。这使得 nftables 成为一个非常灵活和强大的工具，能够满足**多种网络过滤和数据包处理**的需求。

nftables 通过一个统一的框架提供了对**各种协议**的支持，包括 IPv4、IPv6、ARP、以及以太网框架等。用户可以通过 **nft 命令行工具或配置文件来定义规则集**，这些规则集定义了如何处理经过网络接口的数据包。因此，可以说 nftables 的作用跨越了 OSI 模型的多个层次，提供了从网络层到传输层（甚至部分应用层）的数据包过滤和管理能力。

# nftables的基本操作

## nft的基本指令

### 表操作

#### 添加表

```bash
% nft add table [<family>] <name>
```

- 添加ipv4

  - 

  - ```bash
    % nft add table ip foo
    ```

- 添加ipv6

  - 

  - ```bash
    % nft add table ip6 bar
    ```

#### 列出表

```bash
% nft list tables
table ip foo
table ip6 bar
```

你也可以列出每个族的表：

```bash
% nft list tables ip
table ip foo
#默认ipv4族
```

然后你可以列出表的内容：

```bash
% nft list table ip filter
table ip filter {
}
```

#### 删除表

```bash
% nft delete table ip foo
```

故障处理：从 Linux 内核 3.18 开始，你可以用这个命令删除表和它的内容。然而，在这之前，你需要先删除表中的内容，否则你将会遇到下面的错误：

```bash
% nft delete table filter
<cmdline>:1:1-19: Error: Could not delete table: Device or resource busy
delete table filter
^^^^^^^^^^^^^^^^^^^
```

#### 清空表

你可以使用下面的命令删除表中所有的规则：

```bash
% nft flush table ip filter
```

#### 打印规则集

```bash
% nft list ruleset 
```

#### 总结

- add table
- delete table
- list tables
- list ruleset 
- flush table

### 链操作

添加基链的语法为：

```
% nft add chain [<family>] <table_name> <chain_name> { type <type> hook <hook> priority <value> \; [policy <policy> \;] [comment \"text comment\" \;] }
```

以下示例演示如何向 foo 表（必须已创建）添加新的基链输入：

```
% nft 'add chain ip foo input { type filter hook input priority 0 ; }'
```

优先级很重要，因为它决定了**链的顺序**，因此，如果输入钩子中有多个链，则可以决定哪个链在另一个链之前看到数据包。例如，优先级为 -12、-1、0、10 的输入链将完全按照该顺序进行查询。可以为两条基础链提供相同的优先级，但对于连接到同一挂钩位置的具有相同优先级的基础链，没有保证的评估顺序。

如果要使用 nftables 来过滤桌面 Linux 计算机（即**不转发流量的计算机**）的流量，您还可以注册输出链：

```
% nft 'add chain ip foo output { type filter hook output priority 0 ; }'
```

- **`type filter`**：指定链的类型为 `filter`，意味着这个链用于过滤数据包。
- **`hook output`**：指定这个链挂钩（hook）到输出路径上，即处理即将从本机发送出去的数据包。
- **`priority 0`**：设置这个链在输出路径上的优先级为 `0`。nftables 允许为不同的链设置优先级，以确定它们被评估的顺序。数字越小，优先级越高，越早被处理。

现在，您可以筛选传入（定向到本地进程）和传出（由本地进程生成）流量。

#### 链类型

##### Base chain types 基链类型

- **filter**用于过滤数据包。arp、bridge、ip、ip6 和 inet 表系列都支持此功能。
  - **route**用于在修改任何相关 IP 标头字段或数据包标记时重新路由数据包。如果您熟悉 iptables，则此链类型提供与 mangle 表等效的语义，但仅适用于输出钩子（对于其他钩子，请改用类型过滤器）。ip、ip6 和 inet 表系列支持此功能。
- **nat**, nat，用于执行网络地址转换 （NAT）。只有给定流的第一个数据包到达此链;后续数据包会绕过它。因此，切勿使用此链进行过滤。nat 链类型由 ip、ip6 和 inet 表系列支持。

##### Base chain hooks 底链钩

- **ingress** 仅在 Linux 内核 4.2 之后的 netdev 系列中，以及 Linux 内核 5.10 之后的 inet 系列中）：在数据包从 NIC 驱动程序传递后立即查看数据包，甚至在预路由之前。所以你有一个替代 tc 的替代品。
- **prerouting**: 在做出任何路由决策之前查看所有传入数据包。数据包可以寻址到本地或远程系统。
- **input**: 查看寻址到本地系统和运行该本地系统和进程的传入数据包。
- **forward**: 查看未寻址到本地系统的传入数据包。
- **output**: 查看源自本地计算机进程的数据包。
- **postrouting**: 在路由后，在数据包离开本地系统之前查看所有数据包。

>在 nftables 中，"底链"（Base Chains）或"基本链"**是一种特殊类型的链**，直接与网络堆栈的不同部分相连接。这些底链通过所谓的"钩子"（Hooks）与网络堆栈挂钩，允许它们在特定的处理点捕获数据包进行处理。这种设计使得 nftables 能以灵活的方式应用各种过滤规则和策略。
>
>### 钩子（Hooks）
>
>钩子是网络堆栈内的预定点，它们定义了数据包流经网络堆栈时可以被底链捕获和处理的时机。nftables 中的钩子与数据包的处理阶段相关联，例如当数据包到达网络接口、即将离开网络接口、或在本地进程生成数据包时等。通过这些钩子，nftables 可以在数据包的生命周期的关键时刻进行拦截和处理。
>
>### 底链钩的种类
>
>在 nftables 中，常见的钩子种类包括：
>
>- **prerouting**：用于处理刚到达网络接口的入站数据包，在路由决策之前。
>- **input**：用于处理确定目标为本地进程的入站数据包，在路由决策之后。
>- **forward**：用于处理经过本机但目标为其他设备的数据包。
>- **output**：用于处理由本地进程生成，准备发送出去的数据包，在路由决策之后。
>- **postrouting**：用于处理即将离开网络接口的出站数据包，在路由决策之后。
>
>每个底链通过指定类型、钩子和优先级来进行配置。类型（如 filter、nat 等）定义了链的角色，钩子定义了链挂钩网络堆栈的位置，而优先级则决定了在同一钩子点上的多个链的处理顺序。
>
>### 底链的作用
>
>底链的作用是在数据包处理流程中的特定点应用规则集，从而实现包过滤、地址转换、负载分配等功能。这些底链和钩子结合使用，为构建复杂的网络处理规则提供了强大且灵活的框架。
>
>nftables 通过这种底链和钩子的机制，为Linux提供了一个高效、灵活的包处理和网络流量控制系统，它比老旧的iptables系统提供了更好的性能和更简洁的配置语法。

##### Base chain priority 基链优先级

示例：

```bash
table inet filter {
        // 由于优先级，这个链首先被评估
        chain services {
                type filter hook input priority 0; policy accept;

                // 如果匹配，此规则将阻止任何进一步的评估
                tcp dport http drop

                // 如果匹配，尽管接受裁决，数据包仍然进入下面的链
                tcp dport ssh accept

                // 对于任何到达这里并触发默认策略的数据包也是如此
        }

        // 由于优先级，这个链最后被评估
        chain input {
                type filter hook input priority 1; policy drop;
                // 所有进入的数据包最终都会在这里被丢弃！
        }
}
```

解释如下：
```bash
这段代码定义了两个链（chain）：services和input。每个链都有一组规则，用于决定如何处理通过该链的数据包。

services链的优先级为0，因此它会首先被评估。它的默认策略是接受（accept）所有数据包，但如果数据包的目标端口是HTTP（tcp dport http），则会被丢弃（drop）。如果数据包的目标端口是SSH（tcp dport ssh），则会被接受，但仍然会进入下一个链进行评估。

input链的优先级为1，因此它会在services链之后被评估。它的默认策略是丢弃所有数据包，因此所有到达这个链的数据包都会被丢弃。
```

#### 常规链操作

##### 创建常规链

您还可以创建常规链，类似于 iptables 用户定义的链：

```
# nft -i
nft> add chain [family] <table_name> <chain_name> [{ [policy <policy> ;] [comment "text comment about this chain" ;] }]
```

##### 删除链

```
% nft delete chain [family] <table_name> <chain_name>
```

唯一的条件是要删除的链需要为空，否则内核会抱怨链还在使用中。

```
% nft delete chain ip foo input
<cmdline>:1:1-28: Error: Could not delete chain: Device or resource busy
delete chain ip foo input
```

##### 冲洗链

> 就是将链中的所有规则全部删了

```
nft flush chain foo input
```

##### 示例配置：筛选独立计算机的流量

您可以创建一个包含两个基链的表来定义规则，以过滤进出计算机的流量，并总结 IPv4 连接：

```
% nft add table ip filter
#添加一个顾过滤器（表）
% nft 'add chain ip filter input { type filter hook input priority 0 ; }'
% nft 'add chain ip filter output { type filter hook output priority 0 ; }'
#添加两两条规则
```

###  简单的规则管理

#### 添加规则

要添加新规则，您必须指定要使用的相应表和链，例如。

```
% nft add rule filter output ip daddr 8.8.8.8 counter
#其中 filter 是表，输出是链。上面的示例添加了一个规则，以匹配目标为 8.8.8.8 的输出链看到的所有数据包，如果匹配它会更新规则计数器。请注意，计数器在 nftables 中是可选的。
```

#### 列出规则

```
% nft list table filter
table ip filter {
        chain input {
                 type filter hook input priority 0;
        }

        chain output {
                 type filter hook output priority 0;
                 ip daddr 8.8.8.8 counter packets 0 bytes 0
                 tcp dport ssh counter packets 0 bytes 0
        }
}
```

规则解释：

>它定义了一个名为"filter"的表，该表中包含两个链："input"和"output"。
>
>- `nft list table filter`：这是一个命令，用于列出名为"filter"的表的所有内容。
>- `table ip filter`：这定义了一个名为"filter"的表，该表用于过滤IP数据包。
>- `chain input`：这定义了一个名为"input"的链，该链用于处理进入系统的数据包。它的类型是"filter"，挂钩点是"input"，优先级是0。
>- `chain output`：这定义了一个名为"output"的链，该链用于处理从系统发出的数据包。它的类型是"filter"，挂钩点是"output"，优先级是0。
>
>在"output"链中，有两条规则：
>
>1. `ip daddr 8.8.8.8 counter packets 0 bytes 0`：这条规则匹配所有目标地址为8.8.8.8的IP数据包，并对它们进行计数。当前的计数器值为0。
>2. `tcp dport ssh counter packets 0 bytes 0`：这条规则匹配所有目标端口为SSH的TCP数据包，并对它们进行计数。当前的计数器值为0。

按链列出规则 列出output规则

```
% nft list chain filter ouput
table ip filter {
        chain output {
                 type filter hook output priority 0;
                 ip daddr 8.8.8.8 counter packets 0 bytes 0
                 tcp dport ssh counter packets 0 bytes 0
        }
}
```

在列出规则时可以使用大量输出文本修饰符，例如，将 IP 地址转换为 DNS 名称、TCP 协议等。

#### 规则测试

让我们用一个简单的 ping 到 8.8.8.8 来测试这条规则：

```
% ping -c 1 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_req=1 ttl=64 time=1.31 ms
```

前置准备

- 创建表

  - ```bash
    # 创建名为 filter 的表
    nft add table ip filter
    
    # 在 filter 表中创建名为 input 的链
    nft add chain ip filter input { type filter hook input priority 0 \; }
    
    # 在 filter 表中创建名为 output 的链
    nft add chain ip filter output { type filter hook output priority 0 \; }
    ```

- 创建规则

  - ```bash
    # 在 output 链中添加规则，对所有目标地址为 8.8.8.8 的 IP 数据包进行计数
    nft add rule ip filter output ip daddr 8.8.8.8 counter
    
    # 在 output 链中添加规则，对所有目标端口为 SSH 的 TCP 数据包进行计数
    nft add rule ip filter output tcp dport ssh counter
    ```

然后ping 8.8.8.8

查看结果

```bash
root@s4670 ~/nftables # nft -nn list table filter
table ip filter {
        chain input {
                type filter hook input priority 0; policy accept;
        }

        chain output {
                type filter hook output priority 0; policy accept;
                ip daddr 8.8.8.8 counter packets 25 bytes 2100
                tcp dport 22 counter packets 0 bytes 0
        }
}
```

>`nft -nn list table filter`命令的作用是列出名为"filter"的表的所有内容，并且所有的地址、端口、协议和服务都以数字形式显示，而不是尝试解析为名称。

#### 在给定位置添加规则

如果要在给定位置添加规则，则必须使用句柄作为参考：

```
% nft -n -a list table filter
table filter {
        chain output {
                 type filter hook output priority 0;
                 ip protocol tcp counter packets 82 bytes 9680 # handle 8
                 ip saddr 127.0.0.1 ip daddr 127.0.0.6 drop # handle 7
        }
}
# 已知句柄7和句柄8
```

如果要在处理程序编号为 8 的规则之前插入规则，则必须键入：

```
% nft insert rule filter output position 8 ip daddr 127.0.0.8 drop
```

#### 删除规则

您必须通过 -a 选项获取句柄才能删除规则。**句柄由内核自动分配，并唯一标识规则。**

> 我还在上面纠结句柄是哪来的...emmm

```
% nft -a list table filter
table ip filter {
        chain input {
                 type filter hook input priority 0;
        }

        chain output {
                 type filter hook output priority 0;
                 ip daddr 192.168.1.1 counter packets 1 bytes 84 # handle 5
        }
}
# 获取到了句柄5
```

您可以使用以下命令删除句柄为 5 的规则：

```
% nft delete rule filter output handle 5
```

总结：nft delete rule + 表名 方向 + 句柄

注意：有计划通过以下命令来支持规则删除：

```
% nft delete rule filter output ip saddr 192.168.1.1 counter
```

整个命令的意思是：在`filter`表的`output`链中，删除所有源地址为`192.168.1.1`并且有计数器的规则。

#### 删除链中的所有规则

您可以使用以下命令删除链中的所有规则：

```
% nft flush chain filter output
# 单独删除output的相关规则
```

> 就是用flush清空规则（链）

您还可以使用以下命令删除表中的所有规则：

```
% nft flush table filter
# 删除filter的所有规则
```

#### 预置新规则

要通过 insert 命令在新规则前面附加，请执行以下操作：

```
% nft insert rule filter output ip daddr 192.168.1.1 counter
```

解释如下：就是插入一条新规则到顶部

>- `nft`：这是nftables的命令行工具，用于管理和配置nftables。
>- `insert rule`：这是一个命令，用于在已存在的链中插入一个新的规则。这个新的规则会被插入到链的开始位置。
>- `filter output`：这指定了你想要插入规则的表和链。在这个例子中，表是`filter`，链是`output`。
>- `ip daddr 192.168.1.1 counter`：这是规则的匹配条件和动作。这个规则会匹配所有目标地址（`daddr`）为`192.168.1.1`的IP包，并对匹配的包进行计数。

#### 替换规则

您可以通过 replace 命令通过指示规则句柄来替换任何规则，您必须首先使用选项 -a 列出规则集来找到该句柄：

```
# nft -a list ruleset
table ip filter {
        chain input {
                type filter hook input priority 0; policy accept;
                ip protocol tcp counter packets 0 bytes 0 # handle 2
        }
}
# 查找句柄
```

若要将规则替换为句柄 2，请指定其句柄编号和要替换它的新规则：

```
nft replace rule filter input handle 2 counter
```

> filter input    fileter output总是同时出现 成对出现

结果查看：列出上述替换后的规则集：

```
# nft list ruleset
table ip filter {
        chain input {
                type filter hook input priority 0; policy accept;
                counter packets 0 bytes 0 
        }
}
```

>`counter packets 0 bytes 0`：这是一个规则，它会对所有匹配的包进行计数。在这个例子中，因为没有定义匹配条件，所以这个规则会匹配所有的包。`packets 0 bytes 0`表示到目前为止，这个规则匹配到的包的数量和总字节数都是0。

