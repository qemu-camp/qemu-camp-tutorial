# 从 cmodel 说开来 - rd

!!! note "主要贡献者"

    - 作者：[@random25160765-collab](https://github.com/random25160765-collab)

> 感谢 @zevorn 老师搭建的实验框架和笔记分享平台。
>
> 这篇 blog 的内容是笔者完成 GPU 方向进阶实验 1 之后进行的一些技术探索，主要是个人的实验和理解。

---

## 1. cmodel

> 在 QEMU 里跑 cmodel 性能太差了，能不能让它快一点？下面分析瓶颈，逐条优化。

### 性能调优

| 瓶颈 | 对策 | 关键技术 |
|------|------|----------|
| QEMU TCG 译码放大计算量 | 解释器移到 QEMU 外部 | QEMU 设备仅做 guest↔host 管道 |
| 函数指针 + `switch` 分支开销 | `computed-goto` 线程化解释 | GCC `labels-as-values` + `ThOp` 预解码 |
| 译码和执行耦合 | 一次性预解码，执行时只走数组 | 译码/执行完全分离 |
| 未优化常用指令组合 | 新增 RVV + MMA 自定义指令 | 向量/矩阵操作硬件加速 |
| 锁步串行 | 线程池 block 级并发 | Go 重写调度逻辑 |

**问题：QEMU 模拟译码和运算** — 解释器放在 QEMU 设备里运行要走 [TCG 译码](https://qemu.gevico.online/tutorial/2026/ch2/qemu-tcg/)，每条 RISC-V 指令都要被翻译成若干条 x86 指令，总计算量翻了数倍。

- **对策：** 把整个解释器放到 QEMU 外面，QEMU 建模的设备充当 guest 和主机解释器的管道。

**问题：函数指针的设计开销大** — `switch`/`case` 和函数指针带来大量分支和跳转开销。

- **对策：** 采取 `computed-goto` 设计（思路来源于 [泽文老师的线索解释](https://qemu.gevico.online/blogs/misc/simulater-interp/#_3)），使用 GCC 的 `labels-as-values` 扩展。内核执行前一次性预解码为 `ThOp` 数组，每条指令的 `handler` 字段指向对应的 `&&op_xxx` 标签：

```c
// core/engine.c — 预解码后的 computed-goto 分发循环
#define NEXT()                                              \
    do {                                                    \
        gpr[0 * 32 + 0] = ... = gpr[0 * 32 + 31] = 0;     \
        goto *ip++->handler;                                \
    } while (0)

goto *ip++->handler;   // 首条指令

#include "../module/modules.h"   // 聚合所有 ISA handler 标签
```

dispatch 表的填充通过 YAML spec → Python 生成器 → `INSTRUCTION_LIST` 宏自动完成。`INSTRUCTION_LIST` 被两处引用：

`dispatch.h` 用它构建 **trie 解码表**，把每条指令的 bit pattern 转为 mask/match 条目：

```c
// core/dispatch.h — trie 构造 (每条指令注册 mask/match/type 信息)
#define X(name, pattern, op_type, imm_fn)                              \
    d->op_table[idx].mask = pattern_to_mask(pattern),                  \
    d->op_table[idx].match = pattern_to_match(pattern),                \
    d->op_table[idx].exec = NULL, d->op_table[idx].type = op_type, idx++
    { INSTRUCTION_LIST; }
#undef X
```

`engine.c` 用它填充 **computed-goto 跳转表**，把 `DISP_` 枚举索引映射到 `&&op_xxx` 标签地址：

```c
// core/engine.c — computed-goto dispatch 表初始化
#define X(name, pattern, op_type, imm_fn) \
    dispatch[di++] = &&op_##name;         // GCC labels-as-values
    INSTRUCTION_LIST;
#undef X
```

代价：engine 执行函数变得非常大，handler 得自动生成了。不过用代码量换性能，不亏。

**问题：译码和执行完全耦合** — 翻译一条指令执行一条指令，热路径上有解码开销。

- **对策：** kernel 二进制在执行前一次性预解码，执行时只需顺着数组走，解码和执行完全分离。

**问题：未针对常用的指令组合优化** — 最常见的是向量操作和矩阵操作。

- **对策：** 加一个 RVV 扩展，再加上一些 MMA（矩阵乘加）自定义指令。之前的 kernel 全部用新的指令重写。

其实这里本来的设计是图优化，针对特定指令序列进行融合。换个角度想，自定义指令就是这个意思。

#### 问题：锁步串行执行

- **对策：** 使用线程池实现简单的 block 级并发（C 语言写并发很让人头痛，因此我把调度逻辑用 Go 重新写了一遍）。

??? note "更多优化手段"

    **压缩前缀树 (decode trie)** — 在 32-bit 指令空间上构建 bit-by-bit 决策树，压缩单分支路径。每条指令的 bit pattern 注册为 trie 节点，查找从逐条比对变成逐位下降，复杂度从 $O(n)$ 降到 $O(n\log n)$：

    ```c
    // core/decode_trie.h — 快速查找：逐位下降直到叶子
    static inline uint16_t decode_trie_lookup(const DecodeTrie *trie, uint32_t inst)
    {
        uint16_t idx = trie->root_idx;
        const TrieNode *nodes = trie->nodes;
        while (nodes[idx].bit_pos >= 0) {
            int bit = (inst >> nodes[idx].bit_pos) & 1;
            uint16_t next = nodes[idx].child[bit];
            if (next == 0) return 0; /* illegal */
            idx = next;
        }
        return nodes[idx].instr_id;
    }
    ```

    **SoA 寄存器排布** — 同一寄存器的 32 个 lane 值在内存中连续排列（`gpr[reg*32 + lane]`），而非每个 lane 的结构体里存自己的寄存器组。编译器遇到连续内存访问时可以自动向量化（SIMD），性能提升显著：

    ```c
    // core/soa.h — AoS → SoA marshalling
    #define GPR(reg, lane) gpr[(reg) * 32 + (lane)]
    #define FOR_EACH_LANE \
        for (int _li = 0; _li < 32; _li++) \
            if ((_active >> _li) & 1)

    // 每次执行前把 warp 的 AoS 数据转成 SoA 布局
    static inline void aos_to_soa(const GPGPUWarp *warp, uint32_t gpr[32*32], ...)
    {
        for (int lane = 0; lane < 32; lane++)
            for (int r = 0; r < GPGPU_NUM_REGS; r++)
                gpr[r * 32 + lane] = warp->lanes[lane].gpr[r].u32;
    }
    ```

    **快速近似算法 (针对 SFU)** — 不调 `math.h`，用 Taylor 级数 + 参数归约实现 `exp`/`log`/`sin` 等。精度 <1%，速度是 libm 的 3-5 倍：

    ```c
    // core/sfu.h — fastexp: Taylor e^r, 5 项截断
    static inline float fastexp(float x)
    {
        float k = (float)(int32_t)(x * 1.442695f + 0.5f);   // k = round(x/ln2)
        float r = x - k * 0.69314718f;                       // r ∈ [-ln2/2, ln2/2]
        float r2 = r * r;
        float er = 1.0f + r + r2/2.0f + r2*r/6.0f + r2*r2/24.0f + r2*r2*r/120.0f;
        return ldexpf(er, (int)k);   // e^x = er * 2^k
    }
    ```

    **`vsetvli` dispatch 热替换** — RVV 的 SEW 变化时（e8↔e16↔e32），不重新译码，而是原地改写 `dispatch[]` 中的标签地址。Python 生成的 `dispatch_rebind.h` 提供 `RVV_DISPATCH_REBIND()` 宏，直接 `goto` 到新宽度的 handler：

    ```c
    // inst/dispatch_rebind.h — vsetvli 触发，原地热替换
    #define RVV_DISPATCH_REBIND(disp, sew) do { \
        switch (sew) { \
        case 8:  disp[DISP_VADD_VV] = &&rvv_vadd_vv_e8; ... break; \
        case 16: disp[DISP_VADD_VV] = &&rvv_vadd_vv_e16; ... break; \
        case 32: disp[DISP_VADD_VV] = &&rvv_vadd_vv_e32; ... break; \
        } \
    } while(0)
    ```
    其他的诸如弃用 softfloat 库等就省略了。

（其实直接写 JIT 会更好，但是我不想，因为太难了……）

在 agent 帮助下，性能调优的过程已经高度自动化：先确定热路径/经常性事件，再让 agent 梳理调用链，估计每个环节的开销并定位性能瓶颈，然后针对性优化就可以。整个过程中人只需要做方向和安全上的把关。

在优化的过程中，也能轻易发现一些优化的取舍：算法优化（从 $O(n)$ 到 $O(n \log n)$）vs 硬件适配（设计 cache 友好的算法）——往往后者比前者提升大得多。

---

### 架构设计

> 加入的指令集太多，直接手写 `computed-goto` 的 label handler 已经完全不可能——几百个 label、几千行代码，AI 读这种东西幻觉十分严重。于是我设计了一种”多级元编程”的方案。

!!! note "关于 AI 幻觉"

    AI 对数据本身非常不敏感——指令的二进制编码、高度相似且抽象的 label handler，AI 看这些很高效，但**改**这些简直是灾难，非常容易出错。

    AI 喜欢用 `sed` 和 Python 脚本做复杂 regex 匹配，对于上下文耦合紧的长文件，非常容易匹配错误，而且一错一大片，浪费大量时间。最佳实践：不要让 AI 直接处理这种东西，尽量用代码生成。要么就人来改，AI 只看。

灵感来源于 Spike，但做了大量简化。首先是模块的生成：

1. **YAML spec** — 定义每条指令的 bit pattern、操作类型、立即数格式
2. **Python 生成器** — 根据 YAML 吐出 `dispatch_list.h`（`INSTRUCTION_LIST` + `DISP_` 枚举）和 `dispatch_rebind.h`（RVV SEW 热替换）
3. **`#include` 模块打包** — 按硬件单元聚合（`module/`），分离 ISA 规格和微架构
4. **插入执行函数** — `engine.c` 统一 `#include “../module/modules.h”`

```mermaid
flowchart LR
    YAML[YAML spec]
    YAML -->|gen_dispatch.py| DISP_LIST["dispatch_list.h<br/>+ dispatch_rebind.h"]
    YAML -->|gen_&lt;isa&gt;.py| HANDLES["handles/<br/>op_xxx label"]

    DISP_LIST -->|"trie (op_table)"| DISPATCH_H[dispatch.h]
    DISP_LIST -->|"dispatch[] + rebind"| ENGINE[engine.c]

    HANDLES -->|#include| MODULES[modules.h]
    MODULES -->|"op_xxx: labels"| ENGINE

    DISPATCH_H -->|"SIMDDecoder"| ENGINE
```

YAML spec 中每条指令的 decode 信息驱动了整个流程：

```yaml
# inst/rvv/scripts/rvv_spec.yaml
- name: vadd
  op: "+"
  itype: int
  decode:
    label: "op_vadd_vv"          # → DISP_VADD_VV 枚举值
    pattern: "000000? ????? ????? 000 ????? 10101 11"
    type: TYPE_R
    imm: imm0
    variants:                    # SEW 变体 → dispatch_rebind.h
      base: "vadd_vv"
      sew: [8, 16, 32]
```

Python 生成器扫描所有 ISA 的 YAML，收集每条指令的 decode 信息，统一生成 `INSTRUCTION_LIST`。`engine.c` 初始化时通过 `#define X(name, ...) dispatch[di++] = &&op_##name` 一次性填充 dispatch 表。RVV 的 `vsetvli` 指令触发 `RVV_DISPATCH_REBIND()` 宏，根据当前 `SEW` 把 dispatch 槽原地重定向到对应宽度的 handler（如 `&&rvv_vadd_vv_e8`），绕开条件判断。

模块的裁剪和配置可以通过增删头文件来进行。关于配置，我使用了 lua 脚本来进行解释器的配置，好处是可以实现配置热更新而不必重新编译。

目前 Python 生成器耦合太重——直接把解释器细节硬编码进去，导致执行函数和生成器的上下文难以对齐，生成器代码跟着指令数量一起膨胀。**这里必须设计一个 IR 来解耦**，像真正的工业级解释器那样。不过这是以后的事情了。

### 反思

!!! warning "一个不伦不类的东西"

    这个项目既不是完全的解释器，也不是完全的 GPU 模拟器，两边都只实现了一部分功能，并且互相拖累：

    - 因为要当**解释器**，所以选了 `computed-goto` + `#include`，但是元编程做得太浅，没有 IR
    - 因为要当 **GPU 模拟器**，所以引入了 lane/SIMT/barrier 的概念，但实现方式是 `for` 循环 + 栈——这些概念没有对应任何微架构行为

    这也在预料范围内——扩展 cmodel 完全是临时起意，只是想让它”跑得更快”。

代码已开源在 <https://github.com/random25160765-collab/little-gpu-cmodel>。

---

## 2. Vortex 和 POCL

> 这部分是最早做的——当时还很缺乏驾驭一个 GitHub 项目的经验，只会问 AI，踩了非常多的坑。

Vortex 的架构在它的 Deepwiki 上已经讲的很清楚了，关于桥接的其他博客也做了详细的描述。因此接下来的内容不会重复这些，只是随便聊聊，分享一些资料，回顾一下踩坑的过程。

### 桥接 Vortex SimX 与 QEMU

首先要明确的是要做的是一个**桥接工作**，因此得先找到 Vortex 这边的运行接口，在桥接层传入配置并做好兼容。SimX 是一个完整程序，上层包装做得非常好，整个运行流程可以归纳如下：

```text
1. 配置 Arch（配置结构体）
     │
2. 配置 RAM
     │
3. processor 绑定 RAM，配置寄存器
     │
4. 传入指令文件
     │
5. processor 运行
     │
6. 从 RAM 里读回结果
```

可以看到，整个流程十分简单而清晰（具体可以看 `main.cpp`）。桥接层的代码可以照抄上面的这个流程。不过实际上做起来还是有一些坑的：

- **内存一致性** — SimX 是一个完整的 GPU 模拟器，有自己的内存系统和寄存器；QEMU 这边也有 VRAM 和寄存器。客户机驱动把配置和二进制文件写入设备前端的内存，每次执行 kernel 时要对 SimX 内部进行清除和同步，保证 SimX 读取的是设备前端中的最新信息。
- **同步阻塞** — 必须要移到独立线程，否则会卡住 QEMU 主循环。

### 桥接 POCL 与 Vortex SimX

!!! warning "踩坑：造了一堆轮子，最后发现目标后端竟然是 CPU"

    1. 漏看了实验描述，以为 Vortex 没有做 POCL 相关的适配工作
    2. 没有用 Vortex 维护的 LLVM，而是自己下载了一个
    3. Coding agent 为了把这堆东西接起来到处缝补——不仅缝补，还编测试用例，制造出测试全对的表象
    4. 多测两遍发现每次结果都不一样，追溯到最后发现是 **ABI 问题**，POCL 的目标后端竟然是 CPU

    这个时候我才发现有个网站叫做 deepwiki，打开一看才发现之前连续几天的工作全在造轮子。

### 关于 Vortex

之前想对 Vortex simx 的源码做一个解读，但周期级仿真引擎的基础代码过于复杂，到处都是模板元编程和各种复杂的嵌套类，耦合太重，因此只写了一篇就懒得继续写了（或许是我方向搞错了，应该解读 Vortex 的架构本身而非仿真引擎）文章放在 github 上了：[a note of vortex simx](https://gist.github.com/random25160765-collab/530368f76545f2121e2a4ac4b081c42d)。

知乎上有个搞 GPGPU 全栈的大佬做了一些 Vortex SimX 源码的解读和魔改，链接也放在下面，可供参考：<https://www.zhihu.com/people/yahah-97>；CSDN 上也有一些资料：[Vortex GPGPU 的硬件架构和代码结构分析-CSDN 博客](https://blog.csdn.net/weixin_41029027/article/details/140276734)（不过这些都是基于旧版本 Vortex 的解读了）

btw，近日更新的 vortex 3.0 实现了一大批新功能，覆盖了 AI 推理场景下的很多功能。详情可见 <https://github.com/vortexgpgpu/vortex/releases>

---

## 3. 可观测系统

> 被跨层级的 bug 折磨得痛不欲生，或许我需要一个更友善、更先进的监控系统。

??? note "什么是可观测性？"

    > 可观测性这一术语源于控制理论，可观测性是衡量一个系统从其外部输出的知识中推断系统内部状态的一种度量。换句话说，如果你可以观察系统的外部以确定它内部发生了什么，那么该系统就具有可观测性。
    >
    > 在 IT 运维领域，是指获知基础设施、编排平台和服务应用所有层面的必要信息，从而观察所有系统的各类行为是否存在异常。可观测性是通过对开发测试、IT 运维、业务运营、安全合规等全业务运营流程，借助日志、指标、链路等机器数据进行关联分析，衡量、预防、发现、定位、解决业务问题，实现业务效能提升的一种能力。
    >
    > 可观测性是从系统内部出发，基于白盒化的思路去监测系统内部的运行情况。可观测性贯穿应用开发的整个生命周期，通过分析应用的指标、日志和链路等数据，构建完整的观测模型，从而实现故障诊断、根因分析和快速恢复。

    *摘自知乎：[一文搞懂：可观测性到底是什么？](https://zhuanlan.zhihu.com/p/605914260)*

### trace 系统优化

虽说”可观测系统”生态主要围绕云原生/分布式计算展开，现有项目的复杂度与之相差甚远；但监控和观测的基础设施对于小项目同样重要。借此我把 cmodel 的 trace 系统做了优化。

#### 问题：指令级日志 I/O 开销大 + 不同速率的日志互相淹没

- **对策：** 使用无锁环形缓冲区 + 双通道设计。控制事件走 `slow_ring`，指令 trace 走 `fast_ring`，互不干扰：

```c
// hw/gpgpu/vpu/ring/ring.h
typedef struct ring_buf {
    uint8_t *buf;
    size_t   size;
    _Atomic uint32_t r;     // 读指针
    _Atomic uint32_t w;     // 写指针
} ring_buf;

int ring_buf_peek(ring_buf *rb, struct iovec iov[2]);   // 零拷贝读取
void ring_buf_commit(ring_buf *rb, size_t len);          // 消费已读数据
int ring_buf_write(ring_buf *rb, const uint8_t *src, size_t len);
```

**问题：日志体积膨胀** — 一条完整的文本日志包含大量冗余字符：

```text
qemu_log(“\t\t[LH] rd=%d, addr=0x%x, val=0x%04x, sign-extended=0x%x\n”,
         rd, addr, val, (int32_t)(val << 16) >> 16)
```

- **对策：** 把 trace 改造成二进制编码。有效信息只有指令名和两个操作数，用紧凑的二进制帧编码，trace 体积大幅缩小，便于事后分析。协议用 Python 脚本根据 spec 自动生成 `pt_inst.h` 和 `pt_event.h`，平台和 model 共用一份协议确保一致性。

### 观测平台架构

```mermaid
flowchart LR
    VPU[VPU / QEMU 设备] -->|fast_ring| FAST[fast_fd<br/>指令 trace 高频]
    VPU -->|slow_ring| SLOW[slow_fd<br/>控制事件 低频]
    FAST --> PROBE[Unix Socket<br/>probe.c]
    SLOW --> PROBE
    PROBE --> BACKEND[后端 Python<br/>采样 → 协议解码 → JSON]
    BACKEND --> FRONTEND[前端 HTML/JS<br/>渲染 + 结构化展示]
```

model 通过 `probe.c` 创建 Unix socket server，向观测平台暴露两个 ring buffer 通道：

```c
// hw/gpgpu/vpu/socket/probe.c — 双通道 Unix socket server
static int create_server(const char *path)
{
    int fd = socket(AF_UNIX, SOCK_STREAM, 0);
    if (fd < 0) { perror("probe: socket"); return -1; }

    unlink(path);

    struct sockaddr_un addr;
    memset(&addr, 0, sizeof(addr));
    addr.sun_family = AF_UNIX;
    strncpy(addr.sun_path, path, sizeof(addr.sun_path) - 1);

    if (bind(fd, (struct sockaddr *)&addr, sizeof(addr)) < 0) {
        perror("probe: bind"); close(fd); return -1;
    }
    if (listen(fd, 1) < 0) {
        perror("probe: listen"); close(fd); return -1;
    }
    return fd;
}
```

### 后续方向

这个”观测系统”还有不少可以提升的地方：可以把检测逻辑融入解码过程，实现运行时自动化故障拦截；目前只实现了 trace 和一部分 log，还可以添加 metrics；事件节点和解码协议的设计可以更讲究；观测系统可以拓展到 guest 内部 → QEMU → 解释器的全链路；前端展示可以更友好，增加波形与时序图等结构化展示……

不过在这之前，还是先学习一下 Prometheus、Elastic Stack、Grafana 等商业可观测产品的思路吧。

### 模拟云计算？

> 既然已经把解释器放在 QEMU 外部了，让 QEMU 设备控制一整个”运算集群”就是很自然的想法——就是管理一堆解释器进程嘛。

不过这个想法很快就作罢了：

- 多线程的管理调度很复杂，无法驾驭
- 笔记本电脑吃不消，多开几个解释器 WSL 就要崩溃——整个系统的模拟包括开发都用同一个 CPU 的算力

该节涉及的代码开源在 <https://github.com/random25160765-collab/qemu-riscv-simt-gpu> 的 `visible_system` 分支。

---

## 总结

三个方向看上去各自独立——解释器性能优化、GPU 仿真器桥接、可观测平台搭建——但回过头看，它们围绕的是同一件事：**怎么让一个软件模拟器更快、更好用、更可理解。** cmodel 的 `computed-goto` + 多级元编程把译码和执行分离了，但 Python 生成器没有 IR，耦合跟着指令数量一起膨胀；Vortex 桥接让我踩了"不读文档先动手"的坑，POCL 方向白干了几天才发现目标后端是 CPU；可观测系统用 ring buffer + Unix socket + 二进制协议搭了一个麻雀虽小的观测平台，但离 Prometheus / Grafana 那样的成熟方案还有很长的路。三个方向都远没有做完，但每个都让我对"模拟器之外的东西"有了更具体的理解。

---

## 参考资料

- [QEMU TCG 工作原理](https://qemu.gevico.online/tutorial/2026/ch2/qemu-tcg/) — 训练营讲义
- [浅析虚拟机软件仿真中的解释技术](https://qemu.gevico.online/blogs/misc/simulater-interp/) — 泽文，线索解释 / computed-goto 思路来源
- [QEMU GPGPU 模拟](https://qemu.gevico.online/tutorial/2026/ch2/qemu-gpgpu/) — 训练营讲义
- [QEMU PCIe 模拟方法](https://qemu.gevico.online/tutorial/2026/ch2/qemu-pcie/) — 训练营讲义
- [GPU 进阶实验](https://qemu.gevico.online/exercise/2026/stage2/gpu/) — 实验手册
