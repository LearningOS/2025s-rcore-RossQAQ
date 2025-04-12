# Chapter 3. 实验报告

## 荣誉准则

1. 在完成本次实验的过程（含此前学习的过程）中，我曾分别与 **以下各位** 就（与本次实验相关的）以下方面做过交流，还在代码中对应的位置以注释形式记录了具体的交流对象及内容：

   > 无，独立完成

2. 此外，我也参考了 **以下资料** ，还在代码中对应的位置以注释形式记录了具体的参考来源及内容：

   > 代码中没有参考，ch3 练习比较简单。学习过程中主要参考：
   >
   > 1. [*The RISC-V Instruction Set Manual Volume I: Unprivileged ISA*](https://drive.google.com/file/d/1uviu1nH-tScFfgrovvFCrj7Omv8tFtkp/view?usp=drive_link) 
   > 2. [*The RISC-V Instruction Set Manual*](https://drive.google.com/file/d/17GeetSnT5wW3xNuAHI95-SI1gPGd5sJ_/view?usp=drive_link) [*Volume II: Privileged Architecture*](https://drive.google.com/file/d/17GeetSnT5wW3xNuAHI95-SI1gPGd5sJ_/view?usp=drive_link) 
   > 3. [*RISC-V ABIs Specification*](https://drive.google.com/file/d/1Ja_Tpp_5Me583CGVD-BIZMlgGBnlKU4R/view?usp=drive_link)
   > 4. [rustsbi - Rust](https://docs.rs/rustsbi/latest/rustsbi/)
   > 5. [*RISC-V Supervisor Binary Interface Specification*](https://drive.google.com/file/d/1U2kwjqxXgDONXk_-ZDTYzvsV-F_8ylEH/view?usp=drive_link)
   > 6. https://msyksphinz-self.github.io/riscv-isadoc/html

3. 我独立完成了本次实验除以上方面之外的所有工作，包括代码与文档。 我清楚地知道，从以上方面获得的信息在一定程度上降低了实验难度，可能会影响起评分。

4. 我从未使用过他人的代码，不管是原封不动地复制，还是经过了某些等价转换。 我未曾也不会向他人（含此后各届同学）复制或公开我的实验代码，我有义务妥善保管好它们。 我提交至本实验的评测系统的代码，均无意于破坏或妨碍任何计算机系统的正常运转。 我清楚地知道，以上情况均为本课程纪律所禁止，若违反，对应的实验成绩将按“-100”分计。

## 实验总结

本章要求实现 `sys_trace`，根据不同的 `trace_request` 执行不同操作。

考虑到 OS 上跑的是不同的任务，所以自然要在 `TaskManager` 内部实现这些功能，这样才能区分不同的任务。

对于

- `trace_request` 为 0 时，读取当前任务 `id` 地址处的一个字节。在 `TaskManager` 实现一个方法 `read_from_current_unchecked`，直接 `read_volatile` 传入的地址即可。对外暴露 wrapper `read_from_current_task_unchecked`。
- `trace_request` 为 1 时，向 `id` 地址写入 `data`。同样，在 `TaskManager` 实现方法 `write_into_current_unchecked`，直接 `write_volatile` 即可。对外暴露 wrapper，`write_into_current_task_unchecked`。
- `trace_request` 为 2 时，需要实现一个 syscall counter。自然会想到在 `TaskManagerInner` 内部维护一个结构来记录不同任务下不同 syscall 的调用次数。我这里维护了一个 `BTreeMap<usize, BTreeMap<usize, usize>>`，key 代表不同任务的 id，value 是 syscall id 与 syscall count 组成的映射（考虑到这个练习比较简单，就不做类型体操了）。还是，在 `TaskManager` 内实现 `get_syscall_count` 与 `syscall_count` 分别用来获取当前任务下某 syscall 的计数，以及统计 syscall 计数。同样，对外暴露 wrapper `get_syscall_count ` 与 `count_syscall`。

考虑到 1 和 2 两个直接读写内存的操作比较危险，所以函数命名带 `_unchecked`。

最后，在内核态的 syscall 函数中调用 `count_syscall`，在 `sys_trace` 中 match 三种情况分别调用 wrapper 即可。

## 简答作业

正确进入 U 态后，程序的特征还应有：使用 S 态特权指令，访问 S 态寄存器后会报错。 请同学们可以自行测试这些内容（运行 [三个 bad 测例 (ch2b_bad_*.rs)](https://github.com/LearningOS/rCore-Tutorial-Test-2025S/tree/master/src/bin) ）， 描述程序出错行为，同时注意注明你使用的 sbi 及其版本。

> 本人 sbi：[rustsbi] RustSBI version 0.3.0-alpha.2, adapting to RISC-V SBI v1.0.0
>
> 在 ch3 中，运行 ch2b_bad_address 会提示：
>
> PageFault in application, bad addr = 0x0, bad instruction = 0x804003a4, kernel killed it.
>
> 显然并不能读取 0x0 这个地址的内容，出错会被 kernel kill 掉。
>
> 剩下两个 bad 用例都会提示：
>
> IllegalInstruction in application, kernel killed it.
>
> 显然用户态并不能执行内核态的指令，没有权限，会被 kernel kill 掉。

深入理解 [trap.S](https://github.com/LearningOS/rCore-Tutorial-Code-2025S/blob/ch3/os/src/trap/trap.S) 中两个函数 `__alltraps` 和 `__restore` 的作用，并回答如下问题:

1. L40：刚进入 `__restore` 时，`sp` 代表了什么值。请指出 `__restore` 的两种使用情景。

   > `sp` 代表 Stack pointer。rCore 中 `__restore` 用于从内核态进入用户态，自然此时 `sp` 自然是内核栈的地址。
   >
   > 其两种使用情景：
   >
   > 1. 执行 trap handler 后，从内核态转为用户态
   > 2. 系统启动后，第一次进入用户态

2. L43-L48：这几行汇编代码特殊处理了哪些寄存器？这些寄存器的的值对于进入用户态有何意义？请分别解释。

   ```assembly
   ld t0, 32*8(sp)
   ld t1, 33*8(sp)
   ld t2, 2*8(sp)
   csrw sstatus, t0
   csrw sepc, t1
   csrw sscratch, t2
   ```

   > 此时，`sp` 指向内核栈中的 `TrapContext` 起始地址，它的结构为
   >
   > ```rust
   > pub struct TrapContext {
   >     pub x: [usize; 32],
   >     pub sstatus: Sstatus,
   >     pub sepc: usize,
   > }
   > ```
   >
   > 那么自然，会把 `TrapContext::sstatus` 读入寄存器 t0，把 `TrapContext::sepc` 读入寄存器 t1，根据 RISC-V ABI 1.1 的描述，x2 寄存器就是 `sp`，这里不要混，当前处于内核态，`sp` 的值是内核栈地址，而 `2*8(sp)` 是用户栈地址，它会被存入寄存器 t2。
   >
   > 之后使用 `csrw` 将他们读入对应的 CSR。
   >
   > **以下内容的依据是 RISC-V Privileged 4.1 的描述，**
   >
   > 在 `SRET` 执行时，如果 sstatus SPP 位为 0，就进入用户态。
   >
   > `sepc` 指向的是触发 Trap 时那条用户态指令的地址。
   >
   >  而 `sscratch` 存的是用户态栈的地址。

3. L50-L56：为何跳过了 `x2` 和 `x4`？

   ```assembly
   ld x1, 1*8(sp)
   ld x3, 3*8(sp)
   .set n, 5
   .rept 27
      LOAD_GP %n
      .set n, n+1
   .endr
   ```

   > 根据第二题和 RISC-V ABI 1.1 章节，x2 是用户态 `sp`，已经在 48 行保存过了；而 x4 时 Thread pointer，不需要处理。

4. L60：该指令之后，`sp` 和 `sscratch` 中的值分别有什么意义？

   ```assembly
   csrrw sp, sscratch, sp
   ```

   > 首先，`csrrw` 代表原子交换寄存器的值，这里交换了 `sp` 和 `sscratch` 中的值。
   >
   > 继续第二题的描述，`sp` 中的是内核栈，`sscratch` 是用户栈，二者交换后，`sp` 是用户栈，`sscratch` 是内核栈。

5. `__restore`：中发生状态切换在哪一条指令？为何该指令执行之后会进入用户态？

   > 根据 RISC-V Privileged 3.3.2 章节描述，执行 `SRET` 后会从内核态进入用户态。

6. L13：该指令之后，`sp` 和 `sscratch` 中的值分别有什么意义？

   ```assembly
   csrrw sp, sscratch, sp
   ```

   > 与第四题同理，只不过这里是 `__alltraps`，调用这个函数时我们处在用户态，也就是说，`sp` 指向用户栈，`sscratch` 指向内核栈，交换二者后，`sp` 指向内核栈，`sscratch` 指向用户栈，为从用户态进入内核态做准备。

7. 从 U 态进入 S 态是哪一条指令发生的？

   > 执行 `call trap_handler` 之前已经完成了进入 S 态的所有准备工作，调用 `trap_handler` 进入 S 态。

