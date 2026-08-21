# QEMU 训练营 2026 专业阶段总结

!!! note "主要贡献者"

    - 作者：[@random25160765-collab](https://github.com/random25160765-collab)

以下是专业阶段实验的学习过程总结。

## 背景介绍

电子信息工程专业学生，对系统编程/虚拟化/AI infra 感兴趣。

## 专业阶段

笔者完成的是专业阶段 GPU 方向的建模实验，共 10 个实验，17 个测试样例。

### 项目结构

```text
.
├── Kconfig
├── gpgpu.c
├── gpgpu.h
├── gpgpu_core.c
├── gpgpu_core.h
├── inst.h        // 指令和译码相关的宏
├── lpfp.c        // 低精度浮点数的处理
├── lpfp.h
├── memory.h      // 访存相关的宏和函数
├── meson.build
└── utils.h       // 一些工具宏

```

### 实验内容

实验 1-6 对应的内容主要是内存读写，并未涉及复杂逻辑。绝大部分的代码已经写好，在 mmio 读写函数和显存读写里加上几条 case 即可：

```c
// mmio 读写示例
switch (addr) {
 case GPGPU_REG_DEV_ID:
  val = GPGPU_DEV_ID_VALUE;
  break;
 case GPGPU_REG_DEV_VERSION:
  val = GPGPU_DEV_VERSION_VALUE;
  break;
 case GPGPU_REG_VRAM_SIZE_LO:
  val = 0x04000000;
  break;
 case GPGPU_REG_VRAM_SIZE_HI:
  val = 0x00000000;
  break;
 case GPGPU_REG_GLOBAL_CTRL:
  val = gpu->global_ctrl;
  break;
 case GPGPU_REG_GLOBAL_STATUS:
  val = gpu->global_status;
  break;
  ...
```

```c
// vram 读写示例
if(addr + size <= gpu->vram_size) {
 switch (size) {
  case 1:
   val = *(uint8_t*)(gpu->vram_ptr + addr);
   break;
  case 2:
   val = *(uint16_t*)(gpu->vram_ptr + addr);
   break;
  case 4:
   val = *(uint32_t*)(gpu->vram_ptr + addr);
   break;
  case 8:
   val = *(uint64_t*)(gpu->vram_ptr + addr);
   break;
 }
}
...
```

剩下的 case 就不展示了，只需根据手册和 gpgpu.h 头文件中定义的结构体和宏一个一个填进去即可。

实验 7-10 要求实现一个简单的 RISC-V 指令模拟器，支持 RV32IF 和一些低精度浮点转换。RV32I 的部分比较简单，只需要完成 8 条指令（add, addi, slli, andi, lui, sw, ebreak, csrrs）的解析。RV32F 则需要完成 5 条指令（fadd.s, fmul.s, fcvt.s.w, fcvt.w.s, fmv.w.x），不同精度的浮点转换指令则需要完成 8 条（bf16, e4m3, e5m2, e2m1 和 s 分别来回转换）。

备注：csrrs 的立即数是零扩展，而非符号扩展。

这一部分的实现有很多方式。一开始我让 ai 帮我打个样，结果 ai 使用了大量的 if-else 嵌套，虽能通过测试但扩展性极差。恰巧笔者做过南大 PA，对 NEMU 比较熟悉，因此想用宏的方式来简化实现，提高可扩展性。

```c
// NEMU 的指令处理 (笔者在南大 PA 中编写的)
  INSTPAT("0000000 ????? ????? 000 ????? 01100 11", add    , R, R(rd) = src1 + src2);
  INSTPAT("0100000 ????? ????? 000 ????? 01100 11", sub    , R, R(rd) = (int32_t)src1 - (int32_t)src2);
  INSTPAT("0000000 ????? ????? 001 ????? 01100 11", sll    , R, R(rd) = (uint32_t)src1 << (src2 & 0x1F)); // 取 src2 的低 5 位
  INSTPAT("0000000 ????? ????? 010 ????? 01100 11", slt    , R, R(rd) = ((int32_t)src1 < (int32_t)src2) ? 1 : 0);
  INSTPAT("0000000 ????? ????? 011 ????? 01100 11", sltu   , R, R(rd) = ((uint32_t)src1 < (uint32_t)src2) ? 1 : 0);
  ...
```

在这之前别忘了把 kernel 接上设备，只需要再补上一个 case 即可

```c
case GPGPU_REG_DISPATCH: // 触发 Kernel 执行入口
 gpu->global_status = GPGPU_STATUS_BUSY;
 int ret = gpgpu_core_exec_kernel(gpu);
 if (ret == 0) {
  gpu->global_status = GPGPU_STATUS_READY;
  gpu->irq_status |= GPGPU_IRQ_KERNEL_DONE;
 } else {
  gpu->global_status = GPGPU_STATUS_ERROR;
  gpu->error_status |= GPGPU_ERR_KERNEL_FAULT;
  gpu->irq_status |= GPGPU_IRQ_ERROR;
 }
 break;
```

GPU core 是 SIMT 架构，每一个 warp 有 32 个 lane，共享同一个 warp 的指令上下文。

我的思路是：

1. 先查表确定是哪一条指令。
2. 分离出指令的数据并存进 warp 的上下文里。
3. 每一个 lane 读取所属 warp 的指令上下文，执行运算。

备注：由于 warp 的指令上下文是只读的，故不会产生并发和数据竞争风险。

首先是构建指令表。笔者选择的方案是保留 NEMU 风格的二进制字符串，从二进制字符串中直接读取 mask 和 match（识别指令用）和指令的名称和类别。

```c
#define INSTRUCTION_LIST \
    /* RV32I */ \
    X(add,          "0000000 ????? ????? 000 ????? 01100 11", TYPE_R); \
    X(addi,         "??????? ????? ????? 000 ????? 00100 11", TYPE_I); \
    X(slli,         "0000000 ????? ????? 001 ????? 00100 11", TYPE_I); \
    X(andi,         "??????? ????? ????? 111 ????? 00100 11", TYPE_I); \
    X(lui,          "??????? ????? ????? ??? ????? 01101 11", TYPE_U); \
    X(sw,           "??????? ????? ????? 010 ????? 01000 11", TYPE_S); \
    X(csrrs,        "??????? ????? ????? 010 ????? 11100 11", TYPE_CSR); \
    X(ebreak,       "0000000 00001 00000 000 00000 11100 11", TYPE_I); \
    ...
    
static void __attribute__((constructor)) init_opcode_table(void)
{
    int idx = 0;
    
#define X(name, pattern, op_type) \
    do { \
        opcode_table[idx].mask = pattern_to_mask(pattern); \
        opcode_table[idx].match = pattern_to_match(pattern); \
        opcode_table[idx].exec = exec_##name; \
        opcode_table[idx].type = op_type; \
        idx++; \
    } while(0)
    
    INSTRUCTION_LIST
    
#undef X
}
```

通过 `__attribute__((constructor))` ，指令表在初始化阶段根据填入的信息构建完毕。每一次传入新的指令，都遍历一次指令表，找到指令后对指令进行拆解并将信息存入上下文，并调用相应的函数。

```c
// 查表
static opcode_entry_t *lookup_opcode(uint32_t inst)
{    
    for (size_t i = 0; i < opcode_table_count; i++) {
        if ((inst & opcode_table[i].mask) == opcode_table[i].match) {
            return &opcode_table[i];
        }
    }
    return NULL;
}
```

```c
// 构建 warp 上下文
static void get_warp_ctx(exec_ctx_t *ctx, uint32_t inst, int type)
{
    ctx->rd  = BITS(inst, 11, 7);
    ctx->rs1 = BITS(inst, 19, 15);
    ctx->rs2 = BITS(inst, 24, 20);
    ctx->rs3 = BITS(inst, 31, 27);

    switch (type) {
        case TYPE_I: case TYPE_FI:
            ctx->imm = immI(inst); break;
        case TYPE_S: case TYPE_FS:
            ctx->imm = immS(inst); break;
        case TYPE_U:
   ...
```

回调函数根据函数名和传入的逻辑在预处理阶段自动生成。最初的设计是所有回调函数共享统一的上下文。

```c
// 回调函数构造
#define INIT_LANE_CONTEXT() \
    GPGPULane *l = &ctx->warp->lanes[lane_id]; \
    uint32_t src1_u32 = IS_FP_SRC_TYPE(ctx->type) ? l->fpr[ctx->rs1] : l->gpr[ctx->rs1]; \
    uint32_t src2_u32 = IS_FP_SRC_TYPE(ctx->type) ? l->fpr[ctx->rs2] : l->gpr[ctx->rs2]; \
    uint32_t src3_u32 = l->fpr[ctx->rs3]; \
    int32_t imm = ctx->imm; \
    float src1_f, src2_f, src3_f; \
    if (IS_FP_DST_TYPE(ctx->type)) { \
        memcpy(&src1_f, &src1_u32, sizeof(float)); \
        memcpy(&src2_f, &src2_u32, sizeof(float)); \
        memcpy(&src3_f, &src3_u32, sizeof(float)); \
    } else { \
        src1_f = 0; src2_f = 0; src3_f = 0; \
    } \
    uint32_t src1 = src1_u32; \
    uint32_t src2 = src2_u32; \
    uint32_t src3 = src3_u32; \
    (void)src1; (void)src2; (void)src3; (void)imm; (void)src1_f; (void)src2_f; (void)src3_f;

#define EXEC_FUNC(name, code) \
    static void __attribute__((unused)) exec_##name(exec_ctx_t *ctx, int lane_id) { \
        INIT_LANE_CONTEXT(); \
        code \
    }
...

/* RV32I */
EXEC_FUNC(add,      { G(rd) = src1 + src2; })
EXEC_FUNC(addi,     { G(rd) = src1 + imm; })
EXEC_FUNC(slli,     { G(rd) = src1 << (imm & 0x1F); })
EXEC_FUNC(andi,     { G(rd) = src1 & imm; })
EXEC_FUNC(lui,      { G(rd) = imm; })
...
```

但这会导致生成的回调函数极其臃肿。目前笔者采取的还是该方案，暂时能够通过测试。后续有时间会进行优化，根据指令的不同加载不同的上下文，减少回调函数中由于指令兼容导致的冗余逻辑。

GPU 在一个 warp 里串行调用回调函数，完成计算。

### Debug 与经验

1.单个测例可以在 gdb 里调试，看输出。

```shell
export QTEST_QEMU_BINARY="gdb --args {YOUR_PATH}/qemu-camp-2026-exper-{YOUR_NAME}/build/qemu-system-riscv64" 
export QTEST_QEMU_OPTIONS="-S"
{YOUR_PATH}/qemu-camp-2026-exper-random25160765-collab/build/tests/qtest/qos-test -p /riscv64/virt/generic-pcihost/pci-bus-generic/pci-bus/gpgpu/gpgpu-tests/{TEST_NAME} --tap -k
```

让 ai 给程序打上 debug 用的 printf，然后把输出的日志给 ai 分析。一般情况下，ai 会快速发现日志中不对的地方，但具体是为什么不对，还是需要自己找，ai 不一定有头绪。

2.指令的格式以 riscv-manual 和 datasheet 为唯一标准，务必亲自检查。可以把 manual 和 datasheet 给 ai 看，让它生成可读性更好的结果，但不要直接让它生成指令。我严重怀疑 ai 分不清 0 和 1.

3.heisenbug 的解决：执行浮点指令时，出现了某一个 lane 的浮点寄存器写入故障的情况。原因是指令表的大小不对，初始化时发生了越界写入，导致后面的数据被覆盖。当晚 ai 用临时匹配指令的方式暂时解决了这个问题。而真正的原因第二天早上 review 的时候才发现。

备注：我应该准备一份针对 C 语言的 debug-skill，罗列 C 语言中常见的 bug（如越界写入和内存破坏等）debug 陷入循环时，尝试从这个清单里找原因。

## 总结

由于学校课业繁重，时间紧凑，故本次实验笔者只完成了恰好足够通过测试的若干条指令，剩下的没有实现。目前已知的代码细节问题也很多，只不过不干扰测试，暂时就不动了。

这次实验总计花费 15 个小时，历时一天半，主要是对于平时积累的一个输出。在这之前，笔者已经学习了 QEMU 开发者文档的 QOM 部分，跟着 ai 逐行 review 了两个简单设备的源码，了解了 GPU 的架构和并行计算的方法，学了一点 CUDA 编程，因此实验完成得相当顺利。

## 参考资料

- *GPGPU 虚拟加速器实验手册*，QEMU Camp 2026. [在线链接](https://qemu.gevico.online/exercise/2026/stage2/gpu/gpu-exper-manual/)
- *GPGPU 虚拟加速器硬件手册（教学版）*，QEMU Camp 2026. [在线链接](https://qemu.gevico.online/exercise/2026/stage2/gpu/gpu-datasheet/)
- *NEMU (NJU Emulator) 官方文档*，南京大学计算机系统基础实验 (PA). [在线链接](https://ysyx.oscc.cc/docs/ics-pa/)  
- *QEMU Developer Information: QOM (QEMU Object Model)*. [在线链接](https://www.qemu.org/docs/master/devel/)
- *The RISC-V Instruction Set Manual, Volume I: Unprivileged ISA*. [在线链接](https://riscv.org/technical/specifications/)
