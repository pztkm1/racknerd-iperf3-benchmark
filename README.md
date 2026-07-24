# RackNerd 带宽测试完全指南：1Gbps 跑不满怎么办？iperf3 正确测速方法、洛杉矶/圣何塞机房实测对比与全套餐选购建议

上周帮朋友测他新买的 RackNerd VPS，他兴冲冲跑了个 speedtest-cli，结果下行只有 80Mbps，当场就急了——"不是说 1Gbps 吗？这是不是被坑了？"。RackNerd 带宽测试这事我自己也踩过同样的坑，所以这次把测速方法、机房差异、套餐怎么选一次说清楚，省得你再走弯路。

先把概念摆正：RackNerd 所有 VPS 套餐标配 1Gbps 端口，这个 1Gbps 指的是机房上行端口的物理带宽上限，不是你随便下个文件就能跑满的速度。能不能跑满，看你测什么、怎么测、测到哪。

## speedtest-cli 为什么测 RackNerd 带宽不准

很多人买完第一件事就是 `apt install speedtest-cli` 然后跑一下，看到几十兆就慌。其实这个工具天生就不适合给服务器测速。

它本来就是给家庭宽带设计的，测速服务器大多是居民区节点，端口普遍是 1Gbps 共享，你从一个 1Gbps 共享端口去测另一个 1Gbps 端口，瓶颈在对面不在你这边。RackNerd 自己在知识库里也提过这事儿——speedtest-cli 那个版本还有 bug，跑出来的数字不代表你机器的真实网络表现。

更别提单线程测试。一条 TCP 流跑 1Gbps，CPU 单核先顶到天花板，网络还没吃满呢。所以你看到的 80Mbps，大概率不是 RackNerd 给你限速了，是测法本身的天花板。

换个说法：拿一把只能量 30 厘米的尺子去量 1 米的桌子，然后说桌子只有 30 厘米——是尺子的问题，不是桌子的问题。

## 正确的 RackNerd 带宽测试方法：iperf3 多线程

想看到 RackNerd VPS 真实能跑多少，得用 iperf3，而且要开多线程。下面这套流程我自己跑过好几台，比 speedtest-cli 靠谱得多。

**第一步：准备两台服务器**

iperf3 是 C/S 架构，你要测的 RackNerd 机器当客户端，还需要另一台机器当服务端（监听端）。服务端最好也是 1Gbps 或更高端口，不然又卡在对面。两台都装上 iperf3：

bash
# Debian/Ubuntu
apt install iperf3

# CentOS/RHEL
yum install iperf3


**第二步：服务端开监听**

在当服务端的那台机器上执行：

bash
iperf3 -s


它会告诉你默认监听 5201 端口。如果开了防火墙，记得放行 5201，或者临时关掉 iptables 再测。

**第三步：客户端多线程压测**

回到你要测的 RackNerd VPS，执行：

bash
iperf3 -P 10 -c 服务端IP -t 30


`-P 10` 是关键，开 10 条并发流，把负载分摊到多个 CPU 核上，避免单核先瓶颈。`-t 30` 跑 30 秒，够看出稳态吞吐。结果里看 `SUM` 那一行的带宽，那个数字才接近你端口的真实能力。

**第四步：换方向测上传**

iperf3 默认测的是客户端→服务端（上行），加个 `-R` 反过来测下行：

bash
iperf3 -P 10 -c 服务端IP -t 30 -R


**第五步：跨机房对比**

想看不同机房之间的内网互通，把服务端换到 RackNerd 另一个机房的机器上再跑一遍。RackNerd 官方做过演示，洛杉矶到新泽西的 10Gbps 独服实测能稳稳跑到 9.46Gbps，多线程是必须的。

一句话总结：speedtest-cli 看个热闹，iperf3 -P 10 才是认真测。

## RackNerd 各机房对中国用户的实测表现

带宽跑得满是一回事，到你家快不快是另一回事。RackNerd 在全球有 20 个数据中心，对中国用户来说，西海岸那几个机房才是重点。

我整理了一份测试 IP，你可以先在自己电脑上 ping 一遍再决定买哪个机房：

| 机房 | 测试 IPv4 | 备注 |
|---|---|---|
| 洛杉矶 DC-02 | 204.13.154.3 | 优化线路，国内口碑最好 |
| 洛杉矶 DC-03 | 107.174.51.158 | 普通 KVM 常用 |
| 圣何塞 | 192.210.207.88 | 电信联通直连，延迟低 |
| 西雅图 | 192.3.253.2 | 西海岸备选 |
| 芝加哥 | 198.23.228.15 | 中部 |
| 达拉斯 | 198.23.249.100 | 中部，三网走 Cogent |
| 纽约 | 192.3.81.8 | 东海岸，延迟偏高 |
| 阿什本 | 192.3.254.158 | 东海岸 |
| 阿姆斯特丹 | 23.94.101.88 | 欧洲入口 |
| 伦敦 | 185.239.172.14 | 欧洲 |

实测下来几个比较稳的数字：洛杉矶 DC-02 到国内 ping 大概 170-200ms，电信联通移动三条线路上传基本都能过 300Mbps，下载过 400Mbps，晚高峰会掉但不算崩。圣何塞延迟更低一点，160-190ms 区间，电信联通走直连，移动也还行。纽约就别考虑了，250ms 往上，对延迟敏感的应用难受。

下载国内镜像站这种真实场景，白天能跑到 5-8MB/s，晚高峰掉到 2-4MB/s。讲真，对于这个价位段的年付 VPS，这个表现属于正常水平，别拿它去跟 CN2 GIA 优化线路的贵价机器比。

## 中国用户机房怎么选

简单粗暴的结论：首选洛杉矶 DC-02，次选圣何塞。

洛杉矶 DC-02 是 RackNerd 主推的优化线路机房，也是国内用户买得最多的。如果你买的是促销套餐，下单时选这个机房。圣何塞托管在 ColoCrossing 圣克拉拉数据中心，1Gbps 接入，对电信联通用户延迟更低，移动区别不大。

如果是欧洲业务或者面向欧洲用户，阿姆斯特丹和伦敦都行。纯跑流量、不在意延迟的（比如挂 PT、做中转），其实哪个机房都无所谓，挑便宜的促销套餐买就行。

## RackNerd 全套餐对比：KVM、Ryzen、Windows

RackNerd 的 VPS 三条产品线都标配 1Gbps 端口，差别在 CPU 和存储。我把官网全部套餐整理在下面，方便你直接对比选购。

### KVM VPS（Intel Xeon + SSD，主流款）

| 套餐 | CPU | SSD 存储 | 带宽 | IPv4 | 价格 | 购买 |
|---|---|---|---|---|---|---|
| 512MB | 1 vCore | 30GB RAID-10 | 500GB @ 1Gbps | 1 个 | $26.99/年 |  [选择此方案](https://my.racknerd.com/cart.php?a=add&pid=1&aff=11397) |
| 1GB | 2 vCore | 50GB RAID-10 | 1TB @ 1Gbps | 1 个 | $17.99/月 |  [选择此方案](https://my.racknerd.com/cart.php?a=add&pid=20&aff=11397) |
| 2GB | 3 vCore | 75GB RAID-10 | 2TB @ 1Gbps | 1 个 | $20.59/月 |  [选择此方案](https://my.racknerd.com/cart.php?a=add&pid=21&aff=11397) |
| 4GB | 4 vCore | 130GB RAID-10 | 3TB @ 1Gbps | 1 个 | $24.59/月 |  [选择此方案](https://my.racknerd.com/cart.php?a=add&pid=22&aff=11397) |
| 6GB | 5 vCore | 170GB RAID-10 | 4TB @ 1Gbps | 1 个 | $27.59/月 |  [选择此方案](https://my.racknerd.com/cart.php?a=add&pid=23&aff=11397) |
| 8GB | 6 vCore | 220GB RAID-10 | 5TB @ 1Gbps | 1 个 | $36.59/月 |  [选择此方案](https://my.racknerd.com/cart.php?a=add&pid=24&aff=11397) |
| 12GB | 7 vCore | 300GB RAID-10 | 6TB @ 1Gbps | 1 个 | $55.99/月 |  [选择此方案](https://my.racknerd.com/cart.php?a=add&pid=25&aff=11397) |

### AMD Ryzen VPS（NVMe 存储，磁盘性能更强）

| 套餐 | CPU | NVMe 存储 | 带宽 | IPv4 | 价格 | 购买 |
|---|---|---|---|---|---|---|
| 512MB | 1 vCore | 10GB NVMe | 500GB @ 1Gbps | 1 个 | $26.99/年 |  [选择此方案](https://my.racknerd.com/cart.php?a=add&pid=500&aff=11397) |
| 1GB | 1 vCore | 15GB NVMe | 1TB @ 1Gbps | 1 个 | $17.99/月 |  [选择此方案](https://my.racknerd.com/cart.php?a=add&pid=501&aff=11397) |
| 2GB | 2 vCores | 20GB NVMe | 2TB @ 1Gbps | 1 个 | $20.59/月 |  [选择此方案](https://my.racknerd.com/cart.php?a=add&pid=502&aff=11397) |
| 4GB | 2 vCores | 30GB NVMe | 3TB @ 1Gbps | 1 个 | $24.59/月 |  [选择此方案](https://my.racknerd.com/cart.php?a=add&pid=503&aff=11397) |
| 6GB | 3 vCores | 45GB NVMe | 4TB @ 1Gbps | 1 个 | $27.59/月 |  [选择此方案](https://my.racknerd.com/cart.php?a=add&pid=504&aff=11397) |
| 8GB | 3 vCores | 75GB NVMe | 5TB @ 1Gbps | 1 个 | $36.59/月 |  [选择此方案](https://my.racknerd.com/cart.php?a=add&pid=505&aff=11397) |
| 12GB | 4 vCores | 90GB NVMe | 6TB @ 1Gbps | 1 个 | $55.99/月 |  [选择此方案](https://my.racknerd.com/cart.php?a=add&pid=506&aff=11397) |

### Windows VPS（含 Windows Server 授权）

| 套餐 | CPU | NVMe 存储 | 带宽 | IPv4 | 价格 | 购买 |
|---|---|---|---|---|---|---|
| 2GB | 1 vCore | 35GB NVMe | 2TB @ 1Gbps | 1 个 | $27.59/月 |  [选择此方案](https://my.racknerd.com/cart.php?a=add&pid=293&aff=11397) |
| 4GB | 2 vCores | 60GB NVMe | 2TB @ 1Gbps | 1 个 | $30.59/月 |  [选择此方案](https://my.racknerd.com/cart.php?a=add&pid=294&aff=11397) |
| 6GB | 2 vCores | 85GB NVMe | 3TB @ 1Gbps | 1 个 | $35.59/月 |  [选择此方案](https://my.racknerd.com/cart.php?a=add&pid=295&aff=11397) |
| 8GB | 3 vCores | 110GB NVMe | 5TB @ 1Gbps | 1 个 | $44.59/月 |  [选择此方案](https://my.racknerd.com/cart.php?a=add&pid=296&aff=11397) |
| 12GB | 4 vCores | 160GB NVMe | 6TB @ 1Gbps | 1 个 | $64.59/月 |  [选择此方案](https://my.racknerd.com/cart.php?a=add&pid=297&aff=11397) |
| 16GB | 6 vCores | 200GB NVMe | 10TB @ 1Gbps | 1 个 | $89.59/月 |  [选择此方案](https://my.racknerd.com/cart.php?a=add&pid=298&aff=11397) |

除了上面这些常规套餐，RackNerd 的 [👉 查看当前所有促销套餐](https://bit.ly/RacKnerd) 页面常年挂着特价年付方案，黑五、五一、纪念日这些节点会放出 $10.6/年起的 KVM 套餐（1 核 1G 内存 2T 月流量），不需要任何优惠码，直接下单就是促销价。这种年付套餐是 RackNerd 性价比的精髓，预算紧的话盯促销页面比盯常规套餐划算。

## 套餐到底怎么选

不同用途选不同档，别一上来就顶配。

**纯跑流量、挂 PT、做中转**：盯促销页面，$10-15/年那种 1G 内存 2T 流量的 KVM 套餐够用，机房选洛杉矶 DC-02。算下来一天不到一毛钱，跑满流量额度就回本。

**建站、跑小服务**：2GB 内存起步的 Ryzen 套餐，NVMe 磁盘 I/O 能到 1GB/s 以上，比同价位的 SSD 方案快不少。$20.59/月 这个档位是甜点。

**Windows 应用、跑软件**：直接看 Windows VPS 那张表，2GB 起 $27.59/月，Windows Server 授权含在内不用另算。

**重负载、多站点**：8GB 以上的 Ryzen 或 KVM 套餐，6 vCore 起步，5TB 月流量一般够用，跑满了再升级也来得及——RackNerd 支持随时升配，重启一下就生效。

讲真，多数人买 RackNerd 就是冲便宜年付去的。我自己手上有两台，一台促销年付挂着 PT，一台 4GB Ryzen 跑小站，用了两年没出过什么大问题，客服工单回复也算及时。如果你还没用过这家，建议先从低价年付小额试水，确认体验满意再考虑上长周期的大套餐。👉 [前往 RackNerd 查看最新套餐与促销](https://bit.ly/RacKnerd)

## 关于退款这件事，先说在前面

RackNerd 官方政策明确不退款，没有 money-back guarantee 这回事。他们的说法是：会尽一切努力解决问题，直到你 100% 满意。

所以最理性的玩法就是——别一上来就买三年付的大套餐。先用月付或者几十块的年付套餐试一个月，机房、带宽、稳定性都摸一遍，满意了再续长周期。这点跟那些承诺 30 天退款的商家不一样，下单前想清楚。

## 常见问题

**RackNerd 带宽测试用 speedtest-cli 跑出来才几十兆，是不是被骗了？**

不是。speedtest-cli 的测速服务器普遍是 1Gbps 共享端口，且单线程跑会先卡在 CPU 单核，不能反映你机器端口的真实能力。用 iperf3 开 `-P 10` 多线程，再找一台同样是 1Gbps 端口的机器当服务端，才能看到接近端口上限的数字。

**1Gbps 端口能跑到多少？**

理论上 1Gbps 等于约 125MB/s。实际受 TCP 开销、单核瓶颈、对端端口限制影响，iperf3 多线程实测跑到 900Mbps 上下属于正常水平。想跑满得对端也是千兆且没被别人挤占。

**中国用户选哪个机房最快？**

洛杉矶 DC-02 和圣何塞。DC-02 是优化线路，ping 170-200ms，三网上传下载都比较稳；圣何塞电信联通直连，延迟更低一点。东海岸的纽约、阿什本延迟 250ms+，对延迟敏感就别选。

**买的套餐能升级吗？能换机房吗？**

升级随时可以，在控制面板里点一下，重启生效，停机大概一分钟。换机房不直接支持，但你可以重装系统时选别的机房位置（前提是那个机房有库存），相当于迁移。

**黑五促销套餐和常规套餐有什么区别？**

促销套餐是限时年付特价，配置通常是 1 核 1G 内存 2T 流量这种入门档，价格 $10-15/年，不用优惠码直接买就是促销价。常规套餐是月付，配置档位更全，但单价算下来比促销年付贵。能等促销就等促销，急用就买常规月付。

## 最后

如果你是冲着 RackNerd 带宽测试这个话题进来的，记住三件事：别用 speedtest-cli 当唯一依据，iperf3 多线程才是正经测法；中国用户机房认准洛杉矶 DC-02 或圣何塞；下单前先用上面那张测试 IP 表 ping 一遍，确认延迟你能接受。预算紧就盯促销年付，重负载上 Ryzen NVMe，Windows 应用走 Windows 专线。👉 [前往 RackNerd 获取最适合你的方案](https://bit.ly/RacKnerd)
