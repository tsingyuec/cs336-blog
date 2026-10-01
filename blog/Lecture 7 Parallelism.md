# Lecture 7: Parallelism

## 本讲要解决的核心问题（SCQA）

**背景（Situation）**：上一讲我们把视角缩小到**单块 GPU 内部**，学会了用 kernel、tiling、shared memory 等技巧让一块卡跑得更快。这节课要把镜头拉远：你手里不再只有一张卡，而可能是 4 张、8 张甚至 1000 张 GPU，它们彼此用各种线缆连在一起。([【跳转到 00:04】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=4))

**冲突（Complication）**：多卡并不自动等于更快。麻烦有两类：一来，**模型根本装不下**——参数、激活值、梯度和优化器状态会超出单卡显存（比如一张 B200 的显存也就一百多 GB 的量级，要训练一个 1T＝一万亿参数的模型，单卡完全不可能）；二来，即便装得下，为了**更快**而把任务拆到多张卡上，就必须付出**通信**的代价——数据要从一块 GPU 搬到另一块 GPU，而搬运远比计算慢。([【跳转到 01:43】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=103))

**疑问（Question）**：我们怎样把计算合理地"编排"到多张 GPU 上——用什么样的通信原语、在什么样的硬件层级上传输、又如何把模型或数据切开，才能让通信不成为瓶颈？

**回答（Answer，结论先行）**：本讲给出三样东西：① **集合通信（collective operations）**——分布式编程的基本词汇；② **GPU 之间的物理连接**——从 NVLink/NVSwitch 到 InfiniBand/Ethernet 的带宽阶梯，它决定了什么能放在哪里；③ 用 `torch.distributed` 从零实现的**三种并行策略**——数据并行（DDP）、张量并行、流水线并行。贯穿全部内容的心法只有一句：**计算单元离数据越远越慢，一切优化都是为了让"搬数据"不要成为瓶颈**——单 GPU 时代数据远在 HBM 里，多 GPU 时代数据可能在另一块卡上，但原则不变。用很多 GPU 很容易，高效用好它们却很难。([【跳转到 01:00】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=60)、[【跳转到 01:25】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=85))

上一讲的 fusion/tiling 是"减少对内存的访问"；这一讲的主题与之对偶——**通过在 GPU 之间做适当的复制与分片，减少通信量**。([【跳转到 02:23】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=143))

> 课堂提醒：这一讲的示例代码默认使用多进程（multiprocessing），但讲者为了逐行演示，幻灯片上展示的是单进程（single process）模式下的输出。([【跳转到 03:23】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=203))

---

## 一、集合通信：分布式训练的"共同语言"

### 1.1 rank、world size 与八个操作

**集合通信（collective operations）** 是一类分布式编程的基础原语，历史可以追溯到 **1980 年代**——它们不是为大语言模型发明的，但今天依然在用。所谓"集合"（collective），指你指定的是**跨多个设备的通用通信模式或模板**，而不是自己去管理点对点（point-to-point）通信——这样简单得多，系统也能帮你做更多事情，是一种非常成熟可靠的并行编程接口。([【跳转到 04:42】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=282))

约定两个术语（讲者自己也觉得叫法有点怪，但这是并行编程的标准说法）：

- **rank**：一个特定的设备（例如 0、1、2、3），在本课里 rank 就是 GPU（也可以是 TPU 等其他设备）；
- **world size**：设备总数（例子里是 4）。

([【跳转到 05:23】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=323))

![集合通信的设定：四个 rank（world size = 4）。操作分为三档——broadcast/scatter/gather/reduce 是入门热身，all-gather/reduce-scatter/all-reduce 是训练主力，all-to-all 服务于 MoE。](assets/p07/00323.jpg)

本讲覆盖八个操作。前四个是热身，后面的才是训练中反复出现的"主力"：

| 操作 | 含义 | 类比 |
| --- | --- | --- |
| broadcast | 把某个 rank（如 rank 0）上的张量复制给所有 rank | 一个人念，所有人抄 |
| scatter | 把一个大张量切分后分散到各 rank | 把一叠牌发给每个人 |
| gather | scatter 的逆操作，把各 rank 的片段拼到某个 rank 上 | 把大家的牌收回一人手里 |
| reduce | 在各 rank 的部分数据上做规约（如求和），结果放到某个 rank | 收齐后求和 |
| all-gather | 对**所有** rank 都做 gather，人人拿到完整数据 | 每个人都抄一份全集 |
| reduce-scatter | 对每个分量先规约，再把结果分散到不同 rank | 分工求和，各存一块 |
| all-reduce | ＝ reduce-scatter + all-gather，把规约结果复制到所有 rank | 人人都有完整的和 |
| all-to-all | 每个 rank 指定向哪个 rank 发送哪些元素 | 一次全员互换通信 |

几个热身操作的补充细节：

- **broadcast** 一般不出现在训练的核心路径里，多用于初始化阶段——比如加载一个初始检查点再广播给所有 rank，大概只做一次。([【跳转到 06:53】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=413))
- **scatter** 本身不直接使用，但它是理解 reduce-scatter 的基石；**gather** 同理，是理解 all-gather 的基石。([【跳转到 07:23】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=443))
- **reduce** 对函数式编程的人来说并不陌生：执行某种满足**结合律和交换律**的运算（sum、max 等），结果放到指定 rank。有趣的是，gather 也可以看作一种 reduce——它的"规约"就是**拼接（concatenation）**。([【跳转到 08:23】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=503))
- 课堂问答：这里的 broadcast 和 NumPy 的 broadcasting 是不是一回事？讲者的回答是**概念上同源**（把一个东西分发给多个），但因为这是集体通信，实现上有所不同。还有同学问 gather/reduce 的目标 rank 是不是固定死——不是，调用时**指定 rank 即可**，只是必须在调用时确定。([【跳转到 09:10】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=550))

> 记忆口诀：reduce 就是"规约"（满足结合律/交换律的运算）；scatter 是 gather 的逆操作——scatter **分散**、gather **汇聚**；"all" 表示目标是**所有**设备。所以 all-reduce＝规约后人人有份，all-gather＝汇总后人人有份。([【跳转到 15:04】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=904))

### 1.2 all-reduce = reduce-scatter + all-gather：最重要的一条等式

**all-gather**：每个 rank 只有一片参数或数据，all-gather 之后每个 rank 都拿到**完整**的数据。这正是后面"分片存储、用时收集"模式的基石——训练中我们会反复看到"先 gather 干点事，再 scatter，再 gather、再 scatter"的节奏。([【跳转到 09:34】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=574))

![all-gather：四个 rank 各持有标量 0、1、2、3，操作后每个 rank 都拿到 [0,1,2,3]，即"gather 到所有 rank"。](assets/p07/00604.jpg)

**reduce-scatter**：它 = 在张量的**每个分量**上分别做 reduce，然后把结果散给各 rank。例子里四个 rank 各持有一个错位的向量，逐分量求和得到 [6,10,14,18]，四个分量分别留在四个 rank 上。反向传播后要把不同数据分片算出的梯度加总、再重新分配存储，用的正是它。([【跳转到 10:34】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=634)、[【跳转到 11:04】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=664))

**all-reduce**：对一堆张量做规约（如求和），再把结果复制给所有 rank。从"最容易理解"的角度看，它就是"先 reduce-scatter、再 all-gather"两步。注意这个等价关系，下一讲 ZeRO/FSDP 的所有巧妙操作都建立在这条等式上：**因为 all-reduce 可以拆成两步，我们才可能"介入"并对中间状态做分片**。基础版的数据并行用单体 all-reduce 就够了；要进阶到 ZeRO/FSDP，就得把它拆开掌控。([【跳转到 11:34】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=694)、[【跳转到 12:34】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=754))

![reduce-scatter 与 all-reduce：输入是四个 rank 上错位的向量，reduce-scatter 把每个分量各自求和后分散存放；而 all-reduce 等于 reduce-scatter + all-gather，把 [6,10,14,18] 复制到所有 rank。](assets/p07/00724.jpg)

### 1.3 all-to-all 与 MoE 的动态路由

**all-to-all** 是最通用的：每个 rank 既可以持有数据的一个切片，也可以持有一组专家（expert）。MoE 的动态路由最终就变成一次 all-to-all——查自己的数据决定把哪些激活值发给哪些专家。若负载均衡（每个 rank 发给其他 rank 的字节数相同），它本质上就是一次**矩阵转置**；实操中最好让划分尽量均衡——这正是 MoE 里"负载均衡损失"存在的原因。当然 all-to-all 也能处理负载不均的情况，可以向任意 rank 发送任意数量的字节，只是均衡时效率最好。([【跳转到 13:04】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=784)、[【跳转到 14:34】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=874))

![all-to-all：四个 rank 各自把 4 个元素按目的 rank 发出，收完以后每个 rank 得到一组转置后的数据；MoE 的 token 路由就是这种模式。](assets/p07/00844.jpg)

---

## 二、硬件互连：带宽阶梯决定并行边界

### 2.1 从 PCIe/Ethernet 到 NVLink/NVSwitch

要理解并行为什么有代价，先要看 GPU 之间的"路"修得怎么样。把它放进一个广义的存储/通信层次结构里就很清晰：最局部的是单节点单 GPU 内部的寄存器、L1/L2 cache、shared memory，最快；往外是 HBM——上一讲我们还在抱怨它慢，这一讲却要把它当作"高速存储"；再往外是单节点多 GPU，通过 NVLink 连到 NVSwitch；最外层是多节点，通过 InfiniBand 或 Ethernet 互联。([【跳转到 01:30】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=90))

最朴素的配置是：服务器里有 CPU、PCIe 总线（过去连鼠标键盘都走它）、若干 GPU 和内存，同一节点上的 GPU 走 PCIe，机器之间用 Ethernet 连接——讲者吐槽：这就像你买了游戏显卡、和朋友用网线连起来说"我要训练大模型"，那你就只能这么干。([【跳转到 17:06】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=1026))

但真正认真做大模型训练时，配置更接近下面这样（[【跳转到 18:15】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=1095)）：

- 每个节点通常配 **8 块 GPU**，通过 **NVLink** 连接到 **NVSwitch**；从编程视角看，你可以把任意一块 GPU 当作"直接连到"其他任何一块——实际路由交给硬件。以 NVLink 5.0 为例，总带宽达到 **1.8 TB/s**，而 B200 的 HBM 带宽是 **8 TB/s**——NVLink 大约只有 HBM 的四分之一（但比跨网络仍快得多，当然还是比不上 shared memory 或 L1/L2 cache）。
- 集群规模变大后，节点被放进 **pod**，用 **InfiniBand** 互联（经 PCIe → HCA/网卡 → InfiniBand 线缆），带宽约 **0.05 TB/s** 量级，比 NVLink 低得多；GPU 不再直接连 GPU，中间要过好几道环节。
- 再往外，pod 之间只能用 **Ethernet**，数据还得经过 CPU，更慢。这就像内存层次结构：节点越多，越不可能让一个 NVSwitch 去伺候大约 10 万块 GPU。

![典型硬件拓扑：每个节点 8 块 GPU 经 NVLink 连到 NVSwitch，NVSwitch 再经 InfiniBand/Ethernet 引出；下方给出每层带宽与"绕过 CPU"的说明。](assets/p07/01095.jpg)

### 2.2 RDMA 与 RoCE：绕过 CPU 是关键

一个重要的硬件细节是**绕过 CPU**。传统 Ethernet 下，GPU 必须先把数据拷给 CPU（内核套接字缓冲区——这里的 kernel 指操作系统内核，不是 GPU kernel），由内核组装网络数据包、再复制到网卡发出，延迟很大。**RDMA（Remote Direct Memory Access，远程直接内存访问）** 让一块 GPU 直接读写另一块 GPU 的内存，整个过程不需要 CPU 参与。NVLink/NVSwitch 与 InfiniBand 都天然支持 RDMA；标准 Ethernet 不支持，但有 **RoCE（RDMA over Converged Ethernet，基于融合以太网的 RDMA）** 来补上——InfiniBand 通常价格不菲，RoCE 让较便宜的以太网也能拿到相当不错的性能；Meta 发过论文研究这个方向，Llama 的训练可能用了融合以太网、也可能没用到。([【跳转到 20:21】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=1221)、[【跳转到 22:03】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=1323))

要区分两组概念：**InfiniBand、NVSwitch、NVLink 说的是硬件**（有哪些线缆和交换机），**RDMA 说的是操作层面**（通信时到底发生了什么）。实现 RDMA 有多种方式：NVLink/NVSwitch 是一种，InfiniBand 是一种，RoCE 又是一种。([【跳转到 24:51】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=1491))

### 2.3 NVL72：把"极快域"从 8 卡扩到 72 卡

NVIDIA 一直在突破单域规模：面向 B200/B300 的 **NVL72** 用 9 个托盘、每托盘 8 块 GPU，把 **72 块 GPU** 连进同一个 NVLink 域。物理上是这样的：每个托盘装两块 Grace CPU，每块 CPU 连 4 块 GPU（所以每托盘 8 块），托盘堆叠起来、全部连到 NVSwitch。普通用户只有 8 卡在"极快域"内，而 NVLink 域之外速度会骤降。([【跳转到 21:29】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=1289)、[【跳转到 24:28】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=1468))

![NVL72 与 RoCE：9 个托盘、每托盘 8 块 GPU 共 72 卡同处一个 NVLink 域；以及 RoCE 让以太网也能绕过 CPU。](assets/p07/01316.jpg)

课堂问答的延伸：**"有 9 块 GPU 怎么办？"**——取决于拓扑。若第 9 块落在另一个节点、且两节点之间没有 NVLink/NVSwitch，配置就会非常糟糕：那个节点算力少、通信成本还极高；但如果所有设备都挂在 NVSwitch 上，情况就合理多了。([【跳转到 26:21】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=1581)
一句话总结这张"带宽地图"：**shared memory < HBM < NVLink/NVSwitch（单节点 8 卡）< InfiniBand（pod 内）< Ethernet（跨 pod 或跨数据中心）**。后面选择并行策略时，本质上就是在问："这种通信需要多快的链路？"

---

## 三、从概念到代码：NCCL 与 torch.distributed

### 3.1 NCCL：把集合操作"翻译"成 GPU 之间的真实数据包

最底层是 **NCCL（NVIDIA Collective Communications Library，读作"nickel"）**：它把 all-reduce、reduce、broadcast 等集合操作翻译成 GPU 之间实际传输的底层数据包。当你调用一次 all-reduce，NCCL 会**分析硬件拓扑**（有多少节点、交换机，走 NVLink 还是 PCIe）、**决定通信路径**（环形还是树形）并**启动 GPU kernel** 来收发数据——别忘了 GPU 上跑的一切归根结底都是 kernel，通信也不例外（还有专门负责与其他 GPU 通信的通信 kernel）。([【跳转到 23:17】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=1397))

![NCCL：它把集合操作翻译成底层数据包，检测硬件拓扑并选择路径，最终以 GPU kernel 的形式启动收发。](assets/p07/01397.jpg)

课堂问答：NCCL 是专门针对多节点集群优化的吗？讲者的回答是：具体细节不了解，但 NVIDIA 基本一直在为其整个软件栈做大型模型训练/推理的优化——他们最大的客户就是那些大公司和语言模型提供商，如果没针对这类负载优化过，那才让人意外。([【跳转到 25:51】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=1551))

### 3.2 spawn 与 setup：进程模型与协调通道

在 NCCL 之上，PyTorch 提供了 `torch.distributed`，后端可选 **NCCL（GPU）** 或 **gloo（CPU）**——并行编程在 GPU 出现之前就存在了，所以 CPU 上也能跑集合操作。本讲因为要在没有多 GPU 的环境里演示，用的是 gloo 后端。这个库还支持更高级的模型和算法，但本课为了"从零构建"不用那些。([【跳转到 27:48】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=1668))

- `spawn(fn, world_size=4)` 把函数 `fn` 复制运行 4 次，每个进程占据一个 rank（0 到 world_size−1），彼此**异步**、完成顺序任意交错——打印语句出现的顺序就是由硬件决定的任意顺序。([【跳转到 28:25】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=1705))
- `setup` 配置 `MASTER_ADDR/MASTER_PORT`。注意它**不是** GPU 之间的数据通路，只用于通用的元数据和协调；真正的数据必须走 NCCL，否则会非常慢。([【跳转到 29:17】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=1757))

### 3.3 barrier 与同步/异步

- `dist.barrier()` 是**同步屏障**：一旦遇到它，进程就会等待所有其他进程都走到这一步才继续。因为进程之间是异步的、谁先跑完没有保证，所以需要它来确保"某些代码先于其他代码执行"；代价是可能产生不必要的等待。([【跳转到 29:47】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=1787))
- **同步 vs 异步**：`dist.all_reduce(tensor, op=ReduceOp.SUM, async_op=False)` 是同步且**原地写回**——调用后张量内容立刻就是结果；`async_op=True` 会立即返回，之后你可以去做别的事情，最后用 `wait()` 确认完成。使用异步的典型场景是**让计算与通信重叠**：发完通信请求后去处理独立的数据或计算（比如加载下一步需要的数据），等真正需要结果时再同步。([【跳转到 33:13】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=1993)、[【跳转到 33:50】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=2030))

### 3.4 现场演示：all-reduce、reduce-scatter 与 all-gather

讲者现场演示了 all-reduce：四个 rank 分别持有 `[0,1,2,3]`、`[1,2,3,4]`、`[2,3,4,5]`、`[3,4,5,6]`，all-reduce（求和）之后每个 rank 都得到 `[6,10,14,18]`——这正是"每一列的和被复制到所有 rank"。([【跳转到 30:51】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=1851))

接着演示 reduce-scatter：这次**不做原地写入**，而是分别传入输入和输出两个张量——输入基本没被改动，输出里每个分量的规约结果写进对应的 rank。最后把 reduce-scatter 的输出作为输入做 all-gather，得到完整数据复制到所有 rank。这样就用代码实际验证了那条最重要的等式：**all-reduce = reduce-scatter + all-gather**。收尾时讲者还调用了清理环境的函数（`destroy_process_group` 一类的操作）——"清理环境是个好习惯"。([【跳转到 32:02】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=1922)、[【跳转到 34:56】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=2096))

---

## 四、量出通信有多快：all-reduce 的有效带宽

### 4.1 两个陷阱：warmup 与"两层异步"

光知道"能通信"不够，还要**量出通信有多快**。做基准测试时有两个陷阱要避开：

- 先 **warmup**（预热），因为第一次调用往往包含初始化开销；
- 用 `torch.cuda.synchronize()` 等待 **CUDA kernel** 真正结束，再用 `dist.barrier()` 等待**所有进程**都到达同一点。为什么要两个都要？因为系统里存在两层异步：每个进程内的 CUDA 操作默认异步（Python 执行下一行时 kernel 可能还没跑完），进程之间也彼此异步。只加 barrier 并不够——如果 kernel 还在运行，各进程只是各自简单地越过了屏障，并没有真正同步。([【跳转到 35:56】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=2156)、[【跳转到 40:36】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=2436))

### 4.2 公式：有效带宽与世界大小、拓扑无关

然后把时钟套在操作外面，算出**发了多少字节**、**花了多少时间**。示例对一亿个元素做 all-reduce，耗时约 1.6 毫秒——这算快还是慢？要算有效带宽：

```text
sent_bytes   = size_bytes × 2 × (world_size − 1)   # 2 = 发送+规约；W−1 = 规约迭代步数
total_time   = world_size × duration               # 对所有 rank 加权
effective_BW = sent_bytes / total_time
```

因子的来历：W 个 rank 两两合并求和需要 W−1 步迭代；每步既要**发送**又要**规约**，所以乘 2；时间上对 world_size 个 rank 加权取总量。([【跳转到 37:28】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=2248))

随着 `W` 增大，`(W−1)/W → 1`，于是 **`effBW ≈ 2S / T`——有效带宽与世界大小无关，也与拓扑（环形还是树形由 NCCL 决定）无关**。示例中测出约 **400 GB/s**，而且这个数字不随 GPU 数量增加而变差。这正是 all-reduce 被广泛使用的原因。([【跳转到 38:11】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=2291))

![有效带宽的测量：代码里 `sent_bytes = size_bytes * 2 * (world_size - 1)`，再除以总持续时间；注释指出有效带宽与世界大小和拓扑无关。](assets/p07/02248.jpg)

### 4.3 reduce-scatter 对照：少做一半事，带宽却相同

reduce-scatter 的测法类似，但字节数**不乘 2**（只有发送、没有单独的规约步骤），量级同样是 400 GB/s 左右（有时有些随机性，大致 400 多）。用一个统一的视角看：**all-reduce 做了 reduce-scatter 和 all-gather 两份工作，所以移动两倍数据，但它也花了两倍时间——两者相互抵消，于是带宽相同**。([【跳转到 39:10】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=2350)、[【跳转到 39:36】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=2376))

---

## 五、数据并行（DDP）：只比标准训练多一步

### 5.1 切数据、不切模型：反向传播后 all-reduce 梯度

讲者用最基础的 MLP 来演示三种并行——别忘了 **MLP 才是 Transformer 里真正的计算瓶颈**，所以这个例子很具代表性。([【跳转到 41:31】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=2491))

**数据并行**的做法是把**数据**拆开：batch 维切成 world_size 份，每个 rank 拿 `batch_size / world_size` 行数据（比如 batch 128、4 个 rank，每个 rank 处理 32 行），但**每个 rank 都维护完整的一份参数**。实际训练中每个 rank 通常会加载自己的数据以避免瓶颈，演示里只是为了方便。前向、反向都和普通训练一模一样，区别只在反向传播之后：对所有参数的梯度做一次 all-reduce 并取平均，于是各 rank 的梯度一致，参数更新后也保持一致。([【跳转到 42:15】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=2535)、[【跳转到 44:15】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=2655))

![数据并行：每个 rank 拿到数据的一部分（图中 Data 被沿 batch 维切开），但都持有完整的层级参数。](assets/p07/02535.jpg)

```python
loss.backward()
# 标准训练与 DDP 的唯一区别，就是下面这三行：
for param in params:
    dist.all_reduce(tensor=param.grad, op=dist.ReduceOp.AVG, async_op=False)
optimizer.step()
```

### 5.2 优雅在模块化，约束在 batch size

它优雅在**模块化**：DDP 不关心前向传播长什么样，它只负责"同步参数"这件事——换成 Transformer 也一样，前向照旧、DDP 只对参数做平均（有同学问"Transformer 的 DDP 长什么样"，答案就是：基本一样）。两个约束：batch size **至少不小于 world size**（否则有的卡分不到数据），且最好是其整数倍（否则要补零填充——办法是有的，只是整数倍大家都省事）。([【跳转到 45:29】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=2729)、[【跳转到 46:01】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=2761))

![DDP 的关键一行：反向传播后对每个参数的梯度做 all-reduce（取平均），其余代码与单卡训练相同。](assets/p07/02685.jpg)

### 5.3 致命局限：内存零节省

数据并行有一个致命局限：**内存毫无节省**——每个 GPU 都存着参数的完整副本，显存占用随卡数线性增长。各个 rank 的损失值不一样、梯度一开始也不一样，但经过规约后就都一致了。如果参数太大根本装不下，就得靠下一讲的 ZeRO/FSDP 把状态分片——all-reduce 像一个"简单的单体操作"，它要求把模型的所有参数都保留在内存里。([【跳转到 46:26】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=2786)、[【跳转到 46:56】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=2816))

---

## 六、张量并行：沿"宽度"切，用通信量换显存

### 6.1 列式切分与 all-gather

如果说数据并行切的是**数据**，**张量并行（tensor parallelism）** 切的就是**每一层的参数矩阵**：数据不切，每个 rank 只持有每层的一部分维度，因此每个 rank 都能算出**部分激活值**。代价从一开始就注定：通常需要传输的数据量会大得多。([【跳转到 47:07】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=2827)、[【跳转到 47:11】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=2831))

具体做法（本讲演示**列式切分**；按行切也行，这里先不展开）：把每层的参数矩阵沿**列**方向切成 world_size 份，rank i 拿第 i 份。前向传播时，各 rank 用自己那份参数对完整输入 `X` 做矩阵乘法，得到形状为 `batch × 局部维度` 的部分激活；非线性激活函数是逐元素的，局部算完全没问题；然后通过 **all-gather** 把所有 rank 的激活收集齐全，再拼接成完整维度的 `X`，进入下一层——每一层都要这么做一遍。([【跳转到 48:06】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=2886)、[【跳转到 49:36】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=2976))

![张量并行：每个 rank 只持有参数矩阵的一部分（列式切分），前向得到部分激活后用 all-gather 拼成完整激活。](assets/p07/03006.jpg)

这里有一个漂亮的对偶关系：**前向做 all-gather，则反向就做 reduce-scatter**（梯度要按各自的切分散回去）；反之亦然。all-gather 和 reduce-scatter 就具有这种"双重性"。([【跳转到 51:06】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=3066))

### 6.2 手动管理：从零构建的代价与意义

要特别注意——**这一讲里整个过程是手动管理的**：你没有调用通常的 `backward` 自动微分来帮你做分布式的部分，得自己调用 reduce-scatter、自己组织前向。有同学问"这些是 autograd 自动完成的吗"——不是，直接调 `backward` 不会做这些，因为其中没有并行；但 PyTorch 其实自带这些功能，很多事情会自动为你完成。这是刻意设计：本课的目标就是"从零构建"，让你看清每一步；实际使用中（比如 PyTorch 自带的并行 API）很多工作会自动完成。([【跳转到 51:36】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=3096))

还有一点值得对比：数据并行非常优雅，因为它是按数据切分的、模型被当作一个模块来对待；而张量并行**必须对模型本身动手术**——它利用了"矩阵乘法可以拆成一组小矩阵乘法、分别执行再汇总"这一事实。([【跳转到 50:06】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=3006))

### 6.3 通信代价：只适合 NVLink 域内

代价是**通信量大**：几乎每一层都要传输相当大的激活值，而且是 all-gather/reduce-scatter 级别的通信。因此张量并行通常**只放在节点内部**（NVLink 高带宽域），一般不超过 8 路；一旦跨出单机、走到速度慢得多的节点间网络，性能就会大幅下跌。它不会像流水线那样产生"气泡"、复杂度也较低，但非常吃带宽。([【跳转到 57:18】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=3438))

---

## 七、流水线并行：沿"深度"切，用 micro-batch 填满气泡

### 7.1 朴素切法的"气泡"问题

**流水线并行（pipeline parallelism）** 沿**深度**方向切：每个 rank 负责模型的一部分层（`local_num_layers`），但每层内部保留全部维度；rank 之间用**点对点**的 send/recv 传递激活值——rank i 从 rank i−1 接收张量、算完自己负责的层后发给 rank i+1。前向是从第 0 个 rank 一路往后传激活，反向则把部分梯度往前传回来。([【跳转到 52:36】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=3156))

最自然的问题是**流水线气泡（bubble）**：如果一次只处理一个 batch，那么同一时刻只有一块 GPU 在工作，其余都在空等——没有在计算，只是在等别的张量，利用率极差。([【跳转到 54:45】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=3285))

### 7.2 micro-batch 与通信/计算重叠

解决办法是引入 **micro-batch**：把 batch 拆成更小的批次（演示里每 4 个一组），rank 0 处理完一个 micro-batch 立刻传给 rank 1、并马上开始处理下一个，让数据像流水线一样在层级间持续流动，从而把空闲时间压下去。([【跳转到 54:45】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=3285))

**通信与计算的重叠**对流水线并行尤其关键：把接收/发送设计成异步（在函数名前加个 `i`，如 `isend/irecv`），就可以在计算的同时收发数据——"当你在这儿计算的时候，你可以同时接收或发送数据"。其实数据并行也一样：本讲的演示只做了一次前向、最后一次性 all-reduce；但如果你处理得当，反向传播中某个梯度一算完就可以开始发送，让通信"藏"在计算里，而不是等所有梯度算完再一次性 all-reduce。本讲这一部分没有展开，留给了下一讲。([【跳转到 55:33】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=3333))

### 7.3 适用场景：容忍慢网络

流水线并行的**通信量最小**：它只涉及层与层之间 `B×S×H`（batch × 序列长度 × 隐藏维度）的激活值传递，而不是 all-reduce 那样把整个参数矩阵翻来覆去。因此它能容忍慢得多的互联网络——一些**去中心化训练**工作会使用流水线并行，因为 GPU 节点实际上分布在世界各地；那种情况下你绝不会想用张量并行。它能否高效，取决于 batch size 和 micro-batch 的数量：micro-batch 越多，气泡占比越小。([【跳转到 55:03】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=3303)、[【跳转到 57:22】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=3442))

---

## 八、如何选择与组合：先用满数据并行，再按硬件层级往上叠

### 8.1 硬件决定策略

三种策略比较下来，选择很大程度上**取决于硬件**：

- **张量并行**吃带宽，放在 NVLink 域内（节点内）；
- **流水线并行**容忍慢网络，可以跨节点、跨 pod 使用；
- **数据并行**最省心，但受**临界批次大小（critical batch size）** 约束：batch 加到一定程度就会收益递减，再大就是浪费算力——此时改用张量并行更划算。([【跳转到 57:52】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=3472))

### 8.2 典型组合与没讲到的两种并行

典型的组合是"从内到外"：**节点内张量并行 → 叠加数据并行或 FSDP → 最后用流水线并行跨节点**。除此之外还有两种本讲没细讲的并行：**序列并行**（把整个序列拆分成小块，从而实现 attention 计算的并行化）和**专家并行**（并行化 MoE 的各个专家，正是 all-to-all 大显身手之处）；不同并行技术的各种组合也会出现，这些内容在作业中都会涉及。([【跳转到 56:22】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=3382)、[【跳转到 57:52】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=3472))

![三种基本并行与总结：数据并行切 batch、张量并行切宽度、流水线并行切深度；本讲末尾给出"节点内张量并行 → DP/FSDP → 跨节点流水线"的组合顺序。](assets/p07/03442.jpg)

### 8.3 另一条路线：JAX/TPU 的"编译器魔法"

还有一种更省心的路线（如 JAX/TPU）：你只**定义模型和分片策略**——"这块数据得放在这儿、这儿和这儿"——编译器就会自动推导出所需的通信操作并完成其余"魔法"。这确实很吸引人，但代价是少了从零构建的乐趣与掌控感；本课特意选用 PyTorch、而且用很基础的方式调用集合操作，就是为了让你看得更清楚。([【跳转到 58:22】](https://www.bilibili.com/video/BV11LEA6eEuj/?p=7&t=3502))

---

## 小结

1. **集合通信是分布式训练的语言**：broadcast/scatter/gather/reduce 是基础热身，all-gather/reduce-scatter/all-reduce 是主力，all-to-all 服务于 MoE 路由。最重要的一条等式是 **all-reduce = reduce-scatter + all-gather**——它让下一讲的 ZeRO/FSDP 成为可能。
2. **硬件层次决定通信代价**：shared memory < HBM < NVLink/NVSwitch（单节点 8 卡，NVLink 5.0 约 1.8 TB/s）< InfiniBand（pod 内，约 0.05 TB/s）< Ethernet（跨 pod）。RDMA 让 GPU 绕过 CPU 直接访存；RoCE 把这个能力带给了以太网；NVL72 把一个 NVLink 域扩展到 72 块 GPU。
3. **编程接口分两层**：NCCL 负责把集合操作翻译成真实数据包、探测拓扑、启动通信 kernel；`torch.distributed`（NCCL/gloo 后端）提供 `spawn/barrier/all_reduce/all_gather/reduce_scatter` 等接口，异步模式 `async_op=True` 是通信与计算重叠的基础。
4. **测带宽要小心两层异步**：先 warmup，再用 `torch.cuda.synchronize()` + `dist.barrier()` 对齐 CUDA kernel 和进程。all-reduce 的有效带宽 `≈ 2S/T`，**与世界大小和拓扑无关**，示例约 400 GB/s；all-reduce 比 reduce-scatter 多做一倍工作、也多花一倍时间，带宽相同。
5. **数据并行（DDP）** 只比标准训练多一步"反向后 all-reduce 梯度"，模块化、优雅，但每个 GPU 存完整副本，**内存零节省**。
6. **张量并行**沿宽度切、每层都要通信、需手动管理，前后向存在 all-gather/reduce-scatter 的**对偶**；适合 NVLink 域内（≤8 路）。
7. **流水线并行**沿深度切、通信量最小、能容忍慢网络，但要用 **micro-batch 压缩气泡**，并重视通信与计算的重叠。
8. **组合原则**：先尽量把数据并行拉满；放不下时在高速域内用张量并行切开；再用流水线并行/FSDP 跨节点；batch 太小就用梯度累积补。

下一讲 Tatsu 会深入 FSDP 与 ZeRO 等更高级的数据并行技术，以及真正的"4D 并行"。

---

## 关键术语速查

| 术语 | 一句话解释 |
| --- | --- |
| rank / world size | rank 是设备编号（本课中 rank 就是 GPU），world size 是设备总数 |
| 集合通信（collective） | 指定"跨多个设备的通用通信模式"的原语，而不是手动管理点对点通信 |
| broadcast / scatter / gather / reduce | 复制给所有 rank / 把大张量分散 / 把各片段汇聚 / 规约到某个 rank |
| all-gather | 每个 rank 收集齐全部片段，人人拿到完整数据 |
| reduce-scatter | 对每个分量先规约，再把结果分散到各 rank |
| all-reduce | reduce-scatter + all-gather，把规约结果复制到所有 rank |
| all-to-all | 每个 rank 按指定目的地发送元素；MoE 路由的通信模式，均衡时相当于矩阵转置 |
| NVLink / NVSwitch | 单节点内 GPU 的高速互联（NVLink 5.0 约 1.8 TB/s）与交换硬件 |
| InfiniBand / Ethernet | 节点间/pod 间网络，带宽依次降低；前者支持 RDMA |
| RDMA | 远程直接内存访问：GPU 直接读写另一块 GPU 的内存，不经过 CPU |
| RoCE | 基于融合以太网的 RDMA，让较便宜的以太网也能绕过 CPU |
| NCCL | NVIDIA 集合通信库：把集合操作翻译成底层数据包并启动 GPU kernel |
| gloo | PyTorch 的 CPU 集合通信后端 |
| DDP（数据并行） | 切 batch、每卡完整参数，反向传播后 all-reduce 梯度 |
| 张量并行 | 切每层参数矩阵（宽度），前向 all-gather 激活、反向 reduce-scatter |
| 流水线并行 | 按层切分（深度），点对点传激活，需 micro-batch 减少气泡 |
| micro-batch | 把一个大 batch 拆成的小批次，用于填充流水线 |
| 气泡（bubble） | 流水线中某些 GPU 无事可做的空闲时间 |
| 有效带宽 | 传输字节数 ÷ 总耗时；all-reduce 约为 `2S/T`，与 world size 无关 |
| 临界批次大小 | batch 增大到收益开始递减的临界点，超过它就浪费算力 |
