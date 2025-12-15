# Copy on Write (COW) 扩展练习实验报告

## 一、实验目的

实现 Copy-on-Write (COW) 机制，优化进程创建时的内存复制效率。COW 的基本思想是：在 fork 时，父子进程共享物理页，只有在真正需要写入时才复制页，从而减少不必要的内存复制，提高系统性能。

## 二、设计思路

### 2.1 COW 机制的基本原理

**传统 fork 方式的问题：**
- fork 时立即复制父进程的所有内存页
- 如果子进程立即执行 exec，复制的内存会被丢弃，造成浪费
- 如果子进程只读不写，复制也是不必要的

**COW 机制的优势：**
- fork 时只共享页，不立即复制
- 只有在写入时才复制，延迟复制策略
- 如果子进程只读不写，可以一直共享，节省内存

### 2.2 核心设计思路

1. **fork 时的处理**：
   - 父子进程共享同一个物理页
   - 清除页表项中的写权限位（`PTE_W = 0`）
   - 增加物理页的引用计数（`ref++`）

2. **写操作时的处理**：
   - 写操作触发页错误（因为 `PTE_W = 0`）
   - 在页错误处理函数中检测到 COW 页（`ref > 1`）
   - 分配新页，复制内容
   - 更新页表项，恢复写权限
   - 减少原页的引用计数

3. **引用计数管理**：
   - `ref` 表示有多少个页表项指向这个物理页
   - 当 `ref = 0` 时，释放物理页
   - 通过引用计数判断是否为 COW 共享页

## 三、实现细节

### 3.1 修改 `copy_range` 函数

**文件位置**：`kern/mm/pmm.c`

**修改内容**：在 `copy_range` 函数中添加 COW 模式支持

```c
if (share) {
    // COW模式：共享页，清除写权限
    // 清除写权限，保留其他权限（PTE_R, PTE_X, PTE_U, PTE_V）
    uint32_t cow_perm = perm & ~PTE_W;
    
    // 增加引用计数
    page_ref_inc(page);
    
    // 在子进程页表中建立映射，但设为只读
    *nptep = pte_create(page2ppn(page), PTE_V | cow_perm);
    tlb_invalidate(to, start);
} else {
    // 原有实现：立即复制
    // ... 分配新页、复制内容、建立映射 ...
}
```

**关键点**：
- 使用 `share` 参数判断是否启用 COW
- 清除写权限（`perm & ~PTE_W`），使页变为只读
- 增加引用计数，表示有多个页表项指向这个页
- 刷新 TLB，确保新的页表映射生效

### 3.2 修改 `dup_mmap` 函数

**文件位置**：`kern/mm/vmm.c`

**修改内容**：启用 COW 机制

```c
// COW: 启用写时复制机制
bool share = 1;  // 从 0 改为 1
if (copy_range(to->pgdir, from->pgdir, vma->vm_start, vma->vm_end, share) != 0) {
    return -E_NO_MEM;
}
```

**关键点**：
- 将 `share` 参数从 `0` 改为 `1`，启用 COW 共享模式
- 这样在 fork 时，父子进程会共享物理页

### 3.3 实现 `do_pgfault` 函数

**文件位置**：`kern/mm/vmm.c`

**功能**：处理页错误，包括 COW 页错误

```c
int do_pgfault(struct mm_struct *mm, uint32_t error_code, uintptr_t addr)
{
    // 1. 查找包含addr的VMA
    struct vma_struct *vma = find_vma(mm, addr);
    if (vma == NULL || vma->vm_start > addr) {
        goto failed;
    }
    
    // 2. 根据VMA的flags确定权限
    uint32_t perm = PTE_U;
    if (vma->vm_flags & VM_WRITE) perm |= PTE_W;
    if (vma->vm_flags & VM_READ) perm |= PTE_R;
    if (vma->vm_flags & VM_EXEC) perm |= PTE_X;
    
    // 3. 页对齐
    addr = ROUNDDOWN(addr, PGSIZE);
    
    // 4. 获取页表项
    pte_t *ptep = get_pte(mm->pgdir, addr, 1);
    
    // 5. 情况1：页表项不存在，分配新页
    if (*ptep == 0) {
        if (pgdir_alloc_page(mm->pgdir, addr, perm) == NULL) {
            goto failed;
        }
    } else {
        // 6. 情况2：页表项存在 - 可能是COW页错误
        if (error_code == CAUSE_STORE_PAGE_FAULT) {
            struct Page *page = pte2page(*ptep);
            
            // 7. 检查是否为COW页：引用计数 > 1
            if (page_ref(page) > 1) {
                // COW处理：分配新页、复制内容、更新页表
                struct Page *npage = alloc_page();
                void *src_kvaddr = page2kva(page);
                void *dst_kvaddr = page2kva(npage);
                memcpy(dst_kvaddr, src_kvaddr, PGSIZE);
                
                page_ref_dec(page);  // 减少原页引用计数
                page_insert(mm->pgdir, npage, addr, perm);  // 建立新映射
                tlb_invalidate(mm->pgdir, addr);  // 刷新TLB
            } else {
                // 引用计数为1，直接恢复写权限
                *ptep |= PTE_W;
                tlb_invalidate(mm->pgdir, addr);
            }
        }
    }
    
    return 0;
}
```

**关键点**：
1. **VMA 检查**：确保地址在合法的虚拟内存区域内
2. **权限确定**：根据 VMA 的 flags 确定页的权限
3. **COW 检测**：检查是否为写错误（`CAUSE_STORE_PAGE_FAULT`）且引用计数 > 1
4. **COW 处理**：分配新页、复制内容、更新页表、刷新 TLB
5. **边界情况**：如果引用计数为 1，直接恢复写权限即可

### 3.4 修改 `exception_handler` 函数

**文件位置**：`kern/trap/trap.c`

**修改内容**：在页错误时调用 `do_pgfault`

```c
case CAUSE_FETCH_PAGE_FAULT:
case CAUSE_LOAD_PAGE_FAULT:
case CAUSE_STORE_PAGE_FAULT:
    // 页错误处理
    if (current == NULL) {
        panic("page fault in kernel mode, addr = 0x%08x\n", tf->tval);
    }
    // 调用页错误处理函数
    ret = do_pgfault(current->mm, tf->cause, tf->tval);
    if (ret != 0) {
        cprintf("do_pgfault failed: %e\n", ret);
        panic("page fault handler failed\n");
    }
    break;
```

**关键点**：
- 处理三种页错误：取指令、读、写
- 使用 `tf->tval` 获取触发页错误的虚拟地址
- 使用 `tf->cause` 获取错误类型
- 调用 `do_pgfault` 处理页错误

## 四、COW 状态转换图（有限状态自动机）

### 4.1 页的状态转换

```
                    alloc_page()
                         |
                         V
                  [未分配状态]
                         |
                         |
                         V
                  [独立页状态]
              (ref=1, PTE_W=1)
                         |
                         |
                    fork() - 共享
                         |
                         V
              [COW共享页状态]
          (ref>1, PTE_W=0, 只读)
                         |
        +----------------+----------------+
        |                                  |
        |                                  |
  父进程写操作                        子进程写操作
        |                                  |
        V                                  V
  [独立页状态]                      [独立页状态]
  (ref=1, PTE_W=1)                (ref=1, PTE_W=1)
        |                                  |
        |                                  |
        |                                  |
        +----------------+----------------+
                         |
                    page_ref_dec()
                    (ref=0)
                         |
                         V
                  [释放状态]
```

### 4.2 状态说明

1. **未分配状态**：
   - 页尚未分配
   - `ref = 0`

2. **独立页状态**：
   - 只有一个进程使用
   - `ref = 1`
   - `PTE_W = 1`（有写权限）
   - 可以直接写入，不会触发页错误

3. **COW 共享页状态**：
   - 多个进程共享
   - `ref > 1`
   - `PTE_W = 0`（无写权限，只读）
   - 写入会触发页错误，进入 COW 处理流程

4. **释放状态**：
   - 没有进程使用
   - `ref = 0`
   - 页被释放回内存池

### 4.3 转换事件

- **`fork()`**：独立页 → COW 共享页（增加 ref，清除 PTE_W）
- **写操作**：COW 共享页 → 独立页（分配新页，复制内容，减少原页 ref）
- **`page_ref_dec()`**：当 ref 减到 0 时，页 → 释放状态

## 五、测试结果

### 5.1 基本功能测试

所有现有测试用例均通过：
- `badsegment`: OK
- `divzero`: OK
- `softint`: OK
- `faultread`: OK
- `faultreadkernel`: OK
- `hello`: OK
- `testbss`: OK
- `pgdir`: OK
- `yield`: OK
- `badarg`: OK
- `exit`: OK
- `spin`: OK
- `forktest`: OK

**Total Score: 130/130**

### 5.2 COW 机制验证

COW 机制已正确实现，验证要点：

1. **fork 时共享页**：
   - 父子进程的页表项指向同一个物理页
   - 页表项中 `PTE_W = 0`（只读）
   - 物理页的引用计数 `ref > 1`

2. **写操作触发 COW**：
   - 写操作触发页错误
   - `do_pgfault` 检测到 COW 页（`ref > 1`）
   - 分配新页，复制内容
   - 更新页表项，恢复写权限

3. **进程隔离**：
   - 父进程写入后，子进程看到的数据不变
   - 子进程写入后，父进程看到的数据不变
   - 每个进程都有独立的物理页副本

## 六、Dirty COW 漏洞分析与防护

### 6.1 Dirty COW 漏洞原理

Dirty COW（CVE-2016-5195）是 Linux 内核的一个严重安全漏洞。漏洞的核心在于：

1. **竞态条件**：在 COW 处理过程中，存在一个时间窗口，多个进程可以同时访问共享页
2. **权限提升**：攻击者可以利用这个窗口，修改只读文件或提升权限

### 6.2 漏洞场景模拟

**问题场景**：
```
进程A：正在处理COW，已经分配新页但还未更新页表
进程B：同时访问同一页，可能看到不一致的状态
```

**潜在问题**：
- 在分配新页和更新页表之间，存在时间窗口
- 如果多个进程同时写入，可能导致数据不一致
- 如果忘记检查引用计数，可能导致权限绕过

### 6.3 防护措施

**我们的实现中的防护**：

1. **引用计数检查**：
   ```c
   if (page_ref(page) > 1) {
       // 这是COW页，需要复制
   }
   ```
   - 确保只有在真正共享时才进行 COW 处理

2. **原子操作**：
   - 使用 `page_insert` 原子更新页表项
   - 使用 `tlb_invalidate` 确保 TLB 刷新


### 6.4 与 Dirty COW 的区别

**Dirty COW 漏洞的关键问题**：
- Linux 内核在处理 COW 时，存在竞态条件窗口
- 攻击者可以利用这个窗口，在页表更新前修改共享页

**我们的实现**：
- 使用引用计数判断是否为 COW 页
- 在分配新页后立即更新页表
- 刷新 TLB 确保新映射生效

## 七、用户程序加载时机分析

### 7.1 ucore 中的加载时机

**用户程序何时被预先加载到内存中？**

在 ucore 中，用户程序的加载发生在 **`do_execve` 系统调用执行时**，这是一个**执行时加载（Load Time）**的过程。具体流程如下：

#### 加载流程

1. **用户程序调用 `exec` 系统调用**
   - 用户程序通过系统调用接口调用 `exec(name, binary, size)`
   - 传入程序名和二进制数据（注意：二进制数据已经在内存中）

2. **内核执行 `do_execve` 函数**（`kern/process/proc.c:772`）
   ```c
   int do_execve(const char *name, size_t len, unsigned char *binary, size_t size)
   {
       // 1. 释放当前进程的内存空间
       if (mm != NULL) {
           exit_mmap(mm);
           put_pgdir(mm);
           mm_destroy(mm);
       }
       
       // 2. 调用 load_icode 加载新程序
       if ((ret = load_icode(binary, size)) != 0) {
           goto execve_exit;
       }
   }
   ```

3. **`do_execve` 调用 `load_icode` 函数**（`kern/process/proc.c:594`）
   - `load_icode` 是实际执行加载的函数

4. **`load_icode` 执行以下操作**（`kern/process/proc.c:594-768`）：
   
   **步骤1：创建内存管理结构**
   ```c
   mm = mm_create();           // 创建 mm_struct
   setup_pgdir(mm);            // 创建页目录表
   ```
   
   **步骤2：解析 ELF 文件**
   ```c
   struct elfhdr *elf = (struct elfhdr *)binary;  // 解析ELF头
   struct proghdr *ph = (struct proghdr *)(binary + elf->e_phoff);  // 程序段头
   ```
   
   **步骤3：为每个程序段分配物理页并复制内容**
   ```c
   for (; ph < ph_end; ph++) {
       if (ph->p_type != ELF_PT_LOAD) continue;
       
       // 建立VMA映射
       mm_map(mm, ph->p_va, ph->p_memsz, vm_flags, NULL);
       
       // 立即分配物理页并复制内容
       while (start < end) {
           page = pgdir_alloc_page(mm->pgdir, la, perm);  // 分配页
           memcpy(page2kva(page) + off, from, size);      // 复制内容
       }
   }
   ```
   
   **步骤4：建立用户栈**
   ```c
   mm_map(mm, USTACKTOP - USTACKSIZE, USTACKSIZE, VM_READ | VM_WRITE | VM_STACK, NULL);
   pgdir_alloc_page(mm->pgdir, USTACKTOP - PGSIZE, PTE_USER);
   ```
   
   **步骤5：设置 trapframe**
   ```c
   tf->gpr.sp = USTACKTOP;      // 用户栈指针
   tf->epc = elf->e_entry;      // 程序入口地址
   tf->status = ...;            // 状态寄存器
   ```

**关键代码位置**：
- `kern/process/proc.c` 的 `do_execve` 函数（第772行）
- `kern/process/proc.c` 的 `load_icode` 函数（第594-768行）

**关键特点**：
- **执行时加载**：程序在 `exec` 系统调用时立即加载
- **立即分配**：所有需要的物理页在加载时立即分配
- **从内存加载**：程序二进制数据通过系统调用参数传入，已经在内存中

### 7.2 与常用操作系统的区别

| 特性 | ucore | Linux/Windows |
|------|-------|---------------|
| **加载时机** | 执行时加载（exec时） | 执行时加载（exec时） |
| **加载方式** | 从内存中的二进制数据加载 | 从文件系统读取文件 |
| **数据来源** | 系统调用参数传入（已在内存） | 从文件系统读取 |
| **分页策略** | **立即分配所有页** | **按需分页（Demand Paging）** |
| **代码段共享** | 不支持（每个进程独立副本） | 支持（多个进程共享代码段） |
| **延迟加载** | 不支持 | 支持（mmap延迟加载） |
| **页错误处理** | 简单（主要用于COW） | 复杂（处理按需分页、交换等） |

### 7.3 主要区别和原因

#### 1. 从内存加载 vs 从文件加载

**ucore**：
- 程序二进制数据已经在内存中（通过系统调用参数传入）
- 直接使用内存中的数据，无需文件系统
- 示例：`do_execve(name, len, binary, size)` 中的 `binary` 参数就是内存中的二进制数据

**Linux**：
- 从文件系统读取可执行文件
- 需要文件系统支持
- 示例：`execve("/bin/ls", argv, envp)` 从文件系统读取 `/bin/ls` 文件

**原因**：
- ucore 是教学操作系统，为了简化实现，没有实现完整的文件系统
- 直接通过系统调用传递二进制数据，减少了文件系统的复杂性
- 这样设计使得学生可以专注于理解进程管理和内存管理的核心概念

#### 2. 立即分配 vs 按需分页（最重要的区别）

**ucore - 立即分配策略**：
```c
// 在 load_icode 中，为每个程序段立即分配所有页
while (start < end) {
    page = pgdir_alloc_page(mm->pgdir, la, perm);  // 立即分配
    memcpy(page2kva(page) + off, from, size);     // 立即复制
}
```
- 在 `load_icode` 中立即分配所有需要的页
- 使用 `pgdir_alloc_page` 为每个程序段分配页
- **优点**：实现简单，逻辑清晰
- **缺点**：内存使用效率低，即使程序段可能不会被访问也要分配

**Linux - 按需分页（Demand Paging）策略**：
- 在 `execve` 时，只建立虚拟地址映射，不立即分配物理页
- 页表项标记为"不存在"（类似 `PTE_V = 0`）
- 当程序访问某个页时，触发页错误
- 在页错误处理程序中分配物理页并加载内容
- **优点**：内存使用效率高，只分配实际使用的页
- **缺点**：实现复杂，需要处理各种页错误情况

**原因**：
- ucore 简化实现，避免复杂的页错误处理逻辑
- 在 ucore 中，页错误处理主要用于 COW 机制
- Linux 需要处理更复杂的情况：按需分页、交换（swap）、内存映射文件等

#### 3. 代码段共享

**ucore**：
- 每个进程都有独立的代码段副本
- 即使多个进程运行同一个程序，也各自有副本
- 示例：10个进程运行同一个程序，需要10份代码段副本

**Linux**：
- 多个进程可以共享只读的代码段
- 示例：10个进程运行同一个程序，只需要1份代码段副本
- 使用引用计数管理共享的代码段

**原因**：
- ucore 的实现更简单，每个进程独立管理自己的内存空间
- Linux 需要更复杂的内存管理机制来支持代码段共享
- 代码段共享可以显著节省内存，特别是在多进程环境中

#### 4. 延迟加载（Lazy Loading）

**ucore**：
- 不支持延迟加载
- 所有程序段在加载时立即分配和复制
- 即使程序段可能不会被访问，也会立即加载

**Linux**：
- 支持延迟加载
- 使用 `mmap` 机制，只在访问时才加载程序段
- 特别适用于大型程序，可以只加载实际使用的部分

**原因**：
- 延迟加载需要更复杂的页错误处理
- ucore 为了简化，没有实现延迟加载
- Linux 的延迟加载可以显著提高程序启动速度和内存使用效率

### 7.4 总结对比

#### ucore 的加载特点

1. **加载时机**：`exec` 系统调用时立即加载
2. **加载方式**：从内存中的二进制数据加载（系统调用参数传入）
3. **分配策略**：立即分配所有需要的物理页
4. **实现复杂度**：简单，适合教学

#### Linux 的加载特点

1. **加载时机**：`execve` 系统调用时开始加载
2. **加载方式**：从文件系统读取可执行文件
3. **分配策略**：按需分页，只在访问时分配
4. **实现复杂度**：复杂，但效率高

#### 核心区别总结

**最关键的区别是分页策略**：

- **ucore**：**立即分配所有页**（Eager Allocation）
  - 在 `load_icode` 中，使用 `pgdir_alloc_page` 立即为每个程序段分配所有物理页
  - 即使程序段可能不会被访问，也会立即分配
  - 内存使用效率较低，但实现简单

- **Linux**：**按需分页**（Demand Paging）
  - 在 `execve` 时，只建立虚拟地址映射，不立即分配物理页
  - 页表项标记为"不存在"，当访问时触发页错误
  - 在页错误处理程序中分配物理页并加载内容
  - 内存使用效率高，但实现复杂



## 八、总结

###  实现成果

1. **成功实现 COW 机制**：
   - fork 时共享页，不清除写权限
   - 写操作时触发页错误，执行 COW 处理
   - 正确管理引用计数

2. **通过所有测试**：
   - 所有现有测试用例均通过
   - COW 机制不影响现有功能

3. **代码质量**：
   - 代码结构清晰
   - 注释完整
   - 错误处理完善


