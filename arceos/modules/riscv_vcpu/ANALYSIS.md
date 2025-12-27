# riscv_vcpu 模块代码分析文档

## 目录

1. [模块概述](#模块概述)
2. [项目结构](#项目结构)
3. [核心数据结构](#核心数据结构)
4. [CSR 管理系统](#csr-管理系统)
5. [VMExit 处理机制](#vmexit-处理机制)
6. [SBI 调用转发](#sbi-调用转发)
7. [世界切换流程](#世界切换流程)
8. [关键技术点](#关键技术点)
9. [使用示例](#使用示例)
10. [总结](#总结)

---

## 模块概述

`riscv_vcpu` 是 ArceOS Hypervisor 的核心模块，提供了 RISC-V H 扩展的 vCPU 抽象和虚拟化相关接口支持。

### 主要功能

- **vCPU 状态管理**: 保存/恢复 Guest 和 Hypervisor 的寄存器状态
- **VMExit 处理**: 统一处理各种虚拟化退出事件
- **SBI 转发**: 将 Guest 的 SBI 调用转发到 OpenSBI 固件
- **中断虚拟化**: 模拟定时器中断等虚拟中断
- **CSR 管理**: 统一管理虚拟化相关的控制和状态寄存器

### 技术栈

| 依赖 | 说明 |
|------|------|
| **riscv** | RISC-V 寄存器访问 (rcore-os/riscv) |
| **riscv-decode** | RISC-V 指令解码 |
| **sbi-spec/sbi-rt** | RISC-V SBI 规范和运行时 |
| **tock-registers** | 类型安全的寄存器操作 |
| **axerrno** | ArceOS 错误处理 |
| **axhal** | ArceOS 硬件抽象层 |

### 架构定位

```
┌─────────────────────────────────────────────────────────────────┐
│                    Hypervisor (h_2_0/h_3_0/h_4_0)              │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  main.rs:                                                 │  │
│  │    - RISCVVCpu::init()                                   │  │
│  │    - vcpu.run() -> AxVCpuExitReason                       │  │
│  │    - 处理 NestedPageFault 等退出原因                     │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                            |
                            | 依赖
                            V
┌─────────────────────────────────────────────────────────────────┐
│                      riscv_vcpu 模块                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐│
│  │  vcpu.rs    │  │  csrs.rs    │  │  sbi/                   ││
│  │             │  │             │  │  - base.rs              ││
│  │ RISCVVCpu  │  │ CSR 抽象    │  │  - rfnc.rs              ││
│  │ VmCpuRegisters│ │            │  │  - pmu.rs               ││
│  │ VMExit 处理 │  │             │  │  - ...                  ││
│  └─────────────┘  └─────────────┘  └─────────────────────────┘│
│  ┌─────────────┐  ┌─────────────────────────────────────────┐ │
│  │  guest.S    │  │  regs.rs                                 │ │
│  │             │  │  GeneralPurposeRegisters                  │ │
│  │ 世界切换     │  │  GprIndex                                │ │
│  │ 汇编代码    │  │                                          │ │
│  └─────────────┘  └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                            |
                            | 调用
                            V
┌─────────────────────────────────────────────────────────────────┐
│                    OpenSBI (M-mode)                             │
│  - sbi_rt::set_timer()                                          │
│  - sbi_rt::remote_fence_i()                                     │
│  - ...                                                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 项目结构

```
riscv_vcpu/
├── Cargo.toml              # 项目配置
├── src/
│   ├── lib.rs              # 模块入口，导出公共 API
│   ├── vcpu.rs             # vCPU 核心实现 (582 行)
│   ├── csrs.rs             # CSR 管理和定义 (311 行)
│   ├── regs.rs             # 通用寄存器定义 (115 行)
│   ├── guest.S             # 世界切换汇编代码 (182 行)
│   ├── vmexit.rs           # VMExit 处理 (未实现)
│   ├── detect.rs           # H 扩展检测
│   └── sbi/
│       ├── mod.rs          # SBI 消息定义 (89 行)
│       ├── base.rs         # Base 扩展
│       ├── rfnc.rs         # Remote Fence 扩展
│       ├── pmu.rs          # PMU 扩展
│       ├── dbcn.rs         # Debug Console 扩展
│       └── srst.rs         # System Reset 扩展
└── README.md
```

### 文件说明

| 文件 | 行数 | 功能 |
|------|------|------|
| [vcpu.rs](src/vcpu.rs) | 582 | vCPU 实现，VMExit 处理 |
| [csrs.rs](src/csrs.rs) | 311 | CSR 寄存器定义和抽象 |
| [guest.S](src/guest.S) | 182 | 世界切换汇编代码 |
| [regs.rs](src/regs.rs) | 115 | GPR 定义 |
| [sbi/mod.rs](src/sbi/mod.rs) | 89 | SBI 消息定义 |

---

## 核心数据结构

### 1. VmCpuRegisters - vCPU 完整状态

```rust
#[derive(Default)]
#[repr(C)]
pub struct VmCpuRegisters {
    // Hypervisor 和 Guest 共享的寄存器状态
    hyp_regs: HypervisorCpuState,
    pub guest_regs: GuestCpuState,

    // 仅在 V=1 时有效的 VS-level CSR
    vs_csrs: GuestVsCsrs,

    // 虚拟化的 HS-level CSR
    virtual_hs_csrs: GuestVirtualHsCsrs,

    // VMExit 时读取的 CSR
    pub trap_csrs: VmCpuTrapState,
}
```

#### 结构详解

```
VmCpuRegisters 内存布局:
┌─────────────────────────────────────────────────────────────────┐
│  HypervisorCpuState      (保存 Hypervisor 执行状态)            │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  gprs: [ra, gp, tp, s0-s11, a1-a7, sp]                    │  │
│  │  sstatus, scounteren, stvec, sscratch                     │  │
│  └───────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│  GuestCpuState           (保存 Guest 执行状态)                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  gprs: [x0-x31]                                           │  │
│  │  sstatus, hstatus, scounteren, sepc                      │  │
│  └───────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│  GuestVsCsrs             (VS-mode 专用 CSR, V=1 时有效)       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  htimedelta, vsstatus, vsie, vstvec, vsscratch,          │  │
│  │  vsepc, vscause, vstval, vsatp, vstimecmp                │  │
│  └───────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│  GuestVirtualHsCsrs      (虚拟化的 HS-level CSR)              │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  hie, hgeie, hgatp                                       │  │
│  └───────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│  VmCpuTrapState          (VMExit 时读取的 Trap 信息)          │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  scause, stval, htval, htinst                            │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 2. HypervisorCpuState - Hypervisor 状态

```rust
#[derive(Default)]
#[repr(C)]
struct HypervisorCpuState {
    gprs: GeneralPurposeRegisters,  // 除 t0-t6 外的 GPR
    sstatus: usize,
    scounteren: usize,
    stvec: usize,
    sscratch: usize,
}
```

**为什么只保存部分 GPR?**

在 `guest.S` 的汇编实现中：
- **t0-t6**: 临时寄存器，用作世界切换的"暂存寄存器"
- **a0**: 用作传递 `VmCpuRegisters` 指针，保存在 `sscratch` 中

### 3. GuestCpuState - Guest 状态

```rust
#[derive(Default)]
#[repr(C)]
pub struct GuestCpuState {
    pub gprs: GeneralPurposeRegisters,  // 全部 32 个 GPR
    pub sstatus: usize,
    pub hstatus: usize,
    pub scounteren: usize,
    pub sepc: usize,
}
```

### 4. GeneralPurposeRegisters - GPR 管理

```rust
#[derive(Default)]
#[repr(C)]
pub struct GeneralPurposeRegisters([usize; 32]);

#[repr(u32)]
#[derive(Clone, Copy, Debug, Eq, PartialEq)]
pub enum GprIndex {
    Zero = 0,  RA,  SP,  GP,  TP,
    T0,  T1,  T2,  S0,  S1,
    A0,  A1,  A2,  A3,  A4,  A5,  A6,  A7,
    S2,  S3,  S4,  S5,  S6,  S7,  S8,  S9,  S10, S11,
    T3,  T4,  T5,  T6,
}
```

#### 常用方法

```rust
impl GeneralPurposeRegisters {
    // 获取单个寄存器
    pub fn reg(&self, reg_index: GprIndex) -> usize;

    // 设置单个寄存器
    pub fn set_reg(&mut self, reg_index: GprIndex, val: usize);

    // 获取参数寄存器 a0-a7 (用于 SBI 调用)
    pub fn a_regs(&self) -> &[usize];

    // 可变获取参数寄存器
    pub fn a_regs_mut(&mut self) -> &mut [usize];
}
```

### 5. RISCVVCpu - vCPU 实现

```rust
#[derive(Default)]
pub struct RISCVVCpu {
    regs: VmCpuRegisters,
}
```

---

## CSR 管理系统

### CSR 寄存器定义

```rust
pub mod defs {
    // Hypervisor Exception Delegation Register
    pub const CSR_HEDELEG: u16 = 0x602;

    // Hypervisor Interrupt Delegation Register
    pub const CSR_HIDELEG: u16 = 0x603;

    // Hypervisor Status Register
    pub const CSR_HSTATUS: u16 = 0x600;

    // Hypervisor Virtual Interrupt Pending Register
    pub const CSR_HVIP: u16 = 0x645;

    // Hypervisor Guest Address Translation and Protection
    pub const CSR_HGATP: u16 = 0x680;

    // VS-mode CSRs (当 V=1 时有效)
    pub const CSR_VSSTATUS: u16 = 0x200;
    pub const CSR_VSATP: u16 = 0x280;
    // ...
}
```

### CSR 抽象接口

```rust
/// RISC-V CSR 操作 Trait
pub trait RiscvCsrTrait {
    type R: RegisterLongName;

    // 读取 CSR
    fn get_value(&self) -> usize;

    // 写入 CSR
    fn write_value(&self, value: usize);

    // 原子替换
    fn atomic_replace(&self, value: usize) -> usize;

    // 读并置位
    fn read_and_set_bits(&self, bitmasks: usize) -> usize;

    // 读并清零位
    fn read_and_clear_bits(&self, bitmasks: usize) -> usize;
}
```

### CSR 寄存器实例

```rust
pub struct CSR {
    pub sie: ReadWriteCsr<sie::Register, CSR_SIE>,
    pub hstatus: ReadWriteCsr<hstatus::Register, CSR_HSTATUS>,
    pub hedeleg: ReadWriteCsr<hedeleg::Register, CSR_HEDELEG>,
    pub hideleg: ReadWriteCsr<hideleg::Register, CSR_HIDELEG>,
    pub hcounteren: ReadWriteCsr<hcounteren::Register, CSR_HCOUNTEREN>,
    pub hvip: ReadWriteCsr<hvip::Register, CSR_HVIP>,
}

// 全局静态实例
pub const CSR: &CSR = &CSR {
    sie: ReadWriteCsr::new(),
    hstatus: ReadWriteCsr::new(),
    // ...
};
```

### CSR 位字段定义

#### hstatus (Hypervisor Status Register)

```rust
register_bitfields![usize,
pub hstatus [
    vsbe     OFFSET(6) NUMBITS(1) [],      // VS 模式字节序
    gva      OFFSET(6) NUMBITS(1) [],      // GVA 写入 stval
    spv      OFFSET(7) NUMBITS(1) [        // 陷阱时的虚拟化模式
        User = 0,
        Supervisor = 1,
    ],
    spvp     OFFSET(8) NUMBITS(1) [        // 陷阱前的特权级
        User = 0,
        Supervisor = 1,
    ],
    hu       OFFSET(9) NUMBITS(1) [],      // U 模式允许 hypervisor 指令
    vgein    OFFSET(12) NUMBITS(6) [],     // 外部中断源选择
    vtvm     OFFSET(20) NUMBITS(1) [],     // 陷阱 SFENCE/SINVAL/vsatp
    vtw      OFFSET(21) NUMBITS(1) [],     // 陷阱 WFI 超时
    vtsr     OFFSET(22) NUMBITS(1) [],     // 陷阱 SRET
    vsxl     OFFSET(32) NUMBITS(2) [       // VS 模式 ISA 宽度
        Xlen32 = 1,
        Xlen64 = 2,
    ],
]];
```

#### hideleg (Hypervisor Interrupt Delegation Register)

```rust
register_bitfields![usize,
pub hideleg [
    vssoft   OFFSET(2) NUMBITS(1) [],      // 委托虚拟软中断
    vstimer  OFFSET(6) NUMBITS(1) [],      // 委托虚拟定时器中断
    vsext    OFFSET(10) NUMBITS(1) [],     // 委托虚拟外部中断
]];
```

#### hvip (Hypervisor Virtual Interrupt Pending)

```rust
register_bitfields![usize,
pub hvip [
    vssoft   OFFSET(2) NUMBITS(1) [],      // 虚拟软中断挂起
    vstimer  OFFSET(6) NUMBITS(1) [],      // 虚拟定时器中断挂起
    vsext    OFFSET(10) NUMBITS(1) [],     // 虚拟外部中断挂起
]];
```

---

## VMExit 处理机制

### VMExit 处理流程

```
Guest 执行
    |
    | 触发 VMExit (异常/中断/特殊指令)
    V
guest.S: _guest_exit
    |
    | 保存 Guest GPR
    V
guest.S: _restore_csrs
    |
    | 恢复 Hypervisor CSR/GPR
    V
返回 Rust: _run_guest() -> vmexit_handler()
    |
    | 读取 trap_csrs (scause, stval, htval, htinst)
    V
vmexit_handler() - 根据 scause 分发
    |
    +-> VirtualSupervisorEnvCall -> SBI 调用处理
    +-> SupervisorTimer          -> 定时器中断处理
    +-> SupervisorExternal       -> 外部中断
    +-> LoadGuestPageFault       -> 缺页异常
    +-> StoreGuestPageFault      -> 缺页异常
    +-> ...                      -> panic!
    V
返回 AxVCpuExitReason 给 Hypervisor
```

### VMExit 原因分类

```rust
pub enum AxVCpuExitReason {
    // SBI 调用 (内部处理，返回 Nothing)
    Hypercall { nr: u64, args: [u64; 6] },

    // MMIO 读写 (未实现)
    MmioRead { addr: GuestPhysAddr, width: AccessWidth, reg: usize, reg_width: AccessWidth },
    MmioWrite { addr: GuestPhysAddr, width: AccessWidth, data: u64 },

    // I/O 读写 (x86 特有，RISC-V 不使用)
    IoRead { port: Port, width: AccessWidth },
    IoWrite { port: Port, width: AccessWidth, data: u64 },

    // 外部中断
    ExternalInterrupt { vector: u64 },

    // 缺页异常 (EPT violation)
    NestedPageFault { addr: GuestPhysAddr, access_flags: MappingFlags },

    // vCPU 状态
    Halt,
    CpuDown,
    SystemDown,

    // 内部已处理
    Nothing,

    // VMEntry 失败
    FailEntry { hardware_entry_failure_reason: u64 },
}
```

### 核心处理逻辑 ([vcpu.rs:291-374](src/vcpu.rs))

```rust
fn vmexit_handler(&mut self) -> AxResult<AxVCpuExitReason> {
    // 1. 读取 Trap 相关 CSR
    self.regs.trap_csrs.scause = scause::read().bits();
    self.regs.trap_csrs.stval = stval::read();
    self.regs.trap_csrs.htval = htval::read();
    self.regs.trap_csrs.htinst = htinst::read();

    let scause = scause::read();
    match scause.cause() {
        // 2. SBI 调用
        Trap::Exception(Exception::VirtualSupervisorEnvCall) => {
            let sbi_msg = SbiMessage::from_regs(self.regs.guest_regs.gprs.a_regs())?;
            match sbi_msg {
                SbiMessage::Base(base) => self.handle_base_function(base)?,
                SbiMessage::SetTimer(timer) => { /* ... */ },
                SbiMessage::RemoteFence(rfnc) => self.handle_rfnc_function(rfnc)?,
                SbiMessage::PMU(pmu) => self.handle_pmu_function(pmu)?,
                _ => todo!(),
            }
            self.advance_pc(4);
            Ok(AxVCpuExitReason::Nothing)
        }

        // 3. 定时器中断
        Trap::Interrupt(Interrupt::SupervisorTimer) => {
            // 设置虚拟定时器中断
            CSR.hvip.read_and_set_bits(traps::interrupt::VIRTUAL_SUPERVISOR_TIMER);
            // 清除 Host 定时器中断
            CSR.sie.read_and_clear_bits(traps::interrupt::SUPERVISOR_TIMER);
            Ok(AxVCpuExitReason::Nothing)
        }

        // 4. 外部中断
        Trap::Interrupt(Interrupt::SupervisorExternal) => {
            Ok(AxVCpuExitReason::ExternalInterrupt { vector: 0 })
        }

        // 5. 缺页异常
        Trap::Exception(Exception::LoadGuestPageFault)
        | Trap::Exception(Exception::StoreGuestPageFault) => {
            let fault_addr = self.regs.trap_csrs.htval << 2 | self.regs.trap_csrs.stval & 0x3;
            Ok(AxVCpuExitReason::NestedPageFault {
                addr: GuestPhysAddr::from(fault_addr),
                access_flags: MappingFlags::empty(),
            })
        }

        // 6. 未处理的 Trap
        _ => panic!("Unhandled trap: {:?}", scause.cause()),
    }
}
```

---

## SBI 调用转发

### SBI 消息定义

```rust
#[derive(Clone, Copy, Debug)]
pub enum SbiMessage {
    // Base 扩展 (获取 SBI 版本、实现 ID 等)
    Base(BaseFunction),

    // Legacy 扩展
    GetChar,                    // 控制台读取
    PutChar(usize),             // 控制台写入
    SetTimer(usize),            // 设置定时器

    // Time 扩展
    SetTimer(usize),

    // Debug Console 扩展
    DebugConsole(DebugConsoleFunction),

    // System Reset 扩展
    Reset(ResetFunction),

    // Remote Fence 扩展
    RemoteFence(RemoteFenceFunction),

    // PMU 扩展
    PMU(PmuFunction),
}
```

### SBI 消息解析

```rust
impl SbiMessage {
    /// 从 GPR 解析 SBI 调用
    /// a7: Extension ID
    /// a0-a6: 参数
    pub fn from_regs(args: &[usize]) -> AxResult<Self> {
        match args[7] {
            sbi_spec::base::EID_BASE => {
                BaseFunction::from_regs(args).map(SbiMessage::Base)
            }
            sbi_spec::legacy::LEGACY_SET_TIMER => {
                Ok(SbiMessage::SetTimer(args[0]))
            }
            sbi_spec::rfnc::EID_RFNC => {
                RemoteFenceFunction::from_args(args).map(SbiMessage::RemoteFence)
            }
            sbi_spec::pmu::EID_PMU => {
                PmuFunction::from_regs(args).map(SbiMessage::PMU)
            }
            _ => Err(AxError::NotFound),
        }
    }
}
```

### SBI 处理函数

#### 1. Base 扩展 ([vcpu.rs:376-413](src/vcpu.rs))

```rust
fn handle_base_function(&mut self, base: BaseFunction) -> AxResult<()> {
    match base {
        BaseFunction::GetSepcificationVersion => {
            let version = sbi_rt::get_spec_version();
            // 返回: a1 = (major << 24) | minor
            self.set_gpr_from_gpr_index(GprIndex::A1, version.major() << 24 | version.minor());
        }
        BaseFunction::GetImplementationID => {
            let id = sbi_rt::get_sbi_impl_id();
            self.set_gpr_from_gpr_index(GprIndex::A1, id);
        }
        BaseFunction::ProbeSbiExtension(extension) => {
            let extension = sbi_rt::probe_extension(extension as usize).raw;
            self.set_gpr_from_gpr_index(GprIndex::A1, extension);
        }
        // ...
    }
    // a0 = 0 表示成功
    self.set_gpr_from_gpr_index(GprIndex::A0, 0);
    Ok(())
}
```

#### 2. Remote Fence 扩展 ([vcpu.rs:415-443](src/vcpu.rs))

```rust
fn handle_rfnc_function(&mut self, rfnc: RemoteFenceFunction) -> AxResult<()> {
    self.set_gpr_from_gpr_index(GprIndex::A0, 0);
    match rfnc {
        RemoteFenceFunction::FenceI { hart_mask, hart_mask_base } => {
            let sbi_ret = sbi_rt::remote_fence_i(hart_mask as usize, hart_mask_base as usize);
            self.set_gpr_from_gpr_index(GprIndex::A0, sbi_ret.error);
            self.set_gpr_from_gpr_index(GprIndex::A1, sbi_ret.value);
        }
        RemoteFenceFunction::RemoteSFenceVMA { hart_mask, hart_mask_base, start_addr, size } => {
            let sbi_ret = sbi_rt::remote_sfence_vma(
                hart_mask as usize,
                hart_mask_base as usize,
                start_addr as usize,
                size as usize,
            );
            self.set_gpr_from_gpr_index(GprIndex::A0, sbi_ret.error);
            self.set_gpr_from_gpr_index(GprIndex::A1, sbi_ret.value);
        }
    }
    Ok(())
}
```

#### 3. PMU 扩展 ([vcpu.rs:445-471](src/vcpu.rs))

```rust
fn handle_pmu_function(&mut self, pmu: PmuFunction) -> AxResult<()> {
    self.set_gpr_from_gpr_index(GprIndex::A0, 0);
    match pmu {
        PmuFunction::GetNumCounters => {
            self.set_gpr_from_gpr_index(GprIndex::A1, sbi_rt::pmu_num_counters())
        }
        PmuFunction::GetCounterInfo(counter_index) => {
            let sbi_ret = pmu_counter_get_info(counter_index as usize);
            self.set_gpr_from_gpr_index(GprIndex::A0, sbi_ret.error);
            self.set_gpr_from_gpr_index(GprIndex::A1, sbi_ret.value);
        }
        PmuFunction::StopCounter { counter_index, counter_mask, stop_flags } => {
            let sbi_ret = pmu_counter_stop(
                counter_index as usize,
                counter_mask as usize,
                stop_flags as usize,
            );
            self.set_gpr_from_gpr_index(GprIndex::A0, sbi_ret.error);
            self.set_gpr_from_gpr_index(GprIndex::A1, sbi_ret.value);
        }
    }
    Ok(())
}
```

---

## 世界切换流程

### 汇编入口: _run_guest ([guest.S:4-89](src/guest.S))

```assembly
.global _run_guest
_run_guest:
    // ─────────────────────────────────────────────────────────────
    // 第一阶段: 保存 Hypervisor 状态
    // ─────────────────────────────────────────────────────────────
    // a0: VmCpuRegisters 指针

    // 保存 Hypervisor GPR (除了 t0-t6 和 a0)
    sd   ra, ({hyp_ra})(a0)
    sd   gp, ({hyp_gp})(a0)
    // ... 省略其他寄存器
    sd   sp, ({hyp_sp})(a0)

    // 交换 Hypervisor 和 Guest 的 CSR
    ld    t1, ({guest_sstatus})(a0)
    csrrw t1, sstatus, t1          // 原子交换，并保存旧的 sstatus
    sd    t1, ({hyp_sstatus})(a0)

    ld    t1, ({guest_hstatus})(a0)
    csrrw t1, hstatus, t1

    // ... 交换 scounteren, sepc

    // 设置 stvec 指向退出处理代码
    la    t1, _guest_exit
    csrrw t1, stvec, t1
    sd    t1, ({hyp_stvec})(a0)

    // 保存 sscratch，并替换为 VmCpuRegisters 指针
    csrrw t1, sscratch, a0         // a0 -> sscratch, 旧 sscratch -> t1
    sd    t1, ({hyp_sscratch})(a0)

    // ─────────────────────────────────────────────────────────────
    // 第二阶段: 恢复 Guest 状态
    // ─────────────────────────────────────────────────────────────
    // 恢复 Guest GPR (从 sscratch 指向的 VmCpuRegisters)
    ld   ra, ({guest_ra})(a0)
    ld   gp, ({guest_gp})(a0)
    // ... 恢复所有 GPR
    ld   a0, ({guest_a0})(a0)      // 最后恢复 a0 (从 sscratch)

    sret                           // 返回 Guest (VS-mode)
```

### 退出处理: _guest_exit ([guest.S:92-181](src/guest.S))

```assembly
.align 2
_guest_exit:
    // ─────────────────────────────────────────────────────────────
    // 第一阶段: 保存 Guest 状态
    // ─────────────────────────────────────────────────────────────
    // 从 sscratch 恢复 VmCpuRegisters 指针，同时保存 Guest a0
    csrrw a0, sscratch, a0         // Guest a0 -> sscratch, VmCpuRegisters* -> a0

    // 保存 Guest GPR
    sd   ra, ({guest_ra})(a0)
    sd   gp, ({guest_gp})(a0)
    // ... 省略其他寄存器
    sd   t6, ({guest_t6})(a0)
    sd   sp, ({guest_sp})(a0)

    // 保存 Guest a0 (从 sscratch)
    csrr  t0, sscratch
    sd    t0, ({guest_a0})(a0)

    // ─────────────────────────────────────────────────────────────
    // 第二阶段: 恢复 Hypervisor 状态
    // ─────────────────────────────────────────────────────────────
_restore_csrs:
    // 交换回 Hypervisor CSR
    ld    t1, ({hyp_sstatus})(a0)
    csrrw t1, sstatus, t1          // 原子交换，并保存旧的 sstatus
    sd    t1, ({guest_sstatus})(a0)

    csrr  t1, hstatus
    sd    t1, ({guest_hstatus})(a0)

    // ... 恢复 scounteren, stvec, sscratch

    // 保存 Guest EPC
    csrr  t1, sepc
    sd    t1, ({guest_sepc})(a0)

    // 恢复 Hypervisor GPR
    ld   ra, ({hyp_ra})(a0)
    ld   gp, ({hyp_gp})(a0)
    // ... 省略其他寄存器
    ld   sp, ({hyp_sp})(a0)

    ret                            // 返回 Rust 代码
```

### 世界切换的 CSR 变化

```
┌─────────────────────────────────────────────────────────────────┐
│                     进入 Guest 前                                │
├─────────────────────────────────────────────────────────────────┤
│  CSR              │ 保存到                    │ 恢复为           │
│  ─────────────────┼──────────────────────────┼─────────────────│
│  sstatus          │ hyp_regs.sstatus         │ guest_regs.sstatus
│  hstatus          │ (不保存)                 │ guest_regs.hstatus
│  scounteren       │ hyp_regs.scounteren      │ guest_regs.scounteren
│  stvec            │ hyp_regs.stvec           │ _guest_exit       │
│  sscratch         │ hyp_regs.sscratch        │ VmCpuRegisters*   │
│  sepc             │ (不保存)                 │ guest_regs.sepc   │
└─────────────────────────────────────────────────────────────────┘
                            |
                            | sret -> VS-mode
                            V
┌─────────────────────────────────────────────────────────────────┐
│                     Guest 执行 (VS-mode)                         │
│                                                                 │
│  hstatus.SPV = 1  (表示在虚拟化模式)                             │
│  hstatus.SPVP = S (表示之前在 Supervisor 模式)                   │
│  sstatus.SPP = S (sret 后进入 VS-mode, 非 VU-mode)               │
└─────────────────────────────────────────────────────────────────┘
                            |
                            | VMExit (异常/中断)
                            V
┌─────────────────────────────────────────────────────────────────┐
│                     退出 Guest 后                                │
├─────────────────────────────────────────────────────────────────┤
│  CSR              │ 保存到                    │ 恢复为           │
│  ─────────────────┼──────────────────────────┼─────────────────│
│  sstatus          │ guest_regs.sstatus       │ hyp_regs.sstatus
│  hstatus          │ guest_regs.hstatus       │ 硬件自动清除 SPV  │
│  scounteren       │ guest_regs.scounteren    │ hyp_regs.scounteren
│  stvec            │ _guest_exit              │ hyp_regs.stvec    │
│  sscratch         │ VmCpuRegisters*          │ hyp_regs.sscratch │
│  sepc             │ guest_regs.sepc          │ (不恢复)         │
└─────────────────────────────────────────────────────────────────┘
```

### 寄存器偏移量计算

```rust
// 汇编代码使用的偏移量常量
global_asm!(
    include_str!("guest.S"),
    // Hypervisor GPR 偏移
    hyp_ra = const hyp_gpr_offset(GprIndex::RA),
    hyp_sp = const hyp_gpr_offset(GprIndex::SP),
    // ...

    // Guest GPR 偏移
    guest_ra = const guest_gpr_offset(GprIndex::RA),
    guest_sp = const guest_gpr_offset(GprIndex::SP),
    // ...

    // Hypervisor CSR 偏移
    hyp_sstatus = const hyp_csr_offset!(sstatus),
    hyp_stvec = const hyp_csr_offset!(stvec),
    // ...

    // Guest CSR 偏移
    guest_sstatus = const guest_csr_offset!(sstatus),
    guest_hstatus = const guest_csr_offset!(hstatus),
    // ...
);

#[allow(dead_code)]
const fn hyp_gpr_offset(index: GprIndex) -> usize {
    offset_of!(VmCpuRegisters, hyp_regs)
        + offset_of!(HypervisorCpuState, gprs)
        + (index as usize) * size_of::<u64>()
}

#[allow(unused_macros)]
macro_rules! hyp_csr_offset {
    ($reg:tt) => {
        offset_of!(VmCpuRegisters, hyp_regs) + offset_of!(HypervisorCpuState, $reg)
    };
}
```

---

## 关键技术点

### 1. hstatus.SPV 的作用

在 `RISCVVCpu::init()` 中设置 ([vcpu.rs:247-267](src/vcpu.rs)):

```rust
pub fn init() -> Self {
    let mut regs = VmCpuRegisters::default();

    // 设置 hstatus
    let mut hstatus = LocalRegisterCopy::<usize, hstatus::Register>::new(
        riscv::register::hstatus::read().bits(),
    );
    hstatus.modify(hstatus::spv::Supervisor);  // SPV = 1
    hstatus.modify(hstatus::spvp::Supervisor); // SPVP = 1
    CSR.hstatus.write_value(hstatus.get());
    regs.guest_regs.hstatus = hstatus.get();

    // 设置 sstatus
    let mut sstatus = sstatus::read();
    sstatus.set_spp(sstatus::SPP::Supervisor);  // SPP = S
    regs.guest_regs.sstatus = sstatus.bits();

    Self { regs }
}
```

**SPV (Previous Virtualization)**:
- `SPV = 1`: `sret` 后进入 **VS-mode** (Guest 模式)
- `SPV = 0`: `sret` 后进入 **S-mode** (Hypervisor 模式)

**SPVP (Previous Privilege Mode)**:
- `SPVP = S`: 表示 Guest 在 VS-mode 执行
- 用于从 HS-mode 访问 VS-mode 的内存

### 2. 虚拟定时器中断注入

```rust
// Guest 设置定时器
SbiMessage::SetTimer(timer) => {
    // 1. 调用 OpenSBI 设置 Host 定时器
    sbi_rt::set_timer(timer as u64);

    // 2. 清除 Guest 虚拟定时器中断
    CSR.hvip.read_and_clear_bits(traps::interrupt::VIRTUAL_SUPERVISOR_TIMER);

    // 3. 使能 Host 定时器中断
    CSR.sie.read_and_set_bits(traps::interrupt::SUPERVISOR_TIMER);
}

// Host 定时器到期
Trap::Interrupt(Interrupt::SupervisorTimer) => {
    // 1. 设置 Guest 虚拟定时器中断挂起
    CSR.hvip.read_and_set_bits(traps::interrupt::VIRTUAL_SUPERVISOR_TIMER);

    // 2. 清除 Host 定时器中断
    CSR.sie.read_and_clear_bits(traps::interrupt::SUPERVISOR_TIMER);

    // 3. 返回 Guest 后，硬件自动注入 VSTIP
    Ok(AxVCpuExitReason::Nothing)
}
```

### 3. G-Stage 页表设置

```rust
pub fn set_ept_root(&mut self, ept_root: HostPhysAddr) -> AxResult {
    // hgatp 格式:
    // [63:60] MODE = 8 (SV39x4)
    // [59:44]  保留
    // [43:12]  PPN (物理页号)
    // [11:0]   保留
    self.regs.virtual_hs_csrs.hgatp = 8usize << 60 | usize::from(ept_root) >> 12;

    unsafe {
        core::arch::asm!(
            "csrw hgatp, {hgatp}",
            hgatp = in(reg) self.regs.virtual_hs_csrs.hgatp,
        );
        // 刷新 G-Stage TLB
        core::arch::riscv64::hfence_gvma_all();
    }
    Ok(())
}
```

### 4. 中断委托 (hideleg)

在 `setup_csrs()` 中设置 ([lib.rs:25-61](src/lib.rs)):

```rust
pub unsafe fn setup_csrs() {
    // 委托部分异常给 Guest
    CSR.hedeleg.write_value(
        traps::exception::INST_ADDR_MISALIGN
            | traps::exception::BREAKPOINT
            | traps::exception::ENV_CALL_FROM_U_OR_VU
            | traps::exception::INST_PAGE_FAULT
            | traps::exception::LOAD_PAGE_FAULT
            | traps::exception::STORE_PAGE_FAULT
            | traps::exception::ILLEGAL_INST,
    );

    // 委托所有中断给 Guest
    CSR.hideleg.write_value(
        traps::interrupt::VIRTUAL_SUPERVISOR_TIMER
            | traps::interrupt::VIRTUAL_SUPERVISOR_EXTERNAL
            | traps::interrupt::VIRTUAL_SUPERVISOR_SOFT,
    );

    // 清除所有挂起的虚拟中断
    CSR.hvip.read_and_clear_bits(
        traps::interrupt::VIRTUAL_SUPERVISOR_TIMER
            | traps::interrupt::VIRTUAL_SUPERVISOR_EXTERNAL
            | traps::interrupt::VIRTUAL_SUPERVISOR_SOFT,
    );

    // 使能所有计数器访问
    CSR.hcounteren.write_value(0xffff_ffff);

    // 使能 Host 中断
    CSR.sie.write_value(
        traps::interrupt::SUPERVISOR_EXTERNAL
            | traps::interrupt::SUPERVISOR_SOFT
            | traps::interrupt::SUPERVISOR_TIMER,
    );
}
```

### 5. 缺页异常地址计算

```rust
Trap::Exception(Exception::LoadGuestPageFault)
| Trap::Exception(Exception::StoreGuestPageFault) => {
    // htval: 高位转换位 (bits [43:2] << 2)
    // stval: 低位转换位 (bits [1:0])
    let fault_addr = self.regs.trap_csrs.htval << 2 | self.regs.trap_csrs.stval & 0x3;
    Ok(AxVCpuExitReason::NestedPageFault {
        addr: GuestPhysAddr::from(fault_addr),
        access_flags: MappingFlags::empty(),
    })
}
```

---

## 使用示例

### Hypervisor 使用 riscv_vcpu

```rust
use riscv_vcpu::{RISCVVCpu, setup_csrs};

fn main() {
    // 1. 初始化 CSR
    unsafe {
        setup_csrs();
    }

    // 2. 创建 vCPU
    let mut vcpu = RISCVVCpu::init();

    // 3. 设置 Guest 入口地址
    vcpu.set_entry(0x8020_0000.into()).unwrap();

    // 4. 设置 G-Stage 页表根
    let page_table_root: PhysAddr = ...;
    vcpu.set_ept_root(page_table_root).unwrap();

    // 5. 运行 Guest 循环
    loop {
        match vcpu.run() {
            Ok(exit_reason) => match exit_reason {
                AxVCpuExitReason::Nothing => {
                    // 内部已处理 (如 SBI 调用)
                }
                AxVCpuExitReason::NestedPageFault { addr, .. } => {
                    // 处理缺页异常
                    handle_mmio(addr);
                }
                AxVCpuExitReason::ExternalInterrupt { .. } => {
                    // 处理外部中断
                }
                _ => panic!("Unhandled exit: {:?}", exit_reason),
            },
            Err(err) => {
                panic!("vCPU run error: {:?}", err);
            }
        }
    }
}
```

### 设置 SBI 扩展

```rust
// Guest 调用 SBI
// a7 = EID_BASE (扩展 ID)
// a6 = GetSepcificationVersion (功能 ID)

// Hypervisor 自动处理:
// 1. vmexit_handler() 检测到 VirtualSupervisorEnvCall
// 2. SbiMessage::from_regs() 解析出 Base(GetSepcificationVersion)
// 3. handle_base_function() 调用 sbi_rt::get_spec_version()
// 4. 结果写入 a0, a1
// 5. advance_pc(4) 跳过 ecall 指令
// 6. 返回 Nothing (Hypervisor 无需处理)
```

---

## 总结

### 核心功能

1. **vCPU 抽象**: `RISCVVCpu` 提供完整的虚拟 CPU 抽象
2. **世界切换**: `guest.S` 实现高效的 Hypervisor <-> Guest 切换
3. **VMExit 处理**: 统一处理各种虚拟化退出事件
4. **SBI 转发**: 自动转发 Guest 的 SBI 调用到 OpenSBI
5. **中断虚拟化**: 支持定时器中断等虚拟中断注入

### 设计亮点

1. **类型安全**: 使用 `tock-registers` 提供类型安全的 CSR 操作
2. **模块化**: SBI 扩展独立模块，易于扩展
3. **零成本抽象**: 汇编代码直接使用编译时计算的偏移量
4. **Rust 集成**: 外部 Rust 代码可以直接调用，无需了解汇编细节

### 关键代码位置

| 文件 | 行号 | 功能 |
|------|------|------|
| [vcpu.rs](src/vcpu.rs:247-267) | 247-267 | vCPU 初始化 |
| [vcpu.rs](src/vcpu.rs:235-243) | 235-243 | vCPU 运行入口 |
| [vcpu.rs](src/vcpu.rs:291-374) | 291-374 | VMExit 处理核心 |
| [guest.S](src/guest.S:4-89) | 4-89 | 进入 Guest 汇编代码 |
| [guest.S](src/guest.S:92-181) | 92-181 | 退出 Guest 汇编代码 |
| [csrs.rs](src/csrs.rs:1-97) | 1-97 | CSR 抽象接口 |
| [lib.rs](src/lib.rs:25-61) | 25-61 | CSR 初始化 |
