# h_4_0 Hypervisor 代码分析文档

## 目录

1. [项目概述](#项目概述)
2. [与 h_2_0/h_3_0 的对比](#与-h_2_0h_3_0-的对比)
3. [项目结构](#项目结构)
4. [核心模块分析](#核心模块分析)
5. [虚拟设备管理 (vmdev.rs)](#虚拟设备管理-vmdevrs)
6. [Guest 系统分析 (m_1_1)](#guest-系统分析-m_1_1)
7. [数据流分析](#数据流分析)
8. [关键技术点](#关键技术点)
9. [总结](#总结)

---

## 项目概述

`h_4_0` 是 `h_2_0`/`h_3_0` 的重要升级版本，引入了**虚拟设备管理**框架，用于管理 Guest 的 MMIO 设备区域。这个版本使用 `m_1_1` 作为 Guest，这是一个**单体内核** (Monolithic Kernel)，可以加载和运行用户空间程序。

### 主要特性

- **虚拟设备管理**: 新增 `vmdev.rs` 模块，统一管理虚拟设备
- **灵活的 MMIO 处理**: 支持按需查找设备和处理 MMIO 访问
- **单体内核 Guest**: `m_1_1` 是一个简单的单体内核，支持：
  - 用户进程加载
  - 系统调用处理
  - 用户空间执行
- **从 pflash 加载用户程序**: Guest 从虚拟 flash 设备加载用户应用

### 技术验证

| 验证项 | 说明 |
|--------|------|
| **设备抽象** | 统一的虚拟设备管理接口 |
| **MMIO 按需映射** | 缺页时查找对应设备并处理 |
| **用户空间执行** | Guest 内核运行用户程序 |
| **双重虚拟化** | 用户空间 (VU) + Guest 内核 (VS) + Hypervisor (HS) |

### 技术栈

| 组件 | 说明 |
|------|------|
| **ArceOS** | 模块化操作系统框架 |
| **riscv_vcpu** | 独立的 vCPU 实现模块 |
| **RISC-V H 扩展** | 硬件虚拟化支持 |
| **axmm** | ArceOS 内存管理 |
| **axtask** | ArceOS 任务管理 |

---

## 与 h_2_0/h_3_0 的对比

### 代码变化

```
h_2_0 / h_3_0:
┌─────────────────────────────────────┐
│  main.rs                            │
│    ├── 缺页处理: 硬编码地址         │
│    │   assert_eq!(addr, 0x2200_0000)│
│    │   aspace.map_linear(...)       │
│    └── Guest: u_3_0 / u_6_0         │
└─────────────────────────────────────┘

h_4_0:
┌─────────────────────────────────────┐
│  main.rs + vmdev.rs                 │
│    ├── VmDevGroup: 设备管理器        │
│    ├── 缺页处理: 动态查找设备        │
│    │   dev = vmdevs.find_dev(addr)  │
│    │   dev.handle_mmio(...)         │
│    └── Guest: m_1_1 (单体内核)       │
└─────────────────────────────────────┘
```

### Guest 对比

| 特性 | u_6_0 (h_3_0 Guest) | m_1_1 (h_4_0 Guest) |
|------|---------------------|---------------------|
| **类型** | 多线程应用 | 单体内核 |
| **功能** | 生产者-消费者 | 加载并运行用户程序 |
| **用户空间** | 无 | **有** (从 pflash 加载) |
| **系统调用** | 无 | **有** (SYS_EXIT) |
| **地址空间** | 单一 | **内核 + 用户分离** |
| **特权级** | VS-mode | VS-mode + VU-mode |
| **依赖** | multitask + sched_cfs | axstd + axmm + axtask |

### 代码差异

```diff
--- h_3_0/src/main.rs
+++ h_4_0/src/main.rs

+ mod vmdev;

  fn main() {
      // ...
-     let image_fname = "/sbin/u_6_0_riscv64-qemu-virt.bin";
+     let image_fname = "/sbin/m_1_1_riscv64-qemu-virt.bin";
      load_vm_image(image_fname.to_string(), ...);
+
+     // Register pflash device into vm.
+     let mut vmdevs = VmDevGroup::new();
+     vmdevs.add_dev(0x2200_0000.into(), 0x200_0000);

      loop {
          match vcpu_run(&mut arch_vcpu) {
              NestedPageFault{addr, access_flags} => {
-                 assert_eq!(addr, 0x2200_0000.into());
-                 aspace.map_linear(addr, addr.as_usize().into(), 4096, flags);
+                 if addr < PHY_MEM_START.into() {
+                     let dev = vmdevs.find_dev(addr).expect("No dev.");
+                     dev.handle_mmio(addr, &mut aspace).unwrap();
+                 } else {
+                     unimplemented!("Handle #PF for memory region.");
+                 }
              }
          }
      }
  }
```

---

## 项目结构

```
h_4_0/
├── Cargo.toml              # 项目配置
├── src/
│   ├── main.rs             # Hypervisor 主逻辑
│   └── vmdev.rs            # 虚拟设备管理 (新增)

m_1_1/                      # h_4_0 的 Guest (单体内核)
├── Cargo.toml
├── src/
│   ├── main.rs             # 内核入口
│   ├── task.rs             # 任务管理 (用户进程)
│   ├── syscall.rs          # 系统调用处理
│   └── loader.rs           # 从 pflash 加载用户程序
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
        │  GPA: 0x8000_0000 → PA: 0x8000_0000  │
        └───────────────────────────────────────┘
                            │
                            ▼
        ┌───────────────────────────────────────┐
        │  load_vm_image("m_1_1_...bin")       │  加载 Guest 内核
        │  到 0x8020_0000                       │
        └───────────────────────────────────────┘
                            │
                            ▼
        ┌───────────────────────────────────────┐
        │  VmDevGroup::new()                   │  创建设备管理器
        │  vmdevs.add_dev(0x2200_0000, 32MB)   │  添加 pflash 设备
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
        │        vmdevs.find_dev()             │  查找设备
        │        dev.handle_mmio()             │  处理 MMIO
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
    let image_fname = "/sbin/m_1_1_riscv64-qemu-virt.bin";
    load_vm_image(image_fname.to_string(), KERNEL_BASE.into(), &aspace).expect("Failed to load VM images");

    // Register pflash device into vm.
    let mut vmdevs = VmDevGroup::new();
    vmdevs.add_dev(0x2200_0000.into(), 0x200_0000);

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
                    if addr < PHY_MEM_START.into() {
                        // Find dev and handle mmio region.
                        let dev = vmdevs.find_dev(addr).expect("No dev.");
                        dev.handle_mmio(addr, &mut aspace).unwrap();
                    } else {
                        unimplemented!("Handle #PF for memory region.");
                    }
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

### 2. 缺页处理流程

```
Guest 访问未映射地址
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                     硬件 G-Stage 遍历                           │
│  查找 GPA → 失败 (无映射)                                       │
└─────────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                     VMExit                                      │
│  scause = LoadGuestPageFault / StoreGuestPageFault              │
└─────────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Hypervisor (h_4_0)                         │
│  main.rs 主循环                                                 │
│    │                                                             │
│    │ 1. 判断地址类型                                             │
│    ▼                                                             │
│  if addr < PHY_MEM_START (0x8000_0000)                          │
│      → MMIO 设备区域                                             │
│  else                                                            │
│      → 内存区域 (未实现)                                         │
│    │                                                             │
│    ▼                                                             │
│  vmdevs.find_dev(addr)  → 查找对应的设备                         │
│    │                                                             │
│    ▼                                                             │
│  dev.handle_mmio(addr, &mut aspace)                             │
│    │                                                             │
│    ▼                                                             │
│  aspace.map_linear(addr, addr, 4096, flags)  → 透传映射          │
└─────────────────────────────────────────────────────────────────┘
```

### 3. 镜像加载

```rust
fn load_vm_image(image_path: String, image_load_gpa: VirtAddr, aspace: &AddrSpace) -> AxResult {
    use std::io::{BufReader, Read};
    let (image_file, image_size) = open_image_file(image_path.as_str())?;

    let image_load_regions = aspace
        .translated_byte_buffer(image_load_gpa, image_size)
        .expect("Failed to translate kernel image load address");
    let mut file = BufReader::new(image_file);

    for buffer in image_load_regions {
        file.read_exact(buffer).map_err(|err| {
            ax_err_type!(
                Io,
                format!("Failed in reading from file {}, err {:?}", image_path, err)
            )
        })?
    }

    Ok(())
}
```

---

## 虚拟设备管理 (vmdev.rs)

这是 `h_4_0` 的核心新增模块，提供了统一的虚拟设备管理接口。

### 结构定义

```rust
/// 单个虚拟设备
pub struct VmDev {
    start: VirtAddr,    // 设备起始地址
    size: usize,        // 设备大小
}

/// 设备组管理器
pub struct VmDevGroup {
    devices: Vec<Arc<VmDev>>  // 设备列表
}
```

### 接口实现

#### VmDev

```rust
impl VmDev {
    /// 创建新的虚拟设备
    pub fn new(start: VirtAddr, size: usize) -> Self {
        Self { start, size }
    }

    /// 处理 MMIO 访问 (透传模式)
    pub fn handle_mmio(&self, addr: VirtAddr, aspace: &mut AddrSpace) -> AxResult {
        let mapping_flags = MappingFlags::from_bits(0xf).unwrap();
        // GPA = PA (透传)
        aspace.map_linear(addr, addr.as_usize().into(), 4096, mapping_flags)
    }

    /// 检查地址是否属于此设备
    pub fn check_addr(&self, addr: VirtAddr) -> bool {
        addr >= self.start && addr < (self.start + self.size)
    }
}
```

#### VmDevGroup

```rust
impl VmDevGroup {
    /// 创建新的设备组
    pub fn new() -> Self {
        Self { devices: Vec::new() }
    }

    /// 添加虚拟设备
    pub fn add_dev(&mut self, addr: VirtAddr, size: usize) {
        let dev = VmDev::new(addr, size);
        self.devices.push(Arc::new(dev));
    }

    /// 查找包含指定地址的设备
    pub fn find_dev(&self, addr: VirtAddr) -> Option<Arc<VmDev>> {
        self.devices
            .iter()
            .find(|&dev| dev.check_addr(addr))
            .cloned()
    }
}
```

### 使用示例

```rust
// 创建设备管理器
let mut vmdevs = VmDevGroup::new();

// 添加 pflash 设备: 0x2200_0000, 大小 32MB
vmdevs.add_dev(0x2200_0000.into(), 0x200_0000);

// 可以添加更多设备...
vmdevs.add_dev(0x1000_0000.into(), 0x1000);  // UART
vmdevs.add_dev(0x2000_0000.into(), 0x10000); // 其他设备

// 缺页时查找设备
let dev = vmdevs.find_dev(fault_addr)?;
dev.handle_mmio(fault_addr, &mut aspace)?;
```

### 设备查找流程

```
缺页地址: 0x2200_1000
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                     VmDevGroup                                 │
│  devices: [                                                    │
│    VmDev { start: 0x2200_0000, size: 0x200_0000 },  ← pflash  │
│    VmDev { start: 0x1000_0000, size: 0x1000 },     ← UART    │
│  ]                                                             │
└─────────────────────────────────────────────────────────────────┘
        │
        │ 遍历设备列表
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│  VmDev[0].check_addr(0x2200_1000)                              │
│    0x2200_1000 >= 0x2200_0000 && 0x2200_1000 < 0x2400_0000     │
│    → true ✓                                                     │
└─────────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│  返回 Arc<VmDev>                                                │
└─────────────────────────────────────────────────────────────────┘
```

---

## Guest 系统分析 (m_1_1)

`m_1_1` 是一个**单体内核**，可以加载和运行用户空间程序。

### 架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        Guest (m_1_1)                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │              内核空间 (VS-mode)                          │   │
│   │  ┌───────────────────────────────────────────────────┐  │   │
│   │  │  main.rs                                          │  │   │
│   │  │    - 创建用户地址空间                             │  │   │
│   │  │    - 从 pflash 加载用户程序                        │  │   │
│   │  │    - 启动用户任务                                 │  │   │
│   │  └───────────────────────────────────────────────────┘  │   │
│   │  ┌───────────────────────────────────────────────────┐  │   │
│   │  │  task.rs: 任务管理                                │  │   │
│   │  │    - TaskExt: 任务扩展 (uctx, aspace)            │  │   │
│   │  │    - spawn_user_task(): 创建用户任务             │  │   │
│   │  └───────────────────────────────────────────────────┘  │   │
│   │  ┌───────────────────────────────────────────────────┐  │   │
│   │  │  syscall.rs: 系统调用处理                         │  │   │
│   │  │    - SYS_EXIT: 进程退出                           │  │   │
│   │  └───────────────────────────────────────────────────┘  │   │
│   │  ┌───────────────────────────────────────────────────┐  │   │
│   │  │  loader.rs: 程序加载器                            │  │   │
│   │  │    - load_pflash(): 从 pflash 读取 Payload        │  │   │
│   │  │    - load_user_app(): 加载到用户地址空间          │  │   │
│   │  └───────────────────────────────────────────────────┘  │   │
│   └─────────────────────────────────────────────────────────┘   │
│                            │                                      │
│                            │ 系统调用                             │
│                            ▼                                      │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │              用户空间 (VU-mode)                          │   │
│   │  ┌───────────────────────────────────────────────────┐  │   │
│   │  │  用户程序 (从 pflash 加载)                        │  │   │
│   │  │    - 入口: 0x1000                                  │  │   │
│   │  │    - 栈: 动态分配                                  │  │   │
│   │  │    - 系统调用: ecall → SYS_EXIT                   │  │   │
│   │  └───────────────────────────────────────────────────┘  │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
                            │
                            │ 缺页异常
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Hypervisor (h_4_0)                         │
│  • 设备管理 (VmDevGroup)                                         │
│  • MMIO 按需映射                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 主流程 ([main.rs](../m_1_1/src/main.rs))

```rust
const USER_STACK_SIZE: usize = 0x10000;
const KERNEL_STACK_SIZE: usize = 0x40000; // 256 KiB
const APP_ENTRY: usize = 0x1000;

fn main() {
    // 1. 创建用户地址空间
    let mut uspace = axmm::new_user_aspace().unwrap();

    // 2. 从 pflash 加载用户程序
    if let Err(e) = load_user_app(&mut uspace) {
        panic!("Cannot load app! {:?}", e);
    }

    // 3. 初始化用户栈
    let ustack_top = init_user_stack(&mut uspace, true).unwrap();
    ax_println!("New user address space: {:#x?}", uspace);

    // 4. 创建并启动用户任务
    let user_task = task::spawn_user_task(
        Arc::new(Mutex::new(uspace)),
        UspaceContext::new(APP_ENTRY.into(), ustack_top),  // 入口: 0x1000
    );

    // 5. 等待用户进程退出
    let exit_code = user_task.join();
    ax_println!("monolithic kernel exit [{:?}] normally!", exit_code);
}

fn init_user_stack(uspace: &mut AddrSpace, populating: bool) -> io::Result<VirtAddr> {
    let ustack_top = uspace.end();
    let ustack_vaddr = ustack_top - USER_STACK_SIZE;
    ax_println!(
        "Mapping user stack: {:#x?} -> {:#x?}",
        ustack_vaddr, ustack_top
    );
    uspace.map_alloc(
        ustack_vaddr,
        USER_STACK_SIZE,
        MappingFlags::READ | MappingFlags::WRITE | MappingFlags::USER,
        populating,
    ).unwrap();
    Ok(ustack_top)
}
```

### 程序加载器 ([loader.rs](../m_1_1/src/loader.rs))

`m_1_1` 从 pflash 加载用户程序，使用自定义的 Payload 格式：

#### Payload 格式

```
+-------------------+
| Magic (0x646C6670) |  4 bytes ("dlfp")
+-------------------+
| Version (0x01)     |  4 bytes
+-------------------+
| Size              |  4 bytes (用户程序大小)
+-------------------+
| Pad               |  4 bytes
+-------------------+
| User Code         |  Size bytes
+-------------------+
```

#### 核心代码

```rust
const PFLASH_START: usize = 0x2200_0000;
const MAGIC: u32 = 0x64_6C_66_70;  // "dlfp"
const VERSION: u32 = 0x01;

struct PayloadHead {
    _magic: u32,
    _version: u32,
    _size: u32,
    _pad: u32,
}

pub fn load_user_app(uspace: &mut AddrSpace) -> io::Result<()> {
    // 1. 从 pflash 读取 Payload
    let buf = load_pflash();

    // 2. 映射用户代码段 (0x1000)
    uspace.map_alloc(
        APP_ENTRY.into(),
        PAGE_SIZE_4K,
        MappingFlags::READ | MappingFlags::WRITE |
        MappingFlags::EXECUTE | MappingFlags::USER,
        true
    ).unwrap();

    // 3. 获取物理地址
    let (paddr, _, _) = uspace
        .page_table()
        .query(APP_ENTRY.into())
        .unwrap_or_else(|_| panic!("Mapping failed for segment: {:#x}", APP_ENTRY));

    ax_println!("paddr: {:#x}", paddr);

    // 4. 复制程序到用户内存
    unsafe {
        core::ptr::copy_nonoverlapping(
            buf.as_ptr(),
            phys_to_virt(paddr).as_mut_ptr(),
            buf.len(),
        );
    }

    Ok(())
}

pub fn load_pflash() -> Vec<u8> {
    let va = phys_to_virt(PFLASH_START.into());
    let data = va.as_usize() as *const u32;
    let data = unsafe {
        slice::from_raw_parts(data, mem::size_of::<PayloadHead>())
    };

    // 校验魔数和版本
    assert_eq!(data[0], MAGIC);
    assert_eq!(data[1].to_be(), VERSION);

    // 读取用户程序
    let size = data[2].to_be() as usize;
    let start = va + mem::size_of::<PayloadHead>();
    ax_println!("Pflash: start {:#X} size {}", start, size);

    let mut buf = vec![0u8; size];
    unsafe {
        core::ptr::copy_nonoverlapping(
            start.as_usize() as *const u8,
            buf.as_mut_ptr(),
            size,
        );
    }
    buf
}
```

### 任务管理 ([task.rs](../m_1_1/src/task.rs))

```rust
/// 任务扩展数据
pub struct TaskExt {
    pub proc_id: usize,
    pub uctx: UspaceContext,           // 用户空间上下文
    pub aspace: Arc<Mutex<AddrSpace>>, // 地址空间
}

impl TaskExt {
    pub const fn new(uctx: UspaceContext, aspace: Arc<Mutex<AddrSpace>>) -> Self {
        Self {
            proc_id: 1,
            uctx,
            aspace,
        }
    }
}

// 注册任务扩展
axtask::def_task_ext!(TaskExt);

pub fn spawn_user_task(aspace: Arc<Mutex<AddrSpace>>, uctx: UspaceContext) -> AxTaskRef {
    let mut task = TaskInner::new(
        || {
            let curr = axtask::current();
            let kstack_top = curr.kernel_stack_top().unwrap();

            ax_println!(
                "Enter user space: entry={:#x}, ustack={:#x}, kstack={:#x}",
                curr.task_ext().uctx.get_ip(),
                curr.task_ext().uctx.get_sp(),
                kstack_top,
            );

            // 进入用户空间
            unsafe { curr.task_ext().uctx.enter_uspace(kstack_top) };
        },
        "userboot".into(),
        KERNEL_STACK_SIZE,
    );

    // 设置页表根
    task.ctx_mut()
        .set_page_table_root(aspace.lock().page_table_root());

    task.init_task_ext(TaskExt::new(uctx, aspace));
    axtask::spawn_task(task)
}
```

### 系统调用处理 ([syscall.rs](../m_1_1/src/syscall.rs))

```rust
const SYS_EXIT: usize = 93;

#[register_trap_handler(SYSCALL)]
fn handle_syscall(tf: &TrapFrame, syscall_num: usize) -> isize {
    ax_println!("handle_syscall ...");
    let ret = match syscall_num {
        SYS_EXIT => {
            ax_println!("[SYS_EXIT]: process is exiting ..");
            axtask::exit(tf.arg0() as _)
        },
        _ => {
            ax_println!("Unimplemented syscall: {}", syscall_num);
            -LinuxError::ENOSYS.code() as _
        }
    };
    ret
}
```

---

## 数据流分析

### 启动流程

```
┌─────────────────────────────────────────────────────────────────┐
│                   Hypervisor (h_4_0)                            │
└─────────────────────────────────────────────────────────────────┘
                            │
                            │ 1. 初始化
                            ▼
        ┌───────────────────────────────────────┐
        │  setup_csrs()                         │
        │  - 委托异常/中断                      │
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
                            │ 4. 加载 Guest 内核 (m_1_1)
                            ▼
        ┌───────────────────────────────────────┐
        │  m_1_1 内核: 0x8020_0000              │
        └───────────────────────────────────────┘
                            │
                            │ 5. 注册虚拟设备
                            ▼
        ┌───────────────────────────────────────┐
        │  VmDevGroup                           │
        │  - pflash: 0x2200_0000, 32MB         │
        └───────────────────────────────────────┘
                            │
                            │ 6. 运行 Guest
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Guest (m_1_1 内核)                           │
└─────────────────────────────────────────────────────────────────┘
                            │
                            │ 7. 访问 pflash
                            ▼
        ┌───────────────────────────────────────┐
        │  load_pflash()                        │
        │  - 读取 0x2200_0000                   │
        └───────────────────────────────────────┘
                            │
                            │ 8. 缺页异常
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Hypervisor (h_4_0)                            │
│  vmdevs.find_dev(0x2200_0000) → 找到 pflash                      │
│  dev.handle_mmio() → 映射 GPA=PA                                │
└─────────────────────────────────────────────────────────────────┘
                            │
                            │ 9. 返回 Guest
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Guest (m_1_1 内核)                           │
│  解析 Payload → 加载用户程序 → 启动用户任务                      │
└─────────────────────────────────────────────────────────────────┘
```

### MMIO 按需映射流程

```
Guest 访问 pflash
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Guest (m_1_1)                               │
│  执行 load_pflash()                                             │
│    va = phys_to_virt(0x2200_0000)                              │
│    data = *va  ← 访问未映射地址                                 │
└─────────────────────────────────────────────────────────────────┘
                            │
                            │ 触发缺页异常
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     硬件 G-Stage                                │
│  遍历页表: GPA 0x2200_0000 → 无映射                             │
│  触发: LoadGuestPageFault                                       │
└─────────────────────────────────────────────────────────────────┘
                            │
                            │ 世界切换
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Hypervisor (h_4_0)                            │
│  vmexit_handler() 返回 NestedPageFault                           │
│    │                                                             │
│    ▼                                                             │
│  main.rs 主循环:                                                │
│    │                                                             │
│    │ 1. 判断地址类型                                            │
│    ▼                                                             │
│  addr (0x2200_0000) < PHY_MEM_START (0x8000_0000) → MMIO       │
│    │                                                             │
│    │ 2. 查找设备                                                │
│    ▼                                                             │
│  vmdevs.find_dev(0x2200_0000)                                   │
│    │                                                             │
│    │ 3. 遍历设备列表                                            │
│    ▼                                                             │
│  VmDev { start: 0x2200_0000, size: 0x200_0000 }                │
│    │                                                             │
│    │ 4. 检查地址范围                                            │
│    ▼                                                             │
│  check_addr(0x2200_0000) → true                                 │
│    │                                                             │
│    │ 5. 处理 MMIO                                               │
│    ▼                                                             │
│  handle_mmio(0x2200_0000, &mut aspace)                          │
│    │                                                             │
│    │ 6. 透传映射 GPA = PA                                       │
│    ▼                                                             │
│  aspace.map_linear(                                             │
│      GPA: 0x2200_0000,                                          │
│      PA: 0x2200_0000,                                           │
│      size: 4096,                                                │
│      flags: RWXU                                                │
│  )                                                              │
│    │                                                             │
│    ▼                                                             │
│  更新 G-Stage 页表, 刷新 TLB                                     │
└─────────────────────────────────────────────────────────────────┘
                            │
                            │ 下次访问同一页不会缺页
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Guest (m_1_1)                               │
│  继续执行 load_pflash()                                         │
│  成功读取 pflash 数据                                           │
└─────────────────────────────────────────────────────────────────┘
```

### 用户程序加载流程

```
Guest 内核 (m_1_1)
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│  load_user_app(&mut uspace)                                     │
│    │                                                             │
│    │ 1. 从 pflash 读取 Payload                                   │
│    ▼                                                             │
│  load_pflash()                                                  │
│    │                                                             │
│    │ 1.1 读取头部                                               │
│    ▼                                                             │
│  data[0] = MAGIC (0x646C6670)  ✓                                │
│  data[1] = VERSION (0x01)         ✓                                │
│  data[2] = SIZE (用户程序大小)                                   │
│    │                                                             │
│    │ 1.2 读取用户程序                                           │
│    ▼                                                             │
│  buf = vec![0u8; SIZE]                                          │
│  copy from pflash (0x2200_0000 + 16) to buf                    │
│    │                                                             │
│    │ 2. 映射用户代码段                                           │
│    ▼                                                             │
│  uspace.map_alloc(                                              │
│      GPA: 0x1000,                                               │
│      size: PAGE_SIZE_4K,                                        │
│      flags: RWXU                                                │
│  )                                                              │
│    │                                                             │
│    │ 3. 复制程序到用户内存                                       │
│    ▼                                                             │
│  paddr = query(0x1000)                                          │
│  copy from buf to phys_to_virt(paddr)                          │
│    │                                                             │
│    ▼                                                             │
│  Ok(())                                                         │
└─────────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│  spawn_user_task(aspace, uctx)                                   │
│    │                                                             │
│    │ 1. 创建任务                                                │
│    ▼                                                             │
│  TaskInner::new(                                                │
│      closure: || { enter_uspace() },                            │
│      name: "userboot",                                          │
│      kstack_size: 256KB                                         │
│  )                                                              │
│    │                                                             │
│    │ 2. 设置页表根                                              │
│    ▼                                                             │
│  task.ctx_mut().set_page_table_root(uspace.page_table_root())   │
│    │                                                             │
│    │ 3. 初始化任务扩展                                          │
│    ▼                                                             │
│  task.init_task_ext(TaskExt {                                    │
│      proc_id: 1,                                                │
│      uctx: UspaceContext {                                      │
│          ip: 0x1000,                                            │
│          sp: ustack_top,                                        │
│      },                                                         │
│      aspace,                                                    │
│  })                                                             │
│    │                                                             │
│    ▼                                                             │
│  spawn_task(task)                                               │
└─────────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│  任务调度运行                                                    │
│    │                                                             │
│    │ 1. 调度器选择任务                                          │
│    ▼                                                             │
│  执行任务闭包                                                   │
│    │                                                             │
│    │ 2. 进入用户空间                                            │
│    ▼                                                             │
│  unsafe { uctx.enter_uspace(kstack_top) }                       │
│    │                                                             │
│    │ 3. 切换到用户模式                                          │
│    ▼                                                             │
│  sret → VU-mode, entry=0x1000                                   │
└─────────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                    用户程序 (VU-mode)                           │
│  执行用户代码                                                   │
│    │                                                             │
│    │ 系统调用                                                    │
│    ▼                                                             │
│  ecall (SYS_EXIT)                                               │
└─────────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Guest 内核 (VS-mode)                         │
│  handle_syscall(SYS_EXIT)                                       │
│    │                                                             │
│    ▼                                                             │
│  axtask::exit(exit_code)                                        │
│    │                                                             │
│    ▼                                                             │
│  任务退出                                                       │
└─────────────────────────────────────────────────────────────────┘
```

### 特权级切换

`m_1_1` 展示了完整的三层特权级：

```
┌─────────────────────────────────────────────────────────────────┐
│                  M-mode (机器模式)                              │
│                  OpenSBI 固件                                   │
├─────────────────────────────────────────────────────────────────┤
│                  HS-mode (Hypervisor)                           │
│                  h_4_0 Hypervisor                               │
│                    ┌─────────────────────────────────────────┐  │
│                    │  设备管理 (VmDevGroup)                  │  │
│                    │  MMIO 按需映射                          │  │
│                    │  VMExit 处理                            │  │
│                    └─────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                  VS-mode (Guest 内核)                           │
│                  m_1_1 Monolithic Kernel                        │
│                    ┌─────────────────────────────────────────┐  │
│                    │  内存管理                               │  │
│                    │  任务调度                               │  │
│                    │  系统调用处理                           │  │
│                    │  用户程序加载                           │  │
│                    └─────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                  VU-mode (用户程序)                             │
│                  从 pflash 加载的用户程序                        │
│                    ┌─────────────────────────────────────────┐  │
│                    │  用户代码执行                           │  │
│                    │  系统调用 (ecall)                       │  │
│                    └─────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

切换流程：

1. **HS → VS**: `sret` 从 Hypervisor 进入 Guest 内核
   - `hstatus.SPV = 1` 确保 VS-mode
   - `sstatus.SPP = S` 确保 VS-mode (非 VU-mode)

2. **VS → VU**: `sret` 从内核进入用户空间
   - 修改 `sstatus.SPP = U`
   - 设置 `sepc` 为用户入口 (0x1000)
   - `sret` 进入 VU-mode

3. **VU → VS**: `ecall` (系统调用) 从用户空间进入内核
   - `scause` = Exception::EnvCallFromUOrVU
   - 内核处理系统调用

4. **VS → HS**: 异常/中断触发 VMExit
   - 缺页: LoadGuestPageFault / StoreGuestPageFault
   - 定时器: SupervisorTimer
   - Hypervisor 处理后返回

---

## 关键技术点

### 1. 设备抽象设计

`h_4_0` 的 `VmDev` 提供了设备无关的抽象：

```rust
// 当前实现: 透传模式
pub fn handle_mmio(&self, addr: VirtAddr, aspace: &mut AddrSpace) -> AxResult {
    // GPA = PA (直接映射到物理设备)
    aspace.map_linear(addr, addr.as_usize().into(), 4096, flags)
}

// 可能的扩展:

// 模拟模式
pub fn handle_mmio_emulated(&self, addr: VirtAddr, aspace: &mut AddrSpace) -> AxResult {
    // 分配内存并填充模拟数据
    aspace.map_alloc(addr, 4096, flags, true);
    aspace.write(addr, b"Hello from emulated device!");
}

// 半虚拟化模式
pub fn handle_mmio_pv(&self, addr: VirtAddr, aspace: &mut AddrSpace) -> AxResult {
    // 通知 Guest 使用 hypercall
    inject_hypercall(HYPERCALL_DEVICE_IO, addr);
}
```

### 2. 地址空间分层

`m_1_1` 使用两个独立的地址空间：

```
Hypervisor 视角 (G-Stage):
┌─────────────────────────────────────────────────────────────────┐
│  hgatp: G-Stage 页表基址                                        │
│                                                                 │
│  GPA 范围         →        PA                                   │
│  ─────────────────────────────────────────                      │
│  0x8020_0000      →   xxxxx (Guest 内核代码)                    │
│  0x2200_0000      →   0x2200_0000 (pflash 设备, 按需映射)       │
│  0x1000           →   xxxxx (用户代码)                          │
│  (用户栈)         →   xxxxx (用户栈)                            │
└─────────────────────────────────────────────────────────────────┘

Guest 内核视角 (VS-Stage):
┌─────────────────────────────────────────────────────────────────┐
│  vsatp: VS-Stage 页表基址                                       │
│                                                                 │
│  GVA (用户)      →        GPA                                   │
│  ─────────────────────────────────────────                      │
│  0x1000           →   0x1000 (用户代码, 直接映射)                │
│  (用户栈)         →   (用户栈 GPA, 直接映射)                     │
│                                                                 │
│  GVA (内核)      →        GPA                                   │
│  ─────────────────────────────────────────                      │
│  内核代码段       →   0x8020_0000 (内核代码)                    │
│  内核数据段       →   其他 GPA                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 3. Payload 格式

pflash 中的用户程序使用自定义格式：

```
地址偏移    字段            说明
────────────────────────────────────────────────
0x00        Magic          0x646C6670 ("dlfp")
0x04        Version        0x01
0x08        Size           用户程序大小 (字节)
0x0C        Pad            保留
0x10        User Code      用户程序代码
```

代码校验：

```rust
// 大端序读取
assert_eq!(data[1].to_be(), VERSION);  // to_be() 转换为大端序

// 直接读取
assert_eq!(data[0], MAGIC);  // 小端序
```

### 4. 用户入口点

用户程序从 `0x1000` 开始：

```rust
const APP_ENTRY: usize = 0x1000;

UspaceContext::new(APP_ENTRY.into(), ustack_top)
```

这是一个约定的低地址，便于内核管理：

- **低地址**: 避免与内核地址空间冲突
- **页对齐**: 0x1000 是 4KB 对齐的
- **易于管理**: 内核可以统一管理所有用户程序

### 5. 代码与数据分离

`m_1_1` 内核和用户程序完全分离：

```
内存布局:
┌─────────────────────────────────────────────────────────────────┐
│  0x8020_0000                                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              m_1_1 内核代码                              │   │
│  │  - main.rs, task.rs, syscall.rs, loader.rs              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  0x1000                                                         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              用户程序代码 (VSATP 映射)                  │   │
│  │  - 从 pflash 加载                                       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  (动态地址)                                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              用户栈                                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  0x2200_0000                                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              pflash 设备 (按需映射)                     │   │
│  │  - Payload 头部                                         │   │
│  │  - 用户程序数据                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 总结

`h_4_0` 引入了虚拟设备管理框架，是一个重要的架构升级：

### 主要改进

1. **设备抽象**: `VmDev` 和 `VmDevGroup` 提供统一的设备管理接口
2. **灵活查找**: 缺页时动态查找设备，而非硬编码地址
3. **单体内核 Guest**: `m_1_1` 演示了完整的操作系统功能
4. **用户空间支持**: Guest 内核可以加载和运行用户程序

### 技术验证

`h_4_0` 验证了：
- **虚拟设备管理**的可行性
- **MMIO 按需映射**的灵活性
- **单体内核**在虚拟化环境下的运行
- **用户空间**通过系统调用与内核交互
- **三层特权级** (M/HS/VS/VU) 的完整切换

### 演进路线

```
h_1_0: 基础 Hypervisor
   │
   ├──> 单体代码
   ├──> 仅支持 SBI 调用
   │
   ▼
h_2_0: 模块化 + 按需映射
   │
   ├──> riscv_vcpu 独立模块
   ├──> 缺页处理 (硬编码)
   ├──> 完整的 SBI 支持
   │
   ▼
h_3_0: 复杂 Guest 验证
   │
   ├──> 多任务 Guest
   ├──> 定时器虚拟化
   │
   ▼
h_4_0: 虚拟设备管理
   │
   ├──> VmDev 框架
   ├──> 动态设备查找
   ├──> 单体内核 Guest
   ├──> 用户空间支持
   │
   ▼
未来: 多 vCPU、完整设备模拟 (UART、磁盘、网络)...
```

### 关键代码位置

| 文件 | 行号 | 功能 |
|------|------|------|
| [h_4_0/src/main.rs](src/main.rs:27-84) | 27-84 | 主流程 |
| [h_4_0/src/main.rs](src/main.rs:61-73) | 61-73 | 缺页处理 |
| [h_4_0/src/vmdev.rs](src/vmdev.rs:8-49) | 8-49 | 设备管理 |
| [m_1_1/src/main.rs](../m_1_1/src/main.rs:29-51) | 29-51 | 内核主流程 |
| [m_1_1/src/loader.rs](../m_1_1/src/loader.rs:21-42) | 21-42 | 程序加载器 |
| [m_1_1/src/task.rs](../m_1_1/src/task.rs:30-50) | 30-50 | 任务管理 |
| [m_1_1/src/syscall.rs](../m_1_1/src/syscall.rs:9-23) | 9-23 | 系统调用 |

`h_4_0` 展示了如何从简单的透传设备管理，逐步演化为完整的虚拟设备框架，为后续更复杂的设备模拟（如 UART、磁盘、网络等）打下了基础。
