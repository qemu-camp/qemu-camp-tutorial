# 随笔：开头

这个文档几乎是我做这个项目期间"含人量"最高的部分了, 除了ascii-art由agent捉刀, 其他都是我自己写的, 请放心观看.

我做的题目是[建模AI加速卡](https://qemu.gevico.online/tutorial/2026/ch3/qemu-cxlemu/), 在 QEMU 仿真的 CXL Type-2 GPU 链路上运行 Kimi K2.6 IQ1_M 模型, 在满足正确性的前提下提升性能, 我一开始以为这个项目的重点是HetGPU, 毕竟手动做空英伟达生态这个事情听着很有意思~~我又没有英伟达的股票~~, 不过做到后面发现重点似乎是CXL

```text
                         模型与运行时allocation
                                  |
                        placement与生命周期owner
                                  |
                   +--------------+--------------+
                   |                             |
                   v                             v
              GPU HBM                       CXL内存池
          高带宽、容量有限                容量大、访问较慢
          保存执行热工作集                保存权威模型字节和冷数据
                   ^                             |
                   |                             |
             resident/prefetch             direct demand
                   |                             |
                   +--------------+--------------+
                                  |
                                  v
                              GPU kernel
```

传统 CPU offload 已经把模型总容量分在 HBM 和主机内存(SSD)之间，但它通常把主机 DRAM 或每个进程的文件映射当作剩余权重的后备，
消费时仍可能反复执行 HtoD、等待 copy 完成并占用临时 staging, 反正就是内存拷来拷去. 那如果能把 CXL 作为一个内存层级映射进去做一个稍微慢一点的cache池, 在改变了硬件布局的情况下就会有很多有趣的东西可能会出来

实际上我本地还是尝试过在sm86上跑通qwem的HetGPU的二进制翻译的, 想着把kimi也二进制翻译了走通全链路岂不是妙事一件, 但是那个东西没走通, 我印象中rd好像在cnb上走通了kimi的GPU的二进制翻译

这个博客大体上分这几个部分, 可以跳读感兴趣的部分, 但不一定写得完就是(

- 没什么信息量的
  - 第一部分是感悟, 主要是预告下第三四部分的决策动机
  - 第二部分是正确性验收的结果
- 多少有点信息量
  - 第三部分是项目的迭代过程的设计
     - 当前性能的下的技术路线和瓶颈分析
     - 怎么让agent handle所有的开发需求
       - 早知需求如此复杂一开始就不该用shell
  - 第四部分是在正确性验收和性能提升过程中值得分享的事情
     - 控制论方法论指导下的实践例子
     - 海森堡bug?

## 感悟

简单回顾一下参与这个项目的契机是吴伟老师演讲的时候, 说要打造个人开源影响力的部分有点吸引我, 我也想借这个机会提升自己的审美和换份更加有趣的工作. 现在所有的资金和商业上的焦虑都围绕在Ai的领域上, ~~但眼前大部分所谓Agent项目优化看着都不solid就想着开始圈钱了实在是讨厌~~

出于焦虑和对沉默成本的厌恶吧, 或者说毛躁, 这个项目是纯vibe的(感谢训练营, cnb, openai 慷慨的重置和我的钱包的支持), 在没有Token预算上限下做了一些激进的实验. 还是验证出来了一些方法论的, 我对控制论的皮毛理解让我成功地vibe了这个项目(安利一下金观涛华国凡的那个小册子), 主要有以下三点

- 共轭控制 : 其实就是曹冲称象的例子, 在基础设施不完善的时候直接在cnb上跑kimi是很困难的, 如果能换一些好控制的对象来研究那初期的迭代效率会高很多
  - cnb 和本地开发环境的联动, 用本地实验初步实验 KVM/TCG的性能开销等等
- 缩短迭代的决策周期 : 执行结果要40分钟才跑得完和100秒就能跑得完完全不是一个概念
  - cnb 上的增量编译的cache, 各种奇怪的技巧来加速模型的加载
  - 但是其实在拿到结果后用一系列的tool-calling 去获取到上下文也相当的浪费时间, 这个过程也应该自动化
- 提高可观测性 : 负反馈过程中如果反馈不够及时或者动作幅度过大都会导致反馈无法收敛, 进入震荡的状态
  - 我观测agent主要获取信息的方式似乎就是grep-like的工具, 并不怎么倾向于加日志, 更别提搭建观测框架

其实没有vibe coding的时候这些方法论也很重要, 只是现在它们的杠杆效应确实更大了. 我的理解某种意义上搭建这些基础设施的过程算是vibec-coding时期 get hands dirty 的一部分. 它其实都降低了新agent接手或者人类事后进入学习的成本.

同时我还想提的一个对比是, 古法编程时代和vibe-coding时代下, 给我的带来最深的负面情绪
- 如果一个bug放我以前de不出来, 挫败中最浓厚的情绪可能是无力
  - 不过那个时候我最起码有相当的代码是"已知"的, 在痛苦中挣扎出来后培养出来的直觉和手感多多少少能保护以后的我
  - 与此同时要花费大量的时间, 学习成本
- vibe coding 时期最浓厚的负面情绪是被愚弄的愤怒
  - 毕竟我不会我不会装自己会, 但看着agent 每次说`now i have the whole picture`然后一段瞎改螺旋收敛在死循环中烧我的token就火大
  - 一个迭代循环->遇到首次报错->fix后继续的循环看着没问题, 但是看它这个迭代循环里面循环了无数次之后, 问它有没有更高效的解决方案时它说有的时候是真气冲天灵盖
  - 不可控地遇到模型轮番降智和限速的问题

而从对agent-trace的关注来看, 这里相当的feature或者bug-fix交由我人来干, 别说bug-fix, 就算写出这个Bug我也不知道要到猴年马月了, 虽然也有很多时间我看goal的迭代过程也非常的不高效. 我也有很多时候能够感知到如果某些阶段我花全身心投入进去梳理一下脉络, 就算不能省很多时间也能省很多钱. 现在作为一个跑通的baseline 后作为学习和review 的效率可能会高一些, 如果这个项目后有更有趣的值得投入的后续, 更合理的方式是按照严肃的方式搬迁和重写.

也有点好奇这种仿真的意义是什么, 除了像 QEMU 这种后面成为虚拟化的基础设施了, 大部分仿真好像都是为了保护背后的商业决策来着, 单纯的仿真似乎就只是有趣而已.

## 正确性入围

先交差, 正确入围的前提是baseline = concordia, 虽然并没有要求它们的输出和不走虚拟化的结果一直, 但是我的项目验收这里还是做了的, Native 也因为对齐这个输出一致稍微损失了一点性能.

[cnb-k7g-1julm8qrn](https://cnb.cool/gevico.online/jensen/cxl-lab/-/build/logs/cnb-k7g-1julm8qrn) 提供 Native 首 token 数值 oracle 和固定输出, 随后，Same-VM paired 验收任务 [cnb-eqo-1k0ese01t](https://cnb.cool/gevico.online/jensen/cxl-lab/-/build/logs/cnb-eqo-1k0ese01t) 顺序执行 baseline 与 Concordia 并结果一致.

由此签署 baseline、Concordia 与 Native oracle 在该合同下正确性一致。下面是部分日志的截取, 和简单地分析一下性能的现状, 后面会在介绍性能测试的基础设施的时候会进一步讨论

### host run

- [host-run-cnb-lpo-1k0ek8t8u](https://cnb.cool/gevico.online/jensen/cxl-lab/-/build/logs/cnb-lpo-1k0ek8t8u) : 不走QEMU, 直接运行, 部分日志截取如下

```text
=== Kimi K2.6 host result ===
case=baseline status=pass tps=3.45 tokens=63 total_ms=22791.48
stdout_sha256=996046aee80b21708adc9aa7a0510a0299ab0857bb7a693decf5a0f5012e9c43 bytes=432
--- generated text ---
usersystemYou are Kimi, an AI assistant created by Moonshot AI.userSay that you have started in one short sentence.assistantassistantThe user wants me to say that I have started in one short sentence. I need to respond with a brief statement indicating that I (Kimi, the AI assistant) have started/become active. The user specifically asked for "one short sentence."

Possible responses:
- "I have started."
- "I am
> EOF by user


=== end ===
host_benchmark_finalize=pass
```

Host 给出直接 L40 路径的硬件利用率和软件阶段分解, 带了性能分析的cnb-job 是 [](), 日志中会有更详细的report,


### guest run

- [paired-run-cnb-eqo-1k0ese01t](https://cnb.cool/gevico.online/jensen/cxl-lab/-/build/logs/cnb-eqo-1k0ese01t) : 依次运行baseline 和 concordia 的结果, 截取日志片段如下

```text
=== Kimi K2.6 guest result ===
case=baseline status=pass tps=2.15 tokens=63 total_ms=69173.35
case=concordia status=pass tps=2.08 tokens=63 total_ms=60201.64
correctness=unavailable signing_eligible=false
stdout_sha256=996046aee80b21708adc9aa7a0510a0299ab0857bb7a693decf5a0f5012e9c43 bytes=432
--- generated text case=baseline ---
usersystemYou are Kimi, an AI assistant created by Moonshot AI.userSay that you have started in one short sentence.assistantassistantThe user wants me to say that I have started in one short sentence. I need to respond with a brief statement indicating that I (Kimi, the AI assistant) have started/become active. The user specifically asked for "one short sentence."

Possible responses:
- "I have started."
- "I am
> EOF by user


stdout_sha256=996046aee80b21708adc9aa7a0510a0299ab0857bb7a693decf5a0f5012e9c43 bytes=432
--- generated text case=concordia ---
usersystemYou are Kimi, an AI assistant created by Moonshot AI.userSay that you have started in one short sentence.assistantassistantThe user wants me to say that I have started in one short sentence. I need to respond with a brief statement indicating that I (Kimi, the AI assistant) have started/become active. The user specifically asked for "one short sentence."

Possible responses:
- "I have started."
- "I am
> EOF by user


=== end ===
```

基于Host 的性能分析, baseline 再给出虚拟化、供数和同步新增的成本，两份报告才能把差值归到具体路径上。

## 代码组织架构

下面介绍一下当前代码的组织架构, 所有代码可见[这里](https://cnb.cool/gevico.online/jensen), 模型 216 GiB，L40 只有 48G 显存, 所以模型的大部分都进不了显存, 但是如果能够把内存当显存来用(哪怕它延迟高一点)岂不美哉? 那就让 L40 靠 CXL 内存通路按需取数。并在这个基础上进行瓶颈的分析和优化, 下面请agent绘制了一个示意图

```text


       ┌──────── 构建: 独立构建, 产物以 digest 为身份 ──────────┐
   linux-cxl   cxlmemsim    llama     qemu-cxl   concordia  cxl-models
    (内核)    ┌────┴────┐   (GGML)    (设备模型)   (HetGPU)    216GB
              shim  server    │          │          │           │
       │       │        │     │          │          │           │
       ▼       ▼        │     ▼          ▼          ▼           │
   ┌──────────────────┐ │    ┌───────────────────────┐          │
   │   type2-guest    │ │    │    host: QEMU 进程    │          │
   │ kernel+shim+llama│ │    │ 设备模型+server+HetGPU│          │
   └────────┬─────────┘ │    └───────────┬───────────┘          │
            │           │                │                      │
            ▼           ▼                ▼                      ▼
   ┌─────────────────────────── 一次运行 ────────────────────────┐
   │                 216GB 模型 (virtiofs 只读共享)              │
   │                                                             │
   │  native 面:  llama ────────────────────────────► L40        │
   │                     (真机直跑, 出数值 oracle)      ▲        │
   │                                                    │        │
   │  guest 面:   llama → shim → /dev/cxl               │        │
   │                                 │                  │        │
   │                                 ▼ BAR2             │        │
   │                          (仿真的 CXL 链路)         │        │
   │                                 │                  │        │
   │                                 ▼                  │        │
   │                  设备模型 → server → HetGPU ───────┘        │
   │                  (一个 QEMU 进程, 一次 boot)                │
   │                                                             │
   │          两面共用: 同一份 216GB 模型, 同一张 L40            │
   └──────────────────────────────┬──────────────────────────────┘
                                  ▼
               result OCI → digest 校验恢复 → comparator → 证据
                   两面输出逐字节一致 + 首 token 数值 oracle
```


代码按运行时分成两部分：Guest 端跑应用，Host 端接真卡，中间通过 QEMU 的 CXL Type-2 设备模型来桥接。

- 在 Guest 里跑 llama 时，应用完全感知不到自己是在仿真环境里，依然按正常流程调 CUDA。这里的 libcuda 是个伪装层，由 cxlmemsim 生成的 Shim 会把每个 CUDA 调用翻译成硬件命令，直接写入 mmap 出来的 PCI BAR2 窗口。Guest 内核（linux-cxl-type2）负责把这块设备暴露出来，其中 /dev/cxl/cache0 和 /dev/cxl/mem0 两个节点分别对应缓存侧和内存侧。
    - llama 原则上是不应该修改代码的, 最多动一下编译或者运行参数什么的, 但是我为了可观测性还是做了一些改动
- 我理解导师提到的 cxl_gpu0 指的就是这块 Type-2 GPU 设备本身，QEMU 里的 ID 是 cxl-gpu0。Guest 侧由 cxl_type2_accel 驱动绑定，Shim 直接映射 BAR2 和它通信，不需要绕道字符设备。
- BAR2 的另一端是 qemu-cxl-type2。QEMU 收到命令后解析成 Host 层的行为，并下发给 Concordia；Concordia 在 Host 侧提供一套真实的 libnvcuda，把命令最终送到实际的 L40 显卡驱动上去执行。至于 type2-guest，主要就是把内核、Shim 和 llama 一起打成 Guest 镜像。

### 进度现状的定性分析

虽然目标是CXL-数据面的零拷贝扩容, 但是实际上还是没有做到的, 在项目博客延后一周那一段时间实际上项目进度已经到了左图copy-direct的状态了, 剩下的性能瓶颈确实是可以通过CXL来解, 但是vibe的过程把自己狠狠地破防了, 差点连项目都交付不了. 因此这里需要辨析一下

```text
当前 copy-direct                          ->              目标 CXL-backed system-UVA

Host 上的 exact 模型文件                                  Host 上的 exact 模型文件
          |                                                           |
          | virtiofs DAX                                              | 同一 file-backed payload
          v                                                           v
Guest llama mmap 文件页                                    CXLMemSim CXL.mem capacity pool
          |                                                           |
          | router 选择当前 expert                                    | QEMU 在 allocation 出生时分配 extent
          v                                                           v
selected source GVA                                        一个稳定的 system-UVA CUDA pointer
          |                                                           |
          | 0x35                                                      +-------------------------+
          v                                                           |                         |
QEMU 翻译 GVA、查 DAX mapping                                         | 低复用                  | 高复用
          |                                                           v                         v
          | pin / register                                GPU demand 读取 CXL pages     prefetch 同一 UVA 到显存
          v                                                           |                         |
NVIDIA Driver HtoD                                                   +------------+------------+
          |                                                                        |
          v                                                                        v
GPU-local staging / destination                                             CUDA kernel consumer
          |
          | completion / stream ordering
          v
CUDA kernel 从 VRAM 读取

每个 generation 重复支付：                                    长期只保留一份 payload 与一个 pointer：
GVA 翻译 + pin/register + HtoD + staging + wait                CXL.mem 提供容量，显存承接高复用工作集
```

TODO : 进一步的分析

### cxl-lab

鉴于这个项目是纯 vibe 跑起来的，我迫切地想让 agent 在 CNB 上有第一等的控制能力, 如果把 cxl-lab 当作一个开发平台产品来设计(需求来源是项目拓扑和实际开发中本来就有的复杂度), 实际上需要做的工作相当不少

乍一看好像只是一个迭代循环

```text
worktree 改动 -> 增量编译(slot) -> 制品 -> type2-guest 打包 -> QEMU 运行
     ^                                                        |
     |                                                        v
  诊断(Problem->action) <- 判读(record-cnb -> digest -> comparator)
```

但是这里的扰动其实还是挺多的. 因为一次运行是四个正交维度的闭合组合，平台要纳管的是维度本身：
run = 硬件平台(sm86/sm89/sm120/国产GPU)    决定执行后端
    × 编译选项(profile/toolchain digest)   决定制品身份
    × 运行选项(run spec/policy)            决定执行形状 --> 其实这两个我就合拢为软件了, 本身的拓扑什么的再说
    × 模型(task-scoped model view)         决定工作负载

怎么把这些东西管理好可以说只是有点麻烦, agent能设计看上去比较好的方案, 然后交给时间去打磨剩余的摩擦, 让控制链路尽可能保证短而粗即可.

---

但实际上需求不止这些, 而且可能还在演化, 因为实际上我们有两个开发平台, 它们之间其实还有不少各自的需求和复杂度需要处理

一次本地迭代 = 代码组合(多仓 × worktree × 分支)        改什么
             × 编译槽(组件 × 工具链 × profile)         怎么编
             × 执行后端(本地 KVM / CNB TCG / native)   在哪跑
             x 跑什么任务

因为在这个项目下的任务类型值得这样建模, 我们往往需要固定这样一个特定的流程反反复复迭代无数次, 而显式的暴露和治理这些东西能降低和解放agent和人类的心智负担, 并且在此基础上可能还有些延伸的诉求

- 代码组合 : 一个run可能需要不同仓库的不同分支的制品在不同参数的组合结果
  - 如果一大早就做了这个, 那项目的性能优化与我本地 sm86 开启HetGPU的二进制翻译的工作就真的可以同步进行
    - 当然我也烧不起token
- 编译槽 : rust/cpp 的编一次可不老时间, 不能每次都全量编译吧 -- 当然我也忍了很久
    - GC : 忍耐的结果就是不仅浪费时间, 还把我的磁盘给沾满了

在baseline还没有在cnb上跑通(以及跑通后tps低到离谱)的那段时间, 本地通过跑通了一个1.5B的小模型来验证了TCG/KVM的本身不可能带来那么大的虚拟化开销, 同时因为本地还能用perf之类的东西, 随便vibe一下后TCG/KVM的性能就追了上来

但在cnb上开发会有一些其他的坑和技巧需要关注

一次正式执行 = pipeline 构建(控制面收拢本地客户端, 有些可并行)
               -> 为什么不用cnb的工作台? L40的显存占用太多了, host 的脚本都跑步下来就会oom, 那就只能用流水线构建, 抖动程度会小很多
               -> 其实曾经是有不需要L40的验证链路是用cnb的开发平台来做的, agent 可以直接ssh上去, 频繁的流水线构建的主要耗时都在编译组装上
                 -> 但是如果产品化成熟后这类问题本来就该在本地开发, 因此这类坑的经验并不是很有价值
                    -> 那你说就没坑了么也不尽然, 工作区的自动回收并不尊重ssh上去正在操作的agent. 不过我也没有趟明白这个坑的解放, 我浏览器中点击包活似乎都没什么用, 可能得请教一下大白熊
             × 现场(构建期日志阻塞 -> 登录容器看),
               -> 见 cxl-lab 中的 `cnb_build_console.mjs` 的脚本, 可以让agent在流水线构建期间登录容器确认状态, 在任务卡死时有奇效
             × 制品生命周期
               -> [增量编译](https://cnb.cool/examples/ecosystem/use-cache)无需多言
               -> 有些明显不可能影响代码的commit其实也不需要重编
               -> 同时一个鲁棒的构建过程需要处理cnb有时候莫名其妙的gc

---

但但是需求实际上还不只这些, 因为我是要为agent搭建这个控制平台, **理论上把这个平台搭建好之后设置个goal给agent那它就可以自行迭代正确性问题和性能问题了吧?** 不是嚷嚷着控制论么, 还是太年轻了少年!

- cnb 一个任务可能要半个小时甚至若干个小时, 早期卡死被超时回收是常用的事情
  - 如果放任agent运行那它就会轮询, 而此刻的轮询并没有任何意义只会花token -- 除非在我的授意下用`cnb_build_console` 进去看看到底怎么个事
    - 在开发进入稳定期后, 大部分的时间甚至不在制品构建, 如果调度到了一个没有model cache的冷节点, 那拉取两百来G的模型到节点运行往往耗时就是最大的
      - 热缓存下端到端100s秒host能跑完, 冷的话或者部分的话二三十分钟也很正常
      - 指定节点似乎是cnb企业版的功能, 咱们没有, 那调度是随机的
        - 理论上可以探测到模型cache的程度估算出模型加载的耗时, 如果等太久了直接杀了起个新的撞到有cache的其实更快
    - 但是在哪之前, 其实应该要想办法hook住agent, 让它不要轮询烧我的token
      - 实践上是有用的, 但是调试hooks让它稳定地避免轮询查询要费点功夫
- 本地其实也不见得省心, 这个最大最大的痛点其实还是来源于本地 sm86 尝试做 HetGPU 的二进制翻译的工作, 浪费了大量的时间却没有收获成果, 这同时是可观测和迭代周期的问题
  - 其中有相当的部分是重复的tool-calling
    - 那为什么不写脚本? 本质上就是一个 `Problem -> action` 的trigger
       - 名义上很简单, 无非就是有core了触发gdb收集一些堆栈日志中之类的"if xxx 则执行xxx"的动作
       - 但随便乱vibe那可维护性就是各种各样的恶心, 而这可对上面的`cxl-lab`的控制平台提出了新的需求
       - 更极端的问题是too
  - agent 很容易在频繁的上下文收集(json中大海捞针)中反复压缩, 迷失自我
    - 负反馈无法收敛, 进入震荡

举个例子, 下面是codex在狠狠地破了我的心理防线和钱包但还没有把sm86上的GPU二进制翻译跑通的时候, 我质问它时它给我的回话的片段:

```
完整 Harness 本应在第一次 launch diff 后自动完成：

launch 1 first different
-> block 1 first different element
-> writer value异常、writer地址正常
-> 反向追踪 value 到 0x1020 LDG
-> 反向追踪 LDG 地址到 SHF/IMAD/UIADD3/LEA
-> source SASS 与 recovered PTX逐操作数比较
-> 发现 native SHF control=0x00fffdc0、recovered 使用普通 shr
-> 从 Gold 查找；没有则自动生成 PTXAS正向物化 case
-> 在模型执行前否决 shr/wrap/mask
-> 只准许有硬件 oracle支持的 clamp候选
-> fresh target运行
```

如果我能像rd那样不那么心浮气躁, 他其实似乎没有这些花里胡哨的东西, 用ds-flash和cnb的开发平台也一样把问题解了, 更古法编程一些. 效率其实更高, 而且也更省钱.
本来我还想着把这个基础设施做好后没准能在沐曦的 GPU 上跑通这个东西, 别的不说满血 `k3` 还是挺好用的, 得让我实现一段时间的token自由吧... 还是太年轻了

所以!

agent 友好性(选项可发现 × 报错可消化 × 失败可重放 × 等待不烧 token)
     × 性能观测判读(区间可 join × 扰动可控 × 判读固定链)

那这个性能观测判读区间就是下面要稍微多花一点笔墨来介绍的东西, 上面踩坑的其他细节都可以在我cnb上的代码中对照具体的细节, 实际上项目延期后主要时间其实都落在重构和开放cxl-lab上了, 上面的坑几乎全部重新又踩了个变, 还被迫吃了很多新的狗屎(为自己的延期交付小小的辩解一下)

综上, 类似 cxl-lab 的仓库本身是有一定的复杂度的, 上面这些需求如果要考虑到复用给其他人还不知道要踩多少坑.

### 性能观测链路(TODO)

一次 GPU 调用从 guest 里的应用发出，经过 shim、BAR2、QEMU 模拟的设备模型、Concordia，最后才到真实 GPU。时间的消耗可能发生在其中任何一段——guest 用户态、QEMU 里的命令处理、模拟器卡顿、真实驱动、数据传输。要归因，就得能看见整条链路的时间分布。

在正确性通过之后 -- 当然通过之前我就想要搭建一套日志系统来定位正确性通过之后的性能问题了, 现在向agent大人请教了一下这原来就是类似 OpenTelemetry span 的概念, 如果有了解的同学可能可以比较好类比.

但如果你和我一眼本来连span都不知道是什么, 那其实发明一遍它的概念也很简单

- span 是**一次有名有姓的操作区间**, 把一次请求的所有 span 组织起来，看它们的层级和时间包含关系。那就是一个低成本的观测性能开销的树了, TLDR 的版本就是这样, 具体的设计和实现细节可以去问agent.
  - guest 的 libcuda shim（libcuda.c）维护一个观测账本。每条命令执行时，shim 在 begin/end 记录时间，把命令、QEMU handler、driver 各自的耗时整理成嵌套区间——driver 区间 ⊂ qemu 区间 ⊂ 命令区间，时间包含关系就是层级
  - decode 结束，shim 把每个 category 的区间汇总成结构化 stderr 输出

- 同时伟大的agent大人又给我介绍众多性能工具
  - 如 `nsys`, 它可以在跑程序时在旁边记录 CUDA 侧发生的一切：每个 kernel 启动、每块显存拷贝、等等.., 并把这些记录存成一个 SQLite 数据库文件
    - 不过实际上它就是看GPU内的问题, 但大部分时间实际上GPU似乎都在等数据

实际上, 我本来许愿的也就是低成本的全链路观测(常驻) + 高能武器必要下的定点打击, 有了这样一个诉求后很容易能搓出来一个我们想要的东西. DSL 在这里就是我随口起的名字, 因为我本来还想这个东西可以运算和查询的.

```

低扰动层（始终开启）
  DSL / doctor / GPU+process 账本 / 日志账本
    -> 区分到"guest API - QEMU handler - Driver"边界
    -> 还不够时，升级
定点高能武器（显式 diagnostic profile，TPS 不进入 frontier）
  perf  -> 还是不能命名 CPU owner 时
  nsys  -> 还是不能区分 CUDA API/copy/stream/event/kernel 重叠时
  ncu   -> 时间线已经证明某个 kernel 主导后，以明确 kernel 与 launch 预算运行
```

---

下面这个图我看着很喜欢, 虽然我还不能很好地解释它, 但我还是决定留着

```text
┌ track A：guest 内（常开轨）───────────┐   ┌ track B：host 侧（定点开火轨）─────────┐
│                                       │   │                                        │
│  llama stderr                         │   │  nsys-rep                              │
│  [CXL-CUDA] 区间事件                  │   │  （GPU 时间线全量事件）                │
│  QEMU command trace                   │   │         │                              │
│         │                             │   │         ↓                              │
│         ↓                             │   │  analyze_nsys_sqlite.py                │
│  ┌─ reducer ────────────────┐         │   │  （离线区间运算                        │
│  │ 在线聚合 · 固定容量      │         │   │   merge/subtract/overlap）             │
│  │ 不落逐事件 · 无 ID 指针  │         │   │         │                              │
│  │                          │         │   │         ↓                              │
│  │ 嵌套（一条命令）         │         │   │  GPU 时间线解剖                        │
│  │  guest  |█████████████|  │         │   │  ┌───────────────────────┐             │
│  │  qemu     |███████|      │         │   │  │ |██|░|███|░░|██|░|██| │             │
│  │  driver     |██|         │         │   │  │  █ device active      │             │
│  │  违反嵌套 → incomplete   │         │   │  │  ░ outside / sync     │             │
│  │                          │         │   │  └───────────────────────┘             │
│  │ 集合运算                 │         │   │  HtoD 传输链（找供数空洞）             │
│  │  a   |██████|            │         │   │  Runtime API 配对 · 调用栈             │
│  │  b       |██████|        │         │   └──────────────────┬─────────────────────┘
│  │  c                |██|   │         │                      │
│  │  ∪   |██████████|  |██|  │         │                      │
│  │  ∩       |██|            │         │                      │
│  │  gap            |--|     │         │                      │
│  └───────────┬──────────────┘         │                      │
└──────────────┼────────────────────────┘                      │
               │                                               │
               ├────────── clock_anchor（时钟桥）──────────────┤
               │     guest 时钟 → host monotonic 映射          │
               │     不确定度有界；映射失败 → unavailable      │
               ↓                                               ↓
      ┌────────────────── join ────────────────────────────────────┐
      │  回答的问题：host 侧发现的空洞，是 guest 里谁造成的？      │
      │                                                            │
      │   host   |████|░░░░░░░|██|     ← nsys 发现的空洞           │
      │   guest      |████████|        ← guest category 区间       │
      │                ├── 有交集 → 空洞归这个 category            │
      │                └── 无交集 → 未归属 gap                     │
      │                     （标 previous/next 边界，留给下一轮）  │
      └─────────────────────────┬──────────────────────────────────┘
                                ↓
                        checkpoint record
                （唯一聚合事实；incomplete 是一等状态）
                                │
                                ↓
                  performance-report-*.md
                  （只读 checkpoint；自声明
                    joined / juxtaposed-only）

────────────────────────────────────────────────────────────────
观测扰动阶梯（决定 track B 何时何地开火）

低扰动 ←──────────────────────────────────────────→ 高扰动
DSL 账本 ──→ perf ──→ nsys ──→ ncu
 常开                                            定点
        前一层 unresolved / incomplete
        就是下一层的开火坐标
```

其实上面有说过 cxl-lab 需要接入trigger的机制的, 在cxl-lab重构之前, 我本地是有trigger机制能在合适的时候做perf的, 但是cnb上perf似乎开不了. 加上后面就更多进入cnb开发了, 那个工具就放着没管.

### 性能瓶颈的定量分析(TODO)

前面做了(还不存在的)定性分析, 又介绍了性能分析的框架, 那是是时候对性能瓶颈做定量分析了

cxl-lab 重构后开启nsys的host 和 baseline 有点问题, 想基于最新的数据去重新分析


## 开发时间线上值得记录的事情

TODO

- 海森堡bug 暴露的异步问题
- 无穷无尽的负优化
- 饿昏了迟点再补充...

