# h_2_0 Hypervisor 代码分析文档

## 目录

1. [项目概述](#项目概述)
2. [与 h_1_0 的对比](#与-h_1_0-的对比)
3. [项目结构](#项目结构)
4. [核心模块分析](#核心模块分析)
5. [关键技术改进](#关键技术改进)
6. [数据流分析](#数据流分析)
7. [关键技术点](#关键技术点)
8. [总结](#总结)

---

## 项目概述

`h_2_0` 是 `h_1_0` 的升级版本，基于 ArceOS 框架实现的 **RISC-V Hypervisor**。与 `h_1_0` 相比，`h_2_0` 进行了重要的架构重构：将 vCPU 核心实现抽取为独立的 `riscv_vcpu` 模块，实现了更好的模块化和代码复用。

### 主要特性

- **模块化架构**: vCPU 实现独立为 `riscv_vcpu` crate
- **按需内存映射**: 支持缺页异常处理 (NestedPageFault)
- **透传模式**: 支持设备地址的直通映射
- **完整的 SBI 转发**: 支持 Base、Timer、RFNC、PMU 等扩展
- **增强的 VMExit 处理**: 支持更多类型的 VMExit 原因

### 技术栈

| 组件 | 说明 |
|------|------|
| **ArceOS** | 模块化操作系统框架 |
| **riscv_vcpu** | 独立的 vCPU 实现模块 |
| **RISC-V H 扩展** | 硬件虚拟化支持 |
| **SBI Specification** | Supervisor Binary Interface 标准 |
| **axstd** | ArceOS 标准库 |
| **axmm** | ArceOS 内存管理 |
| **axhal** | ArceOS 硬件抽象层 |

### 技术验证

| 验证项 | 说明 |
|--------|------|
| **模块化设计** | vCPU 实现可复用 |
| **按需映射** | 缺页时动态映射设备内存 |
| **定时器虚拟化** | 完整的定时器中断注入 |
| **SBI 转发** | 多种 SBI 扩展支持 |

---

## 与 h_1_0 的对比

### 架构变化

```
h_1_0 架构:
┌─────────────────────────────────────┐
│          h_1_0 (单一 crate)          │
│  ┌─────────────────────────────────┐│
│  │  src/                           ││
│  │  ├── main.rs                    ││
│  │  ├── vcpu.rs                    ││  ← 所有代码在同一个包
│  │  ├── regs.rs                    ││
│  │  ├── csrs.rs                    ││
│  │  ├── guest.S                    ││
│  │  └── sbi/                       ││
│  └─────────────────────────────────┘│
└─────────────────────────────────────┘

h_2_0 架构:
┌─────────────────────────────────────┐
│          h_2_0 (应用层)              │
│  ┌─────────────────────────────────┐│
│  │  src/main.rs                    ││  ← 只负责业务逻辑
│  └─────────────────────────────────┘│
└─────────────────────────────────────┘
              │ 依赖
              ▼
┌─────────────────────────────────────┐
│       riscv_vcpu (通用模块)          │
│  ┌─────────────────────────────────┐│
│  │  src/                           ││
│  │  ├── lib.rs                     ││  ← 可复用的 vCPU 实现
│  │  ├── vcpu.rs                    ││
│  │  ├── regs.rs                    ││
│  │  ├── csrs.rs                    ││
│  │  ├── guest.S                    ││
│  │  ├── vmexit.rs                  ││
│  │  └── sbi/                       ││
│  └─────────────────────────────────┘│
└─────────────────────────────────────┘
```

### 功能对比

| 特性 | h_1_0 | h_2_0 |
|------|-------|-------|
| 代码组织 | 单一 crate | 模块化 (vCPU 独立) |
| VMExit 处理 | 仅 SBI | SBI + 缺页 + 外部中断 |
| 内存映射 | 静态映射 | 按需映射 (缺页处理) |
| 设备支持 | 无 | pflash 透传 |
| SBI 扩展 | 部分 (仅 Reset) | 完整 (Base/Timer/RFNC/PMU) |
| 中断虚拟化 | 无 | 定时器中断注入 |
| CSR 操作 | 直接内联汇编 | 抽象为 RiscvCsrTrait |
| Guest | u_3_0 (简单设备访问) | u_3_0 (简单设备访问) |

### 代码差异

```diff
--- h_1_0/src/main.rs
+++ h_2_0/src/main.rs

- // h_1_0: 所有代码在一个 crate
- mod vcpu;
- mod regs;
- mod csrs;
- mod sbi;
- mod guest;
- mod loader;
- mod task;

+ // h_2_0: 使用独立的 riscv_vcpu 模块
+ use riscv_vcpu::RISCVVCpu;
+ use riscv_vcpu::AxVCpuExitReason;

fn main() {
-     // h_1_0: 直接操作 CSR
-     unsafe { setup_hstatus() };
+     // h_2_0: 通过模块初始化
+     unsafe { riscv_vcpu::setup_csrs() };

-     // h_1_0: 使用本地 vCPU 实现
-     let mut ctx = VmCpuRegisters::default();
-     prepare_guest_context(&mut ctx);
+     // h_2_0: 使用 riscv_vcpu 模块
+     let mut arch_vcpu = RISCVVCpu::init();
+     arch_vcpu.set_entry(KERNEL_BASE.into()).unwrap();
+     arch_vcpu.set_ept_root(aspace.page_table_root()).unwrap();

      loop {
-         // h_1_0: 直接处理 VMExit
-         run_guest(&mut ctx);
-         vmexit_handler(&ctx);
+         // h_2_0: 通过模块处理
+         match vcpu_run(&mut arch_vcpu) {
+             Ok(NestedPageFault { addr, .. }) => {
+                 // 按需映射
+                 aspace.map_linear(addr, addr.as_usize().into(), 4096, flags);
+             }
+             Ok(_) => {}
+             Err(err) => panic!("error: {:?}", err),
+         }
      }
  }
```

---

## 项目结构

```
h_2_0/
├── Cargo.toml              # 项目配置
├── src/
│   └── main.rs             # 主入口 (约 140 行)

modules/riscv_vcpu/         # 独立的 vCPU 模块
├── Cargo.toml
├── src/
│   ├── lib.rs              # 库入口, CSR 初始化
│   ├── vcpu.rs             # RISCVVCpu 核心实现
│   ├── regs.rs             # 通用寄存器定义
│   ├── csrs.rs             # CSR 抽象和位段定义
│   ├── guest.S             # 世界切换汇编实现
│   ├── detect.rs           # H 扩展检测
│   └── sbi/                # SBI 调用处理
│       ├── mod.rs          # SBI 消息定义和分发
│       ├── base.rs         # Base 扩展
│       ├── dbcn.rs         # Debug Console 扩展
│       ├── srst.rs         # System Reset 扩展
│       ├── rfnc.rs         # Remote Fence 扩展
│       └── pmu.rs          # Performance Monitor 扩展

u_3_0/                      # h_2_0 的 Guest
├── Cargo.toml
└── src/
    └── main.rs             # 访问 pflash 设备的简单程序
```

---

## 核心模块分析

### 1. 主流程 ([main.rs](src/main.rs))

#### main 函数流程图

```
┌─────────────────────────────────────────────────────────────┐
│                          main()                             │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
        ┌───────────────────────────────────────┐
        │  setup_csrs()                        │  CSR 初始化
        │  - 委托异常给 Guest                  │
        │  - 委托中断给 Guest                  │
        └───────────────────────────────────────┘
                            │
                            ▼
        ┌───────────────────────────────────────┐
        │  AddrSpace::new_empty()              │  创建 VM 地址空间
        └───────────────────────────────────────┘
                            │
                            ▼
        ┌───────────────────────────────────────┐
        │  map_alloc(物理内存)                  │  映射物理内存区域
        └───────────────────────────────────────┘
                            │
                            ▼
        ┌───────────────────────────────────────┐
        │  load_vm_image("u_3_0_...bin")       │  加载 Guest 镜像
        └───────────────────────────────────────┘
                            │
                            ▼
        ┌───────────────────────────────────────┐
        │  RISCVVCpu::init()                   │  初始化 vCPU
        │  set_entry(0x8020_0000)              │  设置入口地址
        │  set_ept_root(aspace.root())         │  设置 G-Stage 页表
        └───────────────────────────────────────┘
                            │
                            ▼
        ┌───────────────────────────────────────┐
        │  loop {                              │  主循环
        │    match vcpu_run() {                │
        │      NestedPageFault =>              │
        │        map_linear() 透传映射         │
        │    }                                 │
        │  }                                   │
        └───────────────────────────────────────┘
```

#### 核心代码

```rust
fn main() {
    info!("Starting virtualization...");
    unsafe {
        riscv_vcpu::setup_csrs();
    }

    // Setup AddressSpace and regions.
    let mut aspace = AddrSpace::new_empty(VirtAddr::from(VM_ASPACE_BASE), VM_ASPACE_SIZE).unwrap();

    // Physical memory region. Full access flags.
    let mapping_flags = MappingFlags::from_bits(0xf).unwrap();
    aspace.map_alloc(PHY_MEM_START.into(), PHY_MEM_SIZE, mapping_flags, true).unwrap();

    // Load corresponding images for VM.
    info!("VM created success, loading images...");
    let image_fname = "/sbin/u_3_0_riscv64-qemu-virt.bin";
    load_vm_image(image_fname.to_string(), KERNEL_BASE.into(), &aspace).expect("Failed to load VM images");

    // Create VCpus.
    let mut arch_vcpu = RISCVVCpu::init();

    // Setup VCpus.
    info!("bsp_entry: {:#x}; ept: {:#x}", KERNEL_BASE, aspace.page_table_root());
    arch_vcpu.set_entry(KERNEL_BASE.into()).unwrap();
    arch_vcpu.set_ept_root(aspace.page_table_root()).unwrap();

    loop {
        match vcpu_run(&mut arch_vcpu) {
            Ok(exit_reason) => match exit_reason {
                AxVCpuExitReason::Nothing => {},
                NestedPageFault{addr, access_flags} => {
                    debug!("addr {:#x} access {:#x}", addr, access_flags);
                    assert_eq!(addr, 0x2200_0000.into(), "Now we ONLY handle pflash#2.");
                    let mapping_flags = MappingFlags::from_bits(0xf).unwrap();
                    // Passthrough-Mode
                    let _ = aspace.map_linear(addr, addr.as_usize().into(), 4096, mapping_flags);

                    /*
                    // Emulator-Mode
                    // Pretend to load file to fill buffer.
                    let buf = "pfld";
                    aspace.map_alloc(addr, 4096, mapping_flags, true);
                    aspace.write(addr, buf.as_bytes());
                    */
                },
                _ => {
                    panic!("Unhandled VM-Exit: {:?}", exit_reason);
                }
            },
            Err(err) => {
                panic!("run VCpu get error {:?}", err);
            }
        }
    }
}
```

### 2. vCPU 核心 ([riscv_vcpu/src/vcpu.rs](../../modules/riscv_vcpu/src/vcpu.rs))

#### RISCVVCpu 结构

```rust
#[derive(Default)]
pub struct RISCVVCpu {
    regs: VmCpuRegisters,
}

pub struct VmCpuRegisters {
    // Hypervisor 状态 (进入/退出 VM 时保存/恢复)
    hyp_regs: HypervisorCpuState,

    // Guest 状态
    pub guest_regs: GuestCpuState,

    // V=1 时有效的 VS-level CSR
    vs_csrs: GuestVsCsrs,

    // 虚拟化的 HS-level CSR
    virtual_hs_csrs: GuestVirtualHsCsrs,

    // VMExit 时读取的 CSR
    pub trap_csrs: VmCpuTrapState,
}
```

#### 初始化流程

```rust
pub fn init() -> Self {
    let mut regs = VmCpuRegisters::default();

    // 设置 hstatus
    let mut hstatus = LocalRegisterCopy::<usize, hstatus::Register>::new(
        riscv::register::hstatus::read().bits(),
    );
    hstatus.modify(hstatus::spv::Supervisor);  // 设置 SPV=1
    // 设置 SPVP=1, 从 HS-mode 访问 VS-mode 内存
    hstatus.modify(hstatus::spvp::Supervisor);
    CSR.hstatus.write_value(hstatus.get());
    regs.guest_regs.hstatus = hstatus.get();

    // 设置 sstatus (SPP = Supervisor)
    let mut sstatus = sstatus::read();
    sstatus.set_spp(sstatus::SPP::Supervisor);
    regs.guest_regs.sstatus = sstatus.bits();

    // 清除 Guest 定时器中断
    CSR.sie.read_and_clear_bits(traps::interrupt::SUPERVISOR_TIMER);

    Self { regs }
}
```

#### 运行 Guest

```rust
pub fn run(&mut self) -> AxResult<AxVCpuExitReason> {
    let regs = &mut self.regs;
    unsafe {
        // 调用汇编入口进行世界切换
        _run_guest(regs);
    }
    self.vmexit_handler()
}
```

### 3. VMExit 处理 ([riscv_vcpu/src/vcpu.rs](../../modules/riscv_vcpu/src/vcpu.rs))

`h_2_0` 支持多种类型的 VMExit：

```rust
fn vmexit_handler(&mut self) -> AxResult<AxVCpuExitReason> {
    // 读取 trap 相关 CSR
    self.regs.trap_csrs.scause = scause::read().bits();
    self.regs.trap_csrs.stval = stval::read();
    self.regs.trap_csrs.htval = htval::read();
    self.regs.trap_csrs.htinst = htinst::read();

    let scause = scause::read();
    match scause.cause() {
        // 1. SBI 调用 (ecall)
        Trap::Exception(Exception::VirtualSupervisorEnvCall) => {
            let sbi_msg = SbiMessage::from_regs(self.regs.guest_regs.gprs.a_regs()).ok();
            debug!("VSuperEcall: {:?}", sbi_msg);
            if let Some(sbi_msg) = sbi_msg {
                match sbi_msg {
                    SbiMessage::Base(base) => {
                        self.handle_base_function(base).unwrap();
                    }
                    SbiMessage::GetChar => {
                        let c = sbi_rt::legacy::console_getchar();
                        self.set_gpr_from_gpr_index(GprIndex::A0, c);
                    }
                    SbiMessage::PutChar(c) => {
                        sbi_rt::legacy::console_putchar(c);
                    }
                    SbiMessage::SetTimer(timer) => {
                        info!("Set timer... ");
                        sbi_rt::set_timer(timer as u64);
                        // 清除 guest timer 中断
                        CSR.hvip.read_and_clear_bits(traps::interrupt::VIRTUAL_SUPERVISOR_TIMER);
                        // 使能 host timer 中断
                        CSR.sie.read_and_set_bits(traps::interrupt::SUPERVISOR_TIMER);
                    }
                    SbiMessage::Reset(_) => {
                        sbi_rt::system_reset(sbi_rt::Shutdown, sbi_rt::SystemFailure);
                    }
                    SbiMessage::RemoteFence(rfnc) => {
                        self.handle_rfnc_function(rfnc).unwrap();
                    }
                    SbiMessage::PMU(pmu) => {
                        self.handle_pmu_function(pmu).unwrap();
                    }
                    _ => todo!(),
                }
                self.advance_pc(4);
                Ok(AxVCpuExitReason::Nothing)
            } else {
                panic!()
            }
        }

        // 2. 定时器中断 (新增)
        Trap::Interrupt(Interrupt::SupervisorTimer) => {
            info!("timer irq emulation");
            // 注入虚拟定时器中断给 Guest
            CSR.hvip.read_and_set_bits(traps::interrupt::VIRTUAL_SUPERVISOR_TIMER);
            // 清除 host 定时器中断
            CSR.sie.read_and_clear_bits(traps::interrupt::SUPERVISOR_TIMER);
            Ok(AxVCpuExitReason::Nothing)
        }

        // 3. 外部中断 (新增)
        Trap::Interrupt(Interrupt::SupervisorExternal) => {
            Ok(AxVCpuExitReason::ExternalInterrupt { vector: 0 })
        }

        // 4. Guest 缺页异常 (新增, 按需映射)
        Trap::Exception(Exception::LoadGuestPageFault)
        | Trap::Exception(Exception::StoreGuestPageFault) => {
            let fault_addr = self.regs.trap_csrs.htval << 2
                           | self.regs.trap_csrs.stval & 0x3;
            Ok(AxVCpuExitReason::NestedPageFault {
                addr: GuestPhysAddr::from(fault_addr),
                access_flags: MappingFlags::empty(),
            })
        }

        _ => {
            panic!(
                "Unhandled trap: {:?}, sepc: {:#x}, stval: {:#x}",
                scause.cause(),
                self.regs.guest_regs.sepc,
                self.regs.trap_csrs.stval
            );
        }
    }
}
```

### 4. CSR 抽象 ([riscv_vcpu/src/csrs.rs](../../modules/riscv_vcpu/src/csrs.rs))

`h_2_0` 使用 trait 抽象 CSR 操作，提供类型安全的接口：

```rust
pub trait RiscvCsrTrait {
    type R: RegisterLongName;
    fn get_value(&self) -> usize;
    fn write_value(&self, value: usize);
    fn atomic_replace(&self, value: usize) -> usize;
    fn read_and_set_bits(&self, bitmasks: usize) -> usize;
    fn read_and_clear_bits(&self, bitmasks: usize) -> usize;
}

pub struct ReadWriteCsr<R: RegisterLongName, const V: u16> { ... }

pub const CSR: &CSR = &CSR {
    sie: ReadWriteCsr::new(),
    hstatus: ReadWriteCsr::new(),
    hedeleg: ReadWriteCsr::new(),
    hideleg: ReadWriteCsr::new(),
    hcounteren: ReadWriteCsr::new(),
    hvip: ReadWriteCsr::new(),
};
```

#### 使用示例

```rust
// 读取 CSR
let hstatus = CSR.hstatus.get();

// 写入 CSR
CSR.hstatus.write_value(hstatus);

// 原子操作 (读并置位)
CSR.hvip.read_and_set_bits(traps::interrupt::VIRTUAL_SUPERVISOR_TIMER);

// 原子操作 (读并清除)
CSR.sie.read_and_clear_bits(traps::interrupt::SUPERVISOR_TIMER);
```

### 5. 世界切换 ([riscv_vcpu/src/guest.S](../../modules/riscv_vcpu/src/guest.S))

汇编实现与 `h_1_0` 基本相同：

```
_run_guest(state: *mut VmCpuRegisters)
    │
    ├─► 保存 Hypervisor GPRs (ra, gp, tp, s0-s11, a1-a7, sp)
    │
    ├─► 交换 CSR (sstatus, hstatus, scounteren, sepc)
    │   并保存 Hypervisor 值
    │
    ├─► 设置 stvec = _guest_exit
    │
    ├─► 恢复 Guest GPRs (包括 t0-t6)
    │
    ├─► sret  → 进入 Guest
    │
    ║  (Guest 执行, 发生异常/ecall/中断)
    ║
    ▼
_guest_exit
    │
    ├─► 恢复 GuestInfo 指针到 a0
    │
    ├─► 保存 Guest GPRs
    │
    ├─► 交换 CSR 回 Hypervisor
    │   并保存 Guest 值
    │
    ├─► 恢复 Hypervisor GPRs
    │
    └─► ret  → 返回 Rust 代码
```

---

## 关键技术改进

### 1. 按需内存映射 (缺页处理)

`h_2_0` 支持在 Guest 访问未映射地址时动态处理：

```rust
// main.rs 中的处理
match vcpu_run(&mut arch_vcpu) {
    Ok(NestedPageFault { addr, access_flags }) => {
        debug!("addr {:#x} access {:#x}", addr, access_flags);

        // 目前只处理 pflash#2 (0x2200_0000)
        assert_eq!(addr, 0x2200_0000.into());

        // 透传模式: 直接映射 GPA = PA
        let mapping_flags = MappingFlags::from_bits(0xf).unwrap();
        aspace.map_linear(addr, addr.as_usize(), 4096, mapping_flags);
    }
    // ...
}
```

这允许 Guest 访问设备内存区域，而无需预先映射所有可能的地址。

### 2. 定时器中断虚拟化

`h_2_0` 实现了完整的定时器中断虚拟化：

```
Guest 设置定时器 (SBI call)
    │
    ▼
Hypervisor 收到 VSuperEcall
    │
    ├─► 调用 sbi_rt::set_timer() 设置 Host 定时器
    ├─► 清除 Guest 的虚拟定时器中断 (HVIP.VSTIP)
    └─► 使能 Host 的定时器中断 (SIE.STIE)
    │
    ▼
Host 定时器到期
    │
    ▼
Hypervisor 收到 SupervisorTimer
    │
    ├─► 设置 Guest 虚拟定时器中断 (HVIP.VSTIP = 1)
    └─► 清除 Host 定时器中断 (SIE.STIE = 0)
    │
    ▼
下次进入 Guest 时
    │
    └─► 硬件自动注入虚拟中断给 Guest
```

### 3. 完整的 SBI 转发

`h_2_0` 支持更多 SBI 扩展：

| 扩展 | EID | 功能 |
|------|-----|------|
| Base | 0x10 | 获取 SBI 版本/实现 ID |
| Time | 0x54494D45 | 设置定时器 |
| SRST | 0x53525354 | 系统复位 |
| RFNC | 0x52464E43 | 远程 fence 操作 |
| PMU | 0x504D55 | 性能监控单元 |
| Legacy | - | 兼容 v0.1 扩展 (GetChar/PutChar) |

### 4. 模块化设计

通过将 vCPU 实现独立为 `riscv_vcpu` crate：

- **代码复用**: 其他项目可以直接使用 `riscv_vcpu`
- **职责分离**: Hypervisor 业务逻辑与 vCPU 实现分离
- **易于测试**: 可以独立测试 vCPU 模块
- **版本管理**: 可以独立更新 vCPU 模块

---

## 数据流分析

### Guest 启动流程

```
┌─────────────────────────────────────────────────────────────────┐
│                        Hypervisor (HS-mode)                     │
└─────────────────────────────────────────────────────────────────┘
                            │
                            │ 1. setup_csrs()
                            │    - 委托异常给 Guest
                            │    - 委托中断给 Guest
                            │
                            ▼
        ┌───────────────────────────────────────┐
        │  CSR 状态                             │
        │  - hedeleg = 0x1B033 (委托异常)      │
        │  - hideleg = 0x1404 (委托中断)        │
        │  - hcounteren = 0xFFFFFFFF            │
        └───────────────────────────────────────┘
                            │
                            │ 2. 创建地址空间
                            ▼
        ┌───────────────────────────────────────┐
        │  AddrSpace                            │
        │  - base: 0x0                          │
        │  - size: 0x7fff_ffff_f000            │
        └───────────────────────────────────────┘
                            │
                            │ 3. 映射物理内存
                            ▼
        ┌───────────────────────────────────────┐
        │  物理内存映射                         │
        │  - GPA: 0x8000_0000 → PA: 0x8000_0000│
        │  - size: 0x100_0000 (16MB)           │
        └───────────────────────────────────────┘
                            │
                            │ 4. 加载 Guest 镜像
                            ▼
        ┌───────────────────────────────────────┐
        │  Guest Code                           │
        │  - 文件: /sbin/u_3_0_...              │
        │  - 加载到: 0x8020_0000                │
        └───────────────────────────────────────┘
                            │
                            │ 5. 初始化 vCPU
                            ▼
        ┌───────────────────────────────────────┐
        │  RISCVVCpu                            │
        │  - entry: 0x8020_0000                 │
        │  - hgatp: ept_root                    │
        │  - hstatus.SPV = 1                    │
        │  - hstatus.SPVP = 1                   │
        │  - sstatus.SPP = Supervisor           │
        └───────────────────────────────────────┘
                            │
                            │ 6. 运行循环
                            ▼
        ┌───────────────────────────────────────┐
        │  loop {                               │
        │    match vcpu_run() {                 │
        │      NestedPageFault => map_device(), │
        │      _ => {},                         │
        │    }                                  │
        │  }                                    │
        └───────────────────────────────────────┘
```

### 缺页处理流程

```
┌─────────────────────────────────────────────────────────────────┐
│                        Guest (VS-mode)                          │
│  访问地址 0x2200_0000 (pflash)                                  │
└─────────────────────────────────────────────────────────────────┘
                            │
                            │ 未映射的 GPA
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     硬件 G-Stage 页表遍历                       │
│  查找 GPA 0x2200_0000 → 失败 (无映射)                           │
└─────────────────────────────────────────────────────────────────┘
                            │
                            │ 触发 Guest 缺页异常
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     VMExit (自动)                               │
│  scause = LoadGuestPageFault (13) 或 StoreGuestPageFault (15)   │
│  stval = GPA[1:0]                                               │
│  htval = GPA[53:2]                                              │
└─────────────────────────────────────────────────────────────────┘
                            │
                            │ 世界切换
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Hypervisor (HS-mode)                        │
│  vmexit_handler()                                               │
│    │                                                             │
│    │ 1. 计算完整 GPA                                             │
│    │    fault_addr = htval << 2 | stval & 0x3                  │
│    │    fault_addr = 0x2200_0000                               │
│    │                                                             │
│    ▼                                                             │
│  ┌───────────────────────────────────────┐                      │
│  │  返回 NestedPageFault {               │                      │
│  │    addr: 0x2200_0000,                 │                      │
│  │    access_flags: empty,               │                      │
│  │  }                                    │                      │
│  └───────────────────────────────────────┘                      │
│    │                                                             │
│    ▼                                                             │
│  main.rs 主循环                                                 │
│    │                                                             │
│    │ 2. 检查地址                                                 │
│    ▼                                                             │
│  assert_eq!(addr, 0x2200_0000)                                  │
│    │                                                             │
│    │ 3. 透传映射 (GPA = PA)                                     │
│    ▼                                                             │
│  aspace.map_linear(                                             │
│      0x2200_0000,   // GPA                                       │
│      0x2200_0000,   // PA (直接映射)                             │
│      4096,           // size                                     │
│      RWXU                                                       │
│  )                                                              │
│    │                                                             │
│    ▼                                                             │
│  下次运行 Guest 时可以正常访问该地址                             │
└─────────────────────────────────────────────────────────────────┘
```

### 定时器中断流程

```
┌─────────────────────────────────────────────────────────────────┐
│                        Guest (VS-mode)                          │
│  执行 SBI 调用设置定时器                                         │
│                                                                 │
│    a7 = 0x00 (Legacy Set Timer) 或                              │
│    a7 = 0x54494D45 (Time Extension)                             │
│    a0 = timer_value                                             │
└─────────────────────────────────────────────────────────────────┘
                            │
                            │ ecall
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     VMExit                                      │
│  scause = VirtualSupervisorEnvCall                              │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Hypervisor                                  │
│  vmexit_handler()                                               │
│    │                                                             │
│    │ 1. 解析 SBI 消息                                           │
│    ▼                                                             │
│  SbiMessage::SetTimer(timer)                                    │
│    │                                                             │
│    │ 2. 调用底层 SBI 设置 Host 定时器                            │
│    ▼                                                             │
│  sbi_rt::set_timer(timer)                                       │
│    │                                                             │
│    │ 3. 清除 Guest 虚拟定时器中断                                │
│    ▼                                                             │
│  CSR.hvip.read_and_clear_bits(VSTIMER)  ← HVIP.VSTIP = 0        │
│    │                                                             │
│    │ 4. 使能 Host 定时器中断                                     │
│    ▼                                                             │
│  CSR.sie.read_and_set_bits(STIMER)  ← SIE.STIE = 1              │
│    │                                                             │
│    ▼                                                             │
│  advance_pc(4)  → 返回 Guest 继续执行                            │
└─────────────────────────────────────────────────────────────────┘
                            │
                            │ (时间流逝...)
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Host 定时器到期                              │
│  硬件产生 SupervisorTimer 中断                                   │
└─────────────────────────────────────────────────────────────────┘
                            │
                            │ VMExit (因为定时器中断)
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Hypervisor                                  │
│  vmexit_handler()                                               │
│    │                                                             │
│    │ scause = SupervisorTimer                                   │
│    │                                                             │
│    │ 1. 设置 Guest 虚拟定时器中断                                │
│    ▼                                                             │
│  CSR.hvip.read_and_set_bits(VSTIMER)  ← HVIP.VSTIP = 1          │
│    │                                                             │
│    │ 2. 清除 Host 定时器中断                                     │
│    ▼                                                             │
│  CSR.sie.read_and_clear_bits(STIMER)  ← SIE.STIE = 0             │
│    │                                                             │
│    ▼                                                             │
│  返回 Nothing → 继续运行 Guest                                   │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Guest (VS-mode)                          │
│  进入 Guest 时, 硬件检测到 HVIP.VSTIP = 1                        │
│  自动注入虚拟定时器中断给 Guest                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 关键技术点

### 1. hstatus.SPV 的作用

在 `riscv_vcpu` 初始化时设置：

```rust
hstatus.modify(hstatus::spv::Supervisor);  // SPV = 1
```

这确保了：
- `sret` 返回时进入 **VS-mode** (Guest 模式)
- 而不是 HS-mode (Hypervisor 模式)

### 2. 虚拟定时器中断注入

```rust
// Host 定时器到期时的处理
Trap::Interrupt(Interrupt::SupervisorTimer) => {
    // 设置 Guest 虚拟定时器中断
    CSR.hvip.read_and_set_bits(traps::interrupt::VIRTUAL_SUPERVISOR_TIMER);

    // 清除 Host 定时器中断
    CSR.sie.read_and_clear_bits(traps::interrupt::SUPERVISOR_TIMER);
}
```

关键点：
- **HVIP.VSTIP**: 虚拟定时器中断挂起位
- **SIE.STIE**: Host 定时器中断使能位
- 两者配合实现"一对一"的定时器中断虚拟化

### 3. 中断委托 (hideleg)

```rust
// setup_csrs() 中的设置
CSR.hideleg.write_value(
    traps::interrupt::VIRTUAL_SUPERVISOR_TIMER
        | traps::interrupt::VIRTUAL_SUPERVISOR_EXTERNAL
        | traps::interrupt::VIRTUAL_SUPERVISOR_SOFT,
);
```

这告诉硬件：这些中断发生时，直接交付给 Guest，而不是触发 VMExit。

### 4. 异常委托 (hedeleg)

```rust
CSR.hedeleg.write_value(
    traps::exception::INST_ADDR_MISALIGN
        | traps::exception::BREAKPOINT
        | traps::exception::ENV_CALL_FROM_U_OR_VU
        | traps::exception::INST_PAGE_FAULT
        | traps::exception::LOAD_PAGE_FAULT
        | traps::exception::STORE_PAGE_FAULT
        | traps::exception::ILLEGAL_INST,
);
```

这告诉硬件：这些异常发生时，直接交付给 Guest 处理。

### 5. 缺页地址计算

Guest 物理地址由两部分组成：

```rust
let fault_addr = self.regs.trap_csrs.htval << 2
               | self.regs.trap_csrs.stval & 0x3;
```

- `htval`: GPA 的高位 [53:2]
- `stval`: GPA 的低位 [1:0]
- 组合后得到完整的 54 位 Guest 物理地址

---

## 总结

`h_2_0` 是对 `h_1_0` 的重要升级，主要体现在：

### 架构改进

1. **模块化**: 将 vCPU 实现抽取为独立的 `riscv_vcpu` crate
2. **可复用**: 其他 ArceOS 项目可以直接使用 `riscv_vcpu`
3. **职责清晰**: main.rs 只负责业务逻辑，vCPU 实现完全独立

### 功能增强

1. **按需内存映射**: 支持缺页异常处理，可以动态映射设备地址
2. **定时器虚拟化**: 完整实现定时器中断的注入和转发
3. **SBI 完整支持**: 支持 Base/Time/Reset/RFNC/PMU 等扩展
4. **外部中断**: 支持外部中断的 VMExit 处理

### 代码质量

1. **CSR 抽象**: 使用 trait 提供类型安全的 CSR 操作
2. **错误处理**: 统一使用 `AxResult` 进行错误处理
3. **可维护性**: 代码结构清晰，注释完整

### 关键代码位置

| 文件 | 行号 | 功能 |
|------|------|------|
| [h_2_0/src/main.rs](src/main.rs:28-82) | 28-82 | 主流程 |
| [riscv_vcpu/src/lib.rs](../../modules/riscv_vcpu/src/lib.rs:24-61) | 24-61 | CSR 初始化 |
| [riscv_vcpu/src/vcpu.rs](../../modules/riscv_vcpu/src/vcpu.rs:235-243) | 235-243 | 运行 Guest |
| [riscv_vcpu/src/vcpu.rs](../../modules/riscv_vcpu/src/vcpu.rs:291-374) | 291-374 | VMExit 处理 |
| [riscv_vcpu/src/csrs.rs](../../modules/riscv_vcpu/src/csrs.rs:26-97) | 26-97 | CSR 抽象 |
| [riscv_vcpu/src/guest.S](../../modules/riscv_vcpu/src/guest.S:4-181) | 4-181 | 世界切换 |

这个版本展示了如何将一个原型实现 (h_1_0) 重构为一个可复用的模块化组件 (riscv_vcpu)，是学习系统软件架构设计的优秀案例。
