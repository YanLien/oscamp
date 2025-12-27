# h_1_0 Hypervisor 代码分析文档

## 目录

1. [项目概述](#项目概述)
2. [项目结构](#项目结构)
3. [RISC-V H 扩展基础](#risc-v-h-扩展基础)
4. [核心模块分析](#核心模块分析)
5. [数据流分析](#数据流分析)
6. [关键技术点](#关键技术点)

---

## 项目概述

`h_1_0` 是基于 ArceOS 框架实现的 **RISC-V Hypervisor（虚拟机监视器）**，实现了 RISC-V H 扩展（H-extension）的核心功能。这是一个 Type-1 Hypervisor，直接运行在硬件之上，为 Guest 操作系统提供虚拟化环境。

### 主要特性

- **世界切换 (World Switch)**: HS-mode 与 VS-mode 之间的上下文切换
- **VMExit 处理**: 处理来自 Guest 的异常和 SBI 调用
- **两级地址转换**: G-Stage (Stage-2) 地址转换支持
- **SBI 调用转发**: 将 Guest 的 SBI 调用转发到底层固件

### 技术栈

| 组件 | 说明 |
|------|------|
| **ArceOS** | 模块化操作系统框架 |
| **RISC-V H 扩展** | 硬件虚拟化支持 |
| **SBI Specification** | Supervisor Binary Interface 标准 |
| **QEMU virt** | RISC-V 虚拟硬件平台 |

---

## 项目结构

```
h_1_0/
├── Cargo.toml              # 项目配置和依赖
├── src/
│   ├── main.rs             # 主入口，核心流程控制
│   ├── vcpu.rs             # vCPU 状态定义和汇编接口
│   ├── regs.rs             # 通用寄存器 (GPR) 定义
│   ├── csrs.rs             # 控制状态寄存器 (CSR) 抽象
│   ├── guest.S             # 世界切换的汇编实现
│   ├── loader.rs           # Guest 镜像加载器
│   ├── task.rs             # 任务扩展定义
│   └── sbi/                # SBI 调用处理模块
│       ├── mod.rs          # SBI 消息分发
│       ├── base.rs         # Base 扩展
│       ├── dbcn.rs         # Debug Console 扩展
│       ├── srst.rs         # System Reset 扩展
│       ├── rfnc.rs         # Remote Fence 扩展
│       └── pmu.rs          # Performance Monitor 扩展
├── h_1_0_riscv64-qemu-virt.elf   # ELF 可执行文件
└── h_1_0_riscv64-qemu-virt.bin   # 二进制镜像
```

---

## RISC-V H 扩展基础

### 特权级架构

RISC-V H 扩展引入了虚拟化支持，特权级从传统的 3 级扩展为 5 级：

```
┌─────────────────────────────────────────────────────┐
│                    M-mode (机器模式)                 │  ← 固件 (OpenSBI)
│                 Machine-level Software               │
├─────────────────────────────────────────────────────┤
│                   HS-mode (监管者模式)               │  ← Hypervisor (h_1_0)
│                  Hypervisor Software                 │
├─────────────────────────────────────────────────────┤
│                   VS-mode (虚拟监管者模式)           │  ← Guest OS
│                 Guest Supervisor Software            │
├─────────────────────────────────────────────────────┤
│                    VU-mode (虚拟用户模式)            │  ← Guest 应用
│                  Guest Application Software          │
├─────────────────────────────────────────────────────┤
│                     U-mode (用户模式)                │  ← 宿主应用 (可选)
│                  Application Software                │
└─────────────────────────────────────────────────────┘
```

### 两级地址转换

H 扩展引入了 **G-Stage (Stage-2) 地址转换**，实现 GPA (Guest Physical Address) 到 PA (Physical Address) 的映射：

```
Guest Virtual Address (GVA)
        │
        ▼
┌─────────────────────────────────────┐
│   VS-Stage (VSATP)                  │  ← Guest 控制
│   GVA → GPA 转换                    │
└─────────────────────────────────────┘
        │
        ▼
Guest Physical Address (GPA)
        │
        ▼
┌─────────────────────────────────────┐
│   G-Stage (HGATP)                   │  ← Hypervisor 控制
│   GPA → PA 转换                     │
└─────────────────────────────────────┘
        │
        ▼
Physical Address (PA)
```

### 关键 CSR 寄存器

| CSR | 名称 | 功能 |
|-----|------|------|
| `hgatp` | Hypervisor Guest Address Translation and Protection | G-Stage 页表基址 |
| `hstatus` | Hypervisor Status | 虚拟化状态控制 |
| `hedeleg` | Hypervisor Exception Delegation | 异常委托给 Guest |
| `hideleg` | Hypervisor Interrupt Delegation | 中断委托给 Guest |
| `vsstatus` | Virtual Supervisor Status | VS-mode 状态 |
| `vstvec` | Virtual Supervisor Trap Vector | VS-mode 异常处理入口 |
| `vsatp` | Virtual Supervisor Address Translation | VS-stage 页表基址 |

---

## 核心模块分析

### 1. 主流程 ([main.rs](arceos/tour/h_1_0/src/main.rs))

#### main 函数流程图

```
┌─────────────────────────────────────────────────────────────┐
│                          main()                             │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
        ┌───────────────────────────────────────┐
        │   ax_println!("Hypervisor ...")       │
        └───────────────────────────────────────┘
                            │
                            ▼
        ┌───────────────────────────────────────┐
        │   new_user_aspace()                   │  创建新的地址空间
        └───────────────────────────────────────┘
                            │
                            ▼
        ┌───────────────────────────────────────┐
        │   load_vm_image("/sbin/skernel")      │  加载 Guest 镜像
        └───────────────────────────────────────┘
                            │
                            ▼
        ┌───────────────────────────────────────┐
        │   prepare_guest_context(&mut ctx)     │  准备 Guest 上下文
        └───────────────────────────────────────┘
                            │
                            ▼
        ┌───────────────────────────────────────┐
        │  prepare_vm_pgtable(ept_root)         │  设置 G-Stage 页表
        └───────────────────────────────────────┘
                            │
                            ▼
        ┌───────────────────────────────────────┐
        │  run_guest(&mut ctx)                  │  进入 Guest
        └───────────────────────────────────────┘
                            │
                            ▼
        ┌───────────────────────────────────────┐
        │  vmexit_handler(ctx)                  │  处理 VMExit
        └───────────────────────────────────────┘
```

#### 准备 Guest 上下文

```rust
fn prepare_guest_context(ctx: &mut VmCpuRegisters) {
    // 1. 设置 hstatus
    //    - SPV = Guest: 返回时进入 Guest 模式
    //    - SPVP = Supervisor: 从 HS-mode 访问 VS-mode 内存
    let mut hstatus = LocalRegisterCopy::<usize, hstatus::Register>::new(
        riscv::register::hstatus::read().bits(),
    );
    hstatus.modify(hstatus::spv::Guest);
    hstatus.modify(hstatus::spvp::Supervisor);
    CSR.hstatus.write_value(hstatus.get());
    ctx.guest_regs.hstatus = hstatus.get();

    // 2. 设置 sstatus
    //    - SPP = Supervisor: Guest 以 S-mode 运行
    let mut sstatus = sstatus::read();
    sstatus.set_spp(sstatus::SPP::Supervisor);
    ctx.guest_regs.sstatus = sstatus.bits();

    // 3. 设置 Guest 入口地址
    ctx.guest_regs.sepc = VM_ENTRY;  // 0x8020_0000
}
```

#### G-Stage 页表设置

```rust
fn prepare_vm_pgtable(ept_root: PhysAddr) {
    // HGATP 格式:
    // | [63:60] | [59:44] | [43:0]          |
    // |   MODE  |  Reserved |   PPN         |
    //
    // MODE = 8: Sv39x4 (39-bit virtual, 4-level page table for guest)
    // PPN = ept_root >> 12: 页表根地址

    let hgatp = 8usize << 60 | usize::from(ept_root) >> 12;
    unsafe {
        core::arch::asm!("csrw hgatp, {hgatp}", hgatp = in(reg) hgatp);
        core::arch::riscv64::hfence_gvma_all();  // 刷新 G-Stage TLB
    }
}
```

#### VMExit 处理

```rust
fn vmexit_handler(ctx: &VmCpuRegisters) {
    let scause = scause::read();
    match scause.cause() {
        Trap::Exception(Exception::VirtualSupervisorEnvCall) => {
            // Guest 执行了 ecall，请求 SBI 服务
            let sbi_msg = SbiMessage::from_regs(ctx.guest_regs.gprs.a_regs()).ok();
            ax_println!("VmExit Reason: VSuperEcall: {:?}", sbi_msg);

            if let Some(msg) = sbi_msg {
                match msg {
                    SbiMessage::Reset(_) => {
                        ax_println!("Shutdown vm normally!");
                    }
                    _ => todo!(),
                }
            }
        }
        _ => {
            panic!(
                "Unhandled trap: {:?}, sepc: {:#x}, stval: {:#x}",
                scause.cause(),
                ctx.guest_regs.sepc,
                ctx.trap_csrs.stval
            );
        }
    }
}
```

### 2. vCPU 状态管理 ([vcpu.rs](arceos/tour/h_1_0/src/vcpu.rs))

#### VmCpuRegisters 结构

```
┌─────────────────────────────────────────────────────────────────────┐
│                         VmCpuRegisters                              │
├─────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                 HypervisorCpuState                           │   │
│  │  ┌─────────────────────────────────────────────────────┐    │   │
│  │  │  gprs: GeneralPurposeRegisters (32 × usize)          │    │   │
│  │  │  sstatus, scounteren, stvec, sscratch                │    │   │
│  │  └─────────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                 GuestCpuState                                │   │
│  │  ┌─────────────────────────────────────────────────────┐    │   │
│  │  │  gprs: GeneralPurposeRegisters (32 × usize)          │    │   │
│  │  │  sstatus, hstatus, scounteren, sepc                  │    │   │
│  │  └─────────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                 GuestVsCsrs (V=1 时有效)                    │   │
│  │  htimedelta, vsstatus, vsie, vstvec, vsscratch,            │   │
│  │  vsepc, vscause, vstval, vsatp, vstimecmp                  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                 GuestVirtualHsCsrs                          │   │
│  │  hie, hgeie, hgatp                                         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                 VmCpuTrapState                              │   │
│  │  scause, stval, htval, htinst                               │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### 3. 世界切换 ([guest.S](arceos/tour/h_1_0/src/guest.S))

#### _run_guest 汇编流程

```
▶ _run_guest(a0: *mut VmCpuRegisters)
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                     保存 Hypervisor 状态                         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  sd  ra, gp, tp, s0-s11, a1-a7, sp  → hyp_*            │   │
│  │  csrrw t1, sstatus, t1  → 交换并保存 hyp_sstatus        │   │
│  │  csrrw t1, hstatus, t1  → 加载 guest_hstatus            │   │
│  │  csrrw t1, scounteren, t1  → 交换并保存 hyp_scounteren  │   │
│  │  csrw sepc, t1  → 加载 guest_sepc                       │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                     设置 VMExit 处理入口                         │
│  la  t1, _guest_exit                                            │
│  csrrw t1, stvec, t1  → 设置 hyp_stvec                          │
└─────────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                     保存 sscratch，设置 GuestInfo 指针           │
│  csrrw t1, sscratch, a0  → 保存 hyp_sscratch，sscratch=a0      │
└─────────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                     恢复 Guest GPRs                              │
│  ld  ra, gp, tp, s0-s11, a1-a7, t0-t6, sp  ← guest_*          │
│  ld  a0 ← guest_a0                                              │
└─────────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                     sret  → 进入 Guest (VS-mode)                │
└─────────────────────────────────────────────────────────────────┘
                    ║
                    ║  Guest 执行... (发生异常/ecall)
                    ║
                    ▼
▶ _guest_exit  (自动跳转，因为 stvec 已设置)
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                     恢复 GuestInfo 指针                          │
│  csrrw a0, sscratch, a0  → sscratch=guest_a0, a0=GuestInfo      │
└─────────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                     保存 Guest GPRs                              │
│  sd  ra, gp, tp, s0-s11, a1-a7, t0-t6, sp  → guest_*           │
│  csrr t0, sscratch                                               │
│  sd  t0 → guest_a0                                               │
└─────────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                     恢复 Hypervisor CSRs                         │
│  csrrw t1, sstatus, t1  → 交换 hyp_sstatus，保存 guest_sstatus  │
│  csrr t1, hstatus  → 保存 guest_hstatus                         │
│  csrrw t1, scounteren, t1  → 交换 hyp_scounteren               │
│  csrw stvec, t1  → 恢复 hyp_stvec                               │
│  csrw sscratch, t1  → 恢复 hyp_sscratch                         │
│  csrr t1, sepc  → 保存 guest_sepc                               │
└─────────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                     恢复 Hypervisor GPRs                         │
│  ld  ra, gp, tp, s0-s11, a1-a7, sp  ← hyp_*                    │
└─────────────────────────────────────────────────────────────────┘
        │
        ▼
▶ ret  → 返回 Rust 代码 (vmexit_handler)
```

### 4. SBI 调用处理 ([sbi/mod.rs](arceos/tour/h_1_0/src/sbi/mod.rs))

#### SBI 消息定义

```rust
pub enum SbiMessage {
    // Base 扩展 (0x10)
    Base(BaseFunction),

    // Legacy 扩展
    GetChar,                // 0x01
    PutChar(usize),         // 0x01
    SetTimer(usize),        // 0x00
    Shutdown,               // 0x08

    // Time 扩展 (0x54494D45)
    SetTimer(usize),

    // Debug Console (0x4442434E)
    DebugConsole(DebugConsoleFunction),

    // System Reset (0x53525354)
    Reset(ResetFunction),

    // Remote Fence (0x52464E43)
    RemoteFence(RemoteFenceFunction),

    // PMU (0x504D55)
    PMU(PmuFunction),
}
```

#### 寄存器约定 (SBI 调用)

| 寄存器 | 用途 |
|--------|------|
| `a0` | 参数 / 返回值 |
| `a1` | 参数 / 返回值 |
| `a2` ~ `a5` | 参数 |
| `a6` | FID (Function ID) |
| `a7` | EID (Extension ID) |
| `a0` (返回) | error_code |
| `a1` (返回) | return_value |

#### SBI 消息解析

```rust
impl SbiMessage {
    pub fn from_regs(args: &[usize]) -> AxResult<Self> {
        match args[7] {  // a7 = EID
            sbi_spec::base::EID_BASE => {
                BaseFunction::from_regs(args).map(SbiMessage::Base)
            }
            sbi_spec::legacy::LEGACY_CONSOLE_PUTCHAR => {
                Ok(SbiMessage::PutChar(args[0]))
            }
            sbi_spec::srst::EID_SRST => {
                ResetFunction::from_regs(args).map(SbiMessage::Reset)
            }
            sbi_spec::rfnc::EID_RFNC => {
                RemoteFenceFunction::from_args(args).map(SbiMessage::RemoteFence)
            }
            sbi_spec::pmu::EID_PMU => {
                PmuFunction::from_regs(args).map(SbiMessage::PMU)
            }
            _ => Err(AxError::NotFound)
        }
    }
}
```

### 5. CSR 抽象 ([csrs.rs](arceos/tour/h_1_0/src/csrs.rs))

#### CSR 定义

```rust
pub struct CSR {
    pub sie: ReadWriteCsr<sie::Register, CSR_SIE>,
    pub hstatus: ReadWriteCsr<hstatus::Register, CSR_HSTATUS>,
    pub hedeleg: ReadWriteCsr<hedeleg::Register, CSR_HEDELEG>,
    pub hideleg: ReadWriteCsr<hideleg::Register, CSR_HIDELEG>,
    pub hcounteren: ReadWriteCsr<hcounteren::Register, CSR_HCOUNTEREN>,
    pub hvip: ReadWriteCsr<hvip::Register, CSR_HVIP>,
}

pub const CSR: &CSR = &CSR { /* ... */ };
```

#### 使用示例

```rust
// 读取 hstatus
let hstatus = CSR.hstatus.get();

// 修改 hstatus
let mut hstatus = LocalRegisterCopy::<usize, hstatus::Register>::new(
    CSR.hstatus.get()
);
hstatus.modify(hstatus::spv::Guest);
hstatus.modify(hstatus::spvp::Supervisor);

// 写入 hstatus
CSR.hstatus.write_value(hstatus.get());
```

#### hstatus 位段定义

| 位段 | 说明 |
|------|------|
| `vsbe` | VS-mode 字节序控制 |
| `gva` | stval 包含 Guest 虚拟地址 |
| `spv` | 虚拟化模式 (Host=0, Guest=1) |
| `spvp` | 陷阱前的特权级 (User=0, Supervisor=1) |
| `hu` | 允许在 U-mode 执行 hypervisor 指令 |
| `vgein` | Guest 外部中断编号 |
| `vtvm` | 陷阱 SFENCE/SINVAL/vsatp 变化 |
| `vtw` | 陷阱 WFI 超时 |
| `vtsr` | 陷阱 SRET 指令 |
| `vsxl` | VS-mode 原生 ISA 宽度 |

### 6. Guest 镜像加载 ([loader.rs](arceos/tour/h_1_0/src/loader.rs))

```rust
pub fn load_vm_image(fname: &str, uspace: &mut AddrSpace) -> io::Result<()> {
    // 1. 读取 Guest 镜像文件 (最多 64 字节)
    let mut buf = [0u8; 64];
    load_file(fname, &mut buf)?;

    // 2. 在 Guest 地址空间映射内存
    //    VM_ENTRY = 0x8020_0000
    //    权限: R|W|X|USER
    uspace.map_alloc(
        VM_ENTRY.into(),
        PAGE_SIZE_4K,
        MappingFlags::READ | MappingFlags::WRITE |
        MappingFlags::EXECUTE | MappingFlags::USER,
        true
    ).unwrap();

    // 3. 获取物理地址
    let (paddr, _, _) = uspace
        .page_table()
        .query(VM_ENTRY.into())
        .unwrap();

    // 4. 复制镜像到物理内存
    unsafe {
        core::ptr::copy_nonoverlapping(
            buf.as_ptr(),
            phys_to_virt(paddr).as_mut_ptr(),
            PAGE_SIZE_4K,
        );
    }

    Ok(())
}
```

---

## 数据流分析

### Guest 启动流程

```
┌─────────────────────────────────────────────────────────────────┐
│                        Hypervisor (HS-mode)                     │
└─────────────────────────────────────────────────────────────────┘
                            │
                            │ 1. 创建地址空间
                            │    new_user_aspace()
                            │
                            ▼
        ┌───────────────────────────────────────┐
        │  AddrSpace {                          │
        │    page_table_root: PhysAddr          │
        │  }                                    │
        └───────────────────────────────────────┘
                            │
                            │ 2. 加载 Guest 镜像
                            │    load_vm_image("/sbin/skernel")
                            │
                            ▼
        ┌───────────────────────────────────────┐
        │  Guest Code in Memory                │
        │  GPA: 0x8020_0000                    │
        │  PA:  (映射后)                       │
        └───────────────────────────────────────┘
                            │
                            │ 3. 设置 G-Stage 页表
                            │    prepare_vm_pgtable(ept_root)
                            │    hgatp = MODE | PPN
                            │
                            ▼
        ┌───────────────────────────────────────┐
        │  G-Stage Page Table                  │
        │  GPA 0x8020_0000 → PA xxx            │
        └───────────────────────────────────────┘
                            │
                            │ 4. 准备 vCPU 上下文
                            │    prepare_guest_context()
                            │
                            ▼
        ┌───────────────────────────────────────┐
        │  VmCpuRegisters                      │
        │  guest_regs.sepc = 0x8020_0000       │
        │  guest_regs.hstatus.SPV = Guest      │
        │  guest_regs.sstatus.SPP = Supervisor │
        └───────────────────────────────────────┘
                            │
                            │ 5. 世界切换
                            │    _run_guest()
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Guest (VS-mode)                          │
│  执行 Guest 代码，从 0x8020_0000 开始                            │
└─────────────────────────────────────────────────────────────────┘
```

### SBI 调用流程

```
┌─────────────────────────────────────────────────────────────────┐
│                        Guest (VS-mode)                          │
└─────────────────────────────────────────────────────────────────┘
                            │
                            │ Guest 执行 ecall 指令
                            │ a7 = EID (扩展 ID)
                            │ a6 = FID (功能 ID)
                            │ a0-a5 = 参数
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     VMExit (自动触发)                            │
│  scause.Exception.VirtualSupervisorEnvCall                      │
│  stvec 指向 _guest_exit                                          │
└─────────────────────────────────────────────────────────────────┘
                            │
                            │ 保存 Guest 上下文
                            │ 恢复 Hypervisor 上下文
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Hypervisor (HS-mode)                     │
│  vmexit_handler()                                               │
│    │                                                             │
│    │ 1. 解析 SBI 消息                                            │
│    │    SbiMessage::from_regs(ctx.guest_regs.gprs.a_regs())    │
│    │                                                             │
│    ▼                                                             │
│  ┌───────────────────────────────────────┐                      │
│  │  SbiMessage::{EID, FID, 参数}         │                      │
│  └───────────────────────────────────────┘                      │
│    │                                                             │
│    │ 2. 分发处理 (当前只实现了 Reset)                            │
│    ▼                                                             │
│  ┌───────────────────────────────────────┐                      │
│  │  match msg {                          │                      │
│  │    Reset(_) => shutdown(),            │                      │
│  │    _ => todo!(),                      │                      │
│  │  }                                    │                      │
│  └───────────────────────────────────────┘                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## 关键技术点

### 1. hstatus 寄存器的关键位

**SPV (Previous Virtualization Mode)**:
- `0`: 返回时进入 Host 模式
- `1`: 返回时进入 Guest 模式 (VS-mode)
- 在 `sret` 时生效，决定返回后的虚拟化状态

**SPVP (Previous Privilege Mode)**:
- `0`: 陷阱前处于 U/VU-mode
- `1`: 陷阱前处于 S/VS-mode
- 影响内存访问的地址转换方式

### 2. 世界切换的关键步骤

**进入 Guest (Host → Guest)**:
1. 保存 Host 所有 GPRs 和关键 CSRs
2. 设置 `stvec` 指向 `_guest_exit`
3. 恢复 Guest 所有 GPRs 和关键 CSRs
4. 执行 `sret`，此时 `hstatus.SPV=1` 确保进入 Guest

**退出 Guest (Guest → Host)**:
1. 异常自动触发，跳转到 `stvec` (_guest_exit)
2. 保存 Guest 所有 GPRs
3. 恢复 Host CSRs 和 GPRs
4. 执行 `ret` 返回到 Rust 代码

### 3. G-Stage 地址转换

**HGATP 格式** (Sv39x4 模式):

```
| Bit 63-60 | Bit 59-44 | Bit 43-0 |
|    MODE   |  Reserved |   PPN    |
```

- `MODE = 8`: Sv39x4 (Guest 39-bit 虚拟地址，4 级页表)
- `PPN`: G-Stage 页表根地址的物理页号

**TLB 刷新**:
- `hfence_gvma_all()`: 刷新所有 G-Stage TLB 条目
- `hfence_gvma_va(va)`: 刷新指定虚拟地址的 G-Stage TLB

### 4. 异常委托

**hedeleg (Hypervisor Exception Delegation)**:
- 设置为 `1` 的异常会直接交付给 Guest 处理
- 设置为 `0` 的异常会触发 VMExit，由 Hypervisor 处理

**hideleg (Hypervisor Interrupt Delegation)**:
- 类似 hedeleg，但针对中断
- 允许 Guest 直接处理某些中断

### 5. 寄存器偏移计算

```rust
// 使用 memoffset 确保编译时计算偏移
const fn guest_gpr_offset(index: GprIndex) -> usize {
    offset_of!(VmCpuRegisters, guest_regs)
        + offset_of!(GuestCpuState, gprs)
        + (index as usize) * size_of::<u64>()
}

// 汇编中使用这些偏移
global_asm!(
    guest_a0 = const guest_gpr_offset(GprIndex::A0),
    // ...
);
```

---

## 总结

`h_1_0` 是一个精简的 RISC-V Hypervisor 实现，展示了以下核心概念：

1. **虚拟化特权级**: HS-mode (Hypervisor) 和 VS-mode (Guest OS) 的隔离
2. **世界切换**: 通过汇编实现的上下文保存/恢复机制
3. **两级地址转换**: G-Stage (GPA→PA) 和 VS-Stage (GVA→GPA)
4. **VMExit 处理**: 捕获 Guest 的异常和 SBI 调用
5. **SBI 转发**: 将 Guest 的 SBI 请求转发到固件
