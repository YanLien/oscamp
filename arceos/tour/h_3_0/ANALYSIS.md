# h_3_0 Hypervisor 代码分析文档

## 目录

1. [项目概述](#项目概述)
2. [与 h_2_0 的对比](#与-h_2_0-的对比)
3. [项目结构](#项目结构)
4. [核心模块分析](#核心模块分析)
5. [Guest 程序分析 (u_6_0)](#guest-程序分析-u_6_0)
6. [技术验证点](#技术验证点)
7. [数据流分析](#数据流分析)
8. [关键技术点](#关键技术点)
9. [总结](#总结)

---

## 项目概述

`h_3_0` 是 `h_2_0` 的延续版本，**Hypervisor 代码与 h_2_0 完全相同**，但使用了一个更复杂的 Guest 操作系统 `u_6_0`。这个版本的目的是验证 Hypervisor 是否能够正确支持**多任务抢占式调度**的 Guest 系统。

### 主要特性

- **Hypervisor 实现**: 与 h_2_0 完全相同，使用 `riscv_vcpu` 模块
- **Guest 系统**: `u_6_0` - 支持多任务和 CFS 调度器的复杂应用
- **验证目标**: 证明虚拟化环境能够正确支持抢占式多任务

### 技术验证

| 验证项 | 说明 |
|--------|------|
| **定时器虚拟化** | Guest 需要定时器中断进行任务切换 |
| **中断注入** | Hypervisor 需要正确注入虚拟定时器中断 |
| **上下文切换** | Guest 内部的任务切换与虚拟化环境兼容性 |
| **并发执行** | 多个线程在虚拟化环境下的正确性 |

### 技术栈

| 组件 | 说明 |
|------|------|
| **ArceOS** | 模块化操作系统框架 |
| **riscv_vcpu** | 独立的 vCPU 实现模块 |
| **RISC-V H 扩展** | 硬件虚拟化支持 |
| **axstd** | ArceOS 标准库 (multitask + sched_cfs) |

---

## 与 h_2_0 的对比

### 代码层面

`h_3_0` 和 `h_2_0` 的 Hypervisor 代码**完全相同**，唯一的区别是加载的 Guest 镜像：

```diff
// h_2_0
- let image_fname = "/sbin/u_3_0_riscv64-qemu-virt.bin";

// h_3_0
+ let image_fname = "/sbin/u_6_0_riscv64-qemu-virt.bin";
```

### Guest 对比

| 特性 | u_3_0 (h_2_0 Guest) | u_6_0 (h_3_0 Guest) |
|------|---------------------|---------------------|
| **功能** | 访问 pflash 设备 | 多任务生产者-消费者 |
| **依赖** | axstd (paging) | axstd (multitask, sched_cfs) |
| **复杂度** | 简单设备访问 | 抢占式多任务调度 |
| **定时器** | 不需要 | **需要** (任务抢占) |
| **线程数** | 1 | 3 (main + 2 workers) |
| **同步原语** | 无 | Arc + SpinLock |
| **调度方式** | 无 | CFS (完全公平调度器) |

### 代码差异

```diff
--- h_2_0/src/main.rs
+++ h_3_0/src/main.rs

  fn main() {
      // ... (完全相同)

      // 唯一的区别: Guest 镜像
-     let image_fname = "/sbin/u_3_0_riscv64-qemu-virt.bin";
+     let image_fname = "/sbin/u_6_0_riscv64-qemu-virt.bin";

      // ... (其余完全相同)
  }
```

---

## 项目结构

```
h_3_0/
├── Cargo.toml              # 与 h_2_0 相同
└── src/
    └── main.rs             # 与 h_2_0 相同 (仅 Guest 镜像不同)

u_6_0/                      # h_3_0 的 Guest
├── Cargo.toml              # 依赖 multitask + sched_cfs
└── src/
    └── main.rs             # 多任务生产者-消费者程序
```

---

## 核心模块分析

由于 `h_3_0` 的 Hypervisor 代码与 `h_2_0` 完全相同，这里主要分析其与 `u_6_0` Guest 的交互。

### 主流程 ([main.rs](src/main.rs))

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
    let image_fname = "/sbin/u_6_0_riscv64-qemu-virt.bin";
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
                NestedPageFault { addr, access_flags } => {
                    debug!("addr {:#x} access {:#x}", addr, access_flags);
                    assert_eq!(addr, 0x2200_0000.into(), "Now we ONLY handle pflash#2.");
                    let mapping_flags = MappingFlags::from_bits(0xf).unwrap();
                    // Passthrough-Mode
                    let _ = aspace.map_linear(addr, addr.as_usize().into(), 4096, mapping_flags);
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

### VMExit 处理

所有 VMExit 由 `riscv_vcpu` 模块处理，`h_3_0` 主循环只处理缺页异常：

```rust
match vcpu_run(&mut arch_vcpu) {
    Ok(NestedPageFault { addr, .. }) => {
        // 处理设备缺页
        aspace.map_linear(addr, addr.as_usize(), 4096, flags);
    }
    Ok(AxVCpuExitReason::Nothing) => {
        // 其他 VMExit (如 SBI 调用、定时器中断) 已由 vCPU 模块处理
    }
    // ...
}
```

---

## Guest 程序分析 (u_6_0)

### 功能概述

`u_6_0` 实现了一个经典的生产者-消费者模式，验证虚拟化环境下的多任务调度：

```
┌─────────────────────────────────────────────────────────────────┐
│                         Guest (u_6_0)                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│   ┌─────────────────┐         ┌─────────────────┐               │
│   │   Main Thread   │         │   Worker1       │               │
│   │   (producer)    │         │   (producer)    │               │
│   └─────────────────┘         └─────────────────┘               │
│            │                             │                        │
│            │ push(i)                    │ push(i)                │
│            ▼                             ▼                        │
│   ┌─────────────────────────────────────────────────┐           │
│   │         Shared Queue (VecDeque<usize>)          │           │
│   │         Arc<SpinNoIrq<VecDeque<...>>>           │           │
│   └─────────────────────────────────────────────────┘           │
│            ▲                             ▲                        │
│            │ pop_front()                │                        │
│            │                             │                        │
│   ┌─────────────────┐         ┌─────────────────┐               │
│   │   Worker2       │         │   Scheduler     │               │
│   │   (consumer)    │         │   (CFS)         │               │
│   └─────────────────┘         └─────────────────┘               │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
                            │
                            │ 需要抢占式调度
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Hypervisor (h_3_0)                         │
│  • 定时器中断虚拟化                                             │
│  • SBI 调用转发 (设置定时器)                                     │
│  • 虚拟定时器中断注入                                           │
└─────────────────────────────────────────────────────────────────┘
```

### 代码分析

```rust
const LOOP_NUM: usize = 256;

#[no_mangle]
fn main() {
    ax_println!("Multi-task(Preemptible) is starting ...");

    // 1. 创建共享队列
    let q1 = Arc::new(SpinNoIrq::new(VecDeque::new()));
    let q2 = q1.clone();

    // 2. 启动生产者线程 (Worker1)
    let worker1 = thread::spawn(move || {
        ax_println!("worker1 ... {:?}", thread::current().id());
        for i in 0..=LOOP_NUM {
            ax_println!("worker1 [{i}]");
            q1.lock().push_back(i);  // 生产数据
        }
        ax_println!("worker1 ok!");
    });

    // 3. 启动消费者线程 (Worker2)
    let worker2 = thread::spawn(move || {
        ax_println!("worker2 ... {:?}", thread::current().id());
        loop {
            if let Some(num) = q2.lock().pop_front() {
                ax_println!("worker2 [{num}]");
                if num == LOOP_NUM {
                    break;
                }
            } else {
                ax_println!("worker2: nothing to do!");
                thread::yield_now();  // 主动让出 CPU
            }
        }
        ax_println!("worker2 ok!");
    });

    // 4. 等待所有线程完成
    ax_println!("Wait for workers to exit ...");
    let _ = worker1.join();
    let _ = worker2.join();

    ax_println!("Multi-task(Preemptible) ok!");
}
```

### 依赖特性

```toml
[dependencies]
axstd = { workspace = true, features = [
    "alloc",      # 动态内存分配
    "paging",     # 页表管理
    "multitask",  # 多任务支持
    "sched_cfs"   # CFS 调度器
]}
```

### 执行流程

```
main()
    │
    ├─► 创建共享队列 (Arc<VecDeque>)
    │
    ├─► 启动 Worker1 (生产者)
    │   └─► for i in 0..=256
    │       └─► q1.lock().push_back(i)
    │
    ├─► 启动 Worker2 (消费者)
    │   └─► loop
    │       ├─► q2.lock().pop_front()
    │       └─► thread::yield_now() (队列为空时)
    │
    └─► worker1.join()
        worker2.join()
```

### CFS 调度器行为

```
时间线:
t0: main 启动 Worker1, Worker2
t1: Worker1 执行, push(0), push(1), push(2)
t2: 定时器中断 → CFS 切换到 Worker2
t3: Worker2 执行, pop_front() → 0
t4: 定时器中断 → CFS 切换到 Worker1
t5: Worker1 执行, push(3), push(4), ...
...
tN: Worker1 完成, Worker2 消费完所有数据
```

---

## 技术验证点

### 1. 定时器中断虚拟化

`u_6_0` 使用 CFS 调度器，**必须依赖定时器中断**来实现抢占式调度：

```
Guest (u_6_0)                    Hypervisor (h_3_0)
     │                                  │
     │ 1. SBI_SET_TIMER                │
     ├─────────────────────────────────►│
     │                                  │ 2. sbi_rt::set_timer()
     │                                  │    设置 Host 定时器
     │                                  │    CSR.hvip &= ~VSTIMER
     │                                  │    CSR.sie |= STIMER
     │                                  │
     │ (继续执行...)                    │
     │                                  │
     │ ◄─────────────────────────────────┤ 3. Host 定时器到期
     │   VMExit (SupervisorTimer)       │    CSR.hvip |= VSTIMER
     │                                  │    CSR.sie &= ~STIMER
     │                                  │
     │ 4. 返回 Guest                     │
     ├─────────────────────────────────►│
     │                                  │
     │ 5. 硬件自动注入 VSTIP 中断        │
     │   (触发任务切换)                  │
     │                                  │
```

### 2. 上下文切换验证

`u_6_0` 的任务切换流程：

```
┌─────────────────────────────────────────────────────────────────┐
│                     Guest (u_6_0)                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│   Worker1 执行                                                   │
│     │                                                            │
│     │ 定时器中断                                                 │
│     ▼                                                            │
│   保存上下文 (ra, sp, registers...)                              │
│     │                                                            │
│     ▼                                                            │
│   调度器选择下一个任务 (Worker2)                                 │
│     │                                                            │
│     ▼                                                            │
│   恢复 Worker2 上下文                                            │
│     │                                                            │
│     ▼                                                            │
│   Worker2 执行                                                   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

在虚拟化环境下，这个流程需要：
1. **Guest 内的上下文切换** (由 ArceOS 调度器处理)
2. **Guest 和 Host 间的世界切换** (由 Hypervisor 处理)
3. **虚拟中断的正确注入** (由 hstatus/hvip 控制)

### 3. 并发执行验证

`u_6_0` 的执行时序（在虚拟化环境下）：

```
时间 │ Hypervisor           │ Guest (u_6_0)
─────────────────────────────────────────────────────────────────
 t0  │ 运行 Guest          │ [main] 启动 Worker1
     │                     │ [main] 启动 Worker2
     │                     │ [main] 等待 join
─────┼─────────────────────┼────────────────────────────────────
 t1  │ VMExit (SBI call)   │ [Worker1] 设置定时器
     │ 转发 SBI call       │
     │ 返回 Guest          │
─────┼─────────────────────┼────────────────────────────────────
 t2  │ 运行 Guest          │ [Worker1] push(0), push(1), push(2)
     │                     │ [Worker2] pop_front() → 0
     │                     │ [Worker2] pop_front() → 1
─────┼─────────────────────┼────────────────────────────────────
 t3  │ VMExit (Timer)      │ 定时器中断
     │ 注入虚拟定时器中断   │
     │ 返回 Guest          │
─────┼─────────────────────┼────────────────────────────────────
 t4  │ 运行 Guest          │ [调度器] 切换到 Worker1
     │                     │ [Worker1] push(3), push(4), ...
     │                     │
─────┼─────────────────────┼────────────────────────────────────
 ... │ (重复...)           │ (直到 LOOP_NUM = 256)
─────┼─────────────────────┼────────────────────────────────────
 tn  │ 运行 Guest          │ [Worker1] 退出
     │                     │ [Worker2] 退出
     │                     │ [main] join 完成
     │                     │ 打印 "ok!"
```

### 4. 同步原语验证

`u_6_0` 使用了 `Arc<SpinNoIrq<VecDeque>>` 来实现线程间同步：

```rust
let q1 = Arc::new(SpinNoIrq::new(VecDeque::new()));
let q2 = q1.clone();
```

在虚拟化环境下验证：
- **Arc** (原子引用计数): 需要正确的原子操作支持
- **SpinNoIrq** (无中断自旋锁): 在虚拟化环境下不会关中断，避免影响虚拟中断注入
- **VecDeque** (双端队列): 作为生产者-消费者的缓冲区

---

## 数据流分析

### 启动流程

```
┌─────────────────────────────────────────────────────────────────┐
│                   Hypervisor (h_3_0)                            │
└─────────────────────────────────────────────────────────────────┘
                            │
                            │ 1. setup_csrs()
                            ▼
        ┌───────────────────────────────────────┐
        │  CSR 初始化                           │
        │  - hedeleg: 委托异常                 │
        │  - hideleg: 委托中断                 │
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
        │  物理内存: 0x8000_0000 → 0x8000_0000  │
        │  大小: 16MB                           │
        └───────────────────────────────────────┘
                            │
                            │ 4. 加载 Guest (u_6_0)
                            ▼
        ┌───────────────────────────────────────┐
        │  u_6_0 代码: 0x8020_0000              │
        └───────────────────────────────────────┘
                            │
                            │ 5. 初始化 vCPU
                            ▼
        ┌───────────────────────────────────────┐
        │  RISCVVCpu                            │
        │  - entry: 0x8020_0000                 │
        │  - hgatp: ept_root                    │
        └───────────────────────────────────────┘
                            │
                            │ 6. 运行 Guest
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Guest (u_6_0)                               │
│  启动多任务                                                     │
│    Worker1: 生产 0-256                                         │
│    Worker2: 消费 0-256                                         │
│    CFS 调度器抢占式切换                                         │
└─────────────────────────────────────────────────────────────────┘
                            │
                            │ 定时器中断
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Hypervisor (h_3_0)                           │
│  注入虚拟定时器中断                                             │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
                    (重复...)
```

### 定时器中断流程

```
Guest (u_6_0)                    Hypervisor (h_3_0)
     │                                  │
     │ Worker1 执行                      │
     │                                  │
     │ 定时器到期                        │
     ├─────────────────────────────────►│ VMExit
     │                                  │
     │ ◄─────────────────────────────────┤ 返回 Guest
     │                                  │
     │ CFS 调度器切换到 Worker2          │
     │                                  │
     │ Worker2 执行                      │
     │                                  │
     │ 定时器到期                        │
     ├─────────────────────────────────►│ VMExit
     │                                  │
     │ ◄─────────────────────────────────┤ 返回 Guest
     │                                  │
     │ CFS 调度器切换到 Worker1          │
     │                                  │
     ...                                ...
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

### 5. SpinNoIrq 的选择

`u_6_0` 使用 `SpinNoIrq` 而不是普通 `SpinLock`：

```rust
Arc<SpinNoIrq<VecDeque<usize>>>
```

原因：
- **SpinNoIrq**: 不会关中断，允许定时器中断触发任务切换
- **普通 SpinLock**: 可能会关中断，影响调度器工作

---

## 总结

`h_3_0` 虽然在代码层面与 `h_2_0` 完全相同，但其验证目标更加重要：

### 验证了虚拟化环境的核心能力

1. **定时器虚拟化**: 支持需要定时器中断的抢占式调度
2. **中断注入**: 正确地将 Host 中断转换为 Guest 虚拟中断
3. **多任务支持**: Guest 内部的线程调度与虚拟化环境兼容
4. **并发正确性**: 多线程共享数据在虚拟化环境下的正确性

### 演进路线

```
h_1_0: 基础 Hypervisor 实现
   │
   ├──> 只支持简单的 SBI 调用
   │
   ▼
h_2_0: 模块化 + 按需映射
   │
   ├──> 独立的 riscv_vcpu 模块
   ├──> 缺页异常处理
   ├──> 完整的 SBI 支持
   │
   ▼
h_3_0: 复杂 Guest 验证
   │
   ├──> 验证抢占式多任务支持
   ├──> 验证定时器中断虚拟化
   ├──> 验证并发执行正确性
   │
   ▼
未来: 多 vCPU、设备模拟、热迁移...
```

### 关键意义

`h_3_0` 证明了：
- 简单的 Hypervisor 实现 (h_2_0) 已经足够支持**复杂的 Guest 系统**
- **定时器中断虚拟化** 是抢占式调度的关键
- **模块化设计** (riscv_vcpu) 使得 Hypervisor 代码可以不经修改就支持不同的 Guest

这个版本展示了虚拟化技术的**通用性**——同一个 Hypervisor 可以支持从简单的设备访问程序到复杂的多任务操作系统。

### 关键代码位置

| 文件 | 行号 | 功能 |
|------|------|------|
| [h_3_0/src/main.rs](src/main.rs:28-82) | 28-82 | 主流程 (与 h_2_0 相同) |
| [u_6_0/src/main.rs](../u_6_0/src/main.rs:17-54) | 17-54 | 多任务生产者-消费者 |
| [riscv_vcpu/src/vcpu.rs](../../modules/riscv_vcpu/src/vcpu.rs:291-374) | 291-374 | VMExit 处理 |
| [riscv_vcpu/src/vcpu.rs](../../modules/riscv_vcpu/src/vcpu.rs:344-353) | 344-353 | 定时器中断处理 |

### 与 u_3_0 的对比

| 特性 | u_3_0 | u_6_0 |
|------|-------|-------|
| **代码行数** | ~25 行 | ~55 行 |
| **复杂度** | 简单 | 中等 |
| **线程数** | 1 | 3 |
| **同步** | 无 | Arc + SpinLock |
| **调度** | 无 | CFS 抢占式 |
| **验证重点** | 设备访问 | 定时器虚拟化 |

`h_3_0` 的价值在于：**用最小的 Hypervisor 代码变更，验证了最复杂的虚拟化场景**。
