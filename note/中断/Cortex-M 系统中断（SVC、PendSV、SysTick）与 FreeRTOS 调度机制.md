> 平台：ARM Cortex-M3/M4（如 STM32F1/F4）
> 
> **RTOS**：FreeRTOS
> 
> **目标**：理解 SVC、PendSV、SysTick 三大系统异常如何协同实现任务调度

---

## **一、Cortex-M 内核关键特性回顾**

ARM Cortex-M 是专为嵌入式微控制器设计的 32 位 RISC 内核，其核心特性包括：

- **两种运行模式**：
    - **Thread Mode**：运行用户任务（可配置为特权/非特权）
    - **Handler Mode**：执行异常/中断（始终特权）
- **两个堆栈指针**：
    - **MSP（Main Stack Pointer）**：用于 Handler 模式和主程序
    - **PSP（Process Stack Pointer）**：用于 Thread 模式下的用户任务
- **嵌套向量中断控制器（NVIC）**：支持低延迟、可嵌套中断
- **固定系统异常编号**：前 16 项为内核异常（含 Reset、NMI、HardFault、SVC、PendSV、SysTick 等）

这些特性为实时操作系统（RTOS）提供了硬件级支持。

---

## **二、三大系统中断详解**

### **1. SysTick（系统节拍定时器）**

- **异常号**：15
    
- **作用**：提供周期性“心跳”，驱动 RTOS 时间管理
    
- **在 FreeRTOS 中的角色**：
    
    - 默认每 1ms 触发一次（可配置）
    - 在 `SysTick_Handler` 中：
        - 更新系统节拍计数（`xTickCount`）
        - 检查是否有任务延时到期
        - 若需调度，则**挂起 PendSV 异常**
- **关键代码（简化）**：
    ``` c
    void SysTick_Handler(void) {
        if (xTaskIncrementTick() != pdFALSE) {
            portYIELD_FROM_ISR();// 实际是触发 PendSV
        }
    }
    ```

### **2. SVC（Supervisor Call）**

- **异常号**：11
- **触发方式**：执行 `svc #imm` 指令
- **特点**：立即执行，不可挂起
- **在 FreeRTOS 中的角色**：
    - **仅用于启动第一个任务**（在 `vTaskStartScheduler()` 中）
    - 通过 SVC 进入特权模式，完成：
        - 设置 PSP 指向第一个任务的栈顶
        - 配置 PendSV 为最低优先级
        - 切换到 Thread Mode + PSP
- **为什么只用一次？**启动后所有任务已在 Thread Mode 运行，后续调度无需再进入 SVC。

### **3. PendSV（Pendable Service Call）**

- **异常号**：14
- **触发方式**：写 `SCB->ICSR |= SCB_ICSR_PENDSVSET_Msk`
- **特点**：**可挂起**，通常设为**最低优先级**
- **在 FreeRTOS 中的核心角色**：
    - **唯一执行上下文切换的地方**
    - 因其“可延迟”特性，确保：
        - 不会打断高优先级中断（如 UART、ADC）
        - 上下文切换总是在“安全时刻”（无活跃中断）发生
- **切换流程**：
    1. 保存当前任务寄存器（R4–R11）到其堆栈
    2. 调用调度器选择下一个就绪任务
    3. 从新任务堆栈恢复寄存器
    4. 更新 PSP，返回新任务

> ✅ 设计哲学：
> 
> “**调度判断在 SysTick，实际切换在 PendSV**” —— 分离决策与执行，保证实时性。

---

## **三、FreeRTOS 启动与调度流程**

### **启动阶段（`main()` → 第一个任务）**

``` text
main()
  └─ vTaskStartScheduler()
        ├─ 创建 Idle 任务
        ├─ 配置 SysTick
        └─ 调用 xPortStartScheduler()
              └─ 执行 svc 0 指令
                    └─ SVC_Handler:
                          - 设置 PSP = task1 栈顶
                          - 设置 PendSV 优先级 = configKERNEL_INTERRUPT_PRIORITY（最低）
                          - bx lr → 切换到 task1（Thread Mode + PSP）
```

### **调度阶段（任务切换）**

``` text
SysTick 中断发生
  └─ SysTick_Handler()
        └─ xTaskIncrementTick() → 发现有更高优先级任务就绪
              └─ portYIELD_FROM_ISR()
                    └─ 触发 PendSV（设置 ICSR.PENDSVSET）

[所有高优先级中断处理完毕后]

PendSV_Handler()
  ├─ 保存当前任务上下文（R4-R11）
  ├─ 调用 vTaskSwitchContext() 选择新任务
  ├─ 恢复新任务上下文
  └─ bx lr → 跳转到新任务
```

---

## **四、关键寄存器与代码片段**

### **1. 触发 PendSV（FreeRTOS 实现）**

``` c

#define portNVIC_INT_CTRL_REG     (*(volatile uint32_t *)0xe000ed04)
#define portNVIC_PENDSVSET_BIT    (1UL << 28UL)

#define portYIELD() \\
    portNVIC_INT_CTRL_REG = portNVIC_PENDSVSET_BIT
```

### **2. PendSV_Handler 核心逻辑（简化）**

``` asm

PendSV_Handler:
    mrs r0, psp; 获取当前任务 PSP
    stmdb r0!, {r4-r11}; 保存 R4-R11
    bl vTaskSwitchContext; 选择下一个任务
    ldmia r0!, {r4-r11}; 恢复新任务 R4-R11
    msr psp, r0; 更新 PSP
    bx lr; 返回新任务
```

### **3. 优先级设置（确保 PendSV 最低）**

``` c
// 在 port.c 中
NVIC_SetPriority(PendSV_IRQn, configKERNEL_INTERRUPT_PRIORITY);
NVIC_SetPriority(SysTick_IRQn, configKERNEL_INTERRUPT_PRIORITY);
// configKERNEL_INTERRUPT_PRIORITY 通常为 0xFF（最低）
```

---

## **五、为什么这样设计？**

|**设计**|**原因**|
|---|---|
|**PendSV 优先级最低**|避免打断外设中断，保证硬实时响应|
|**上下文切换在 PendSV**|利用其“可挂起”特性，实现原子切换|
|**SVC 仅用于启动**|启动后无需再进入特权模式调用内核（任务已运行在合适模式）|
|**SysTick 只做调度判断**|快速退出，减少中断延迟|

> 💡 经典比喻：
> 
> - **SysTick** 是“裁判哨声”（提醒该换人了）
> - **PendSV** 是“换人动作”（实际执行上下场）
> - **SVC** 是“开幕式入场”（只在比赛开始时用一次）

---

## **六、学习收获与思考**

1. **硬件与软件深度协同**：Cortex-M 的异常模型为 RTOS 提供了天然支持。
2. **实时性保障**：通过 PendSV 的“延迟执行”机制，FreeRTOS 在保证功能的同时不牺牲中断响应速度。
3. **特权级安全**：SVC 提供了从用户态到内核态的安全入口，防止应用代码直接操作关键寄存器。
4. **可移植性**：FreeRTOS 将架构相关代码（如 PendSV_Handler）放在 `port.c`，便于移植到不同内核。

---

## **七、延伸思考**

- 如果 PendSV 优先级设得太高，会发生什么？
    
    → 可能频繁抢占外设中断，导致数据丢失或通信错误。
    
- 能否不用 PendSV 实现 RTOS？
    
    → 理论上可以，但无法保证在中断中安全切换，违背 RTOS 实时性原则。
    
- 在 ARMv8-M（如 M33）中，TrustZone 如何影响这些中断？
    
    → SVC/PendSV/SysTick 可配置为 Secure 或 Non-Secure，需额外考虑安全边界。
    

---

> 总结：
> 
> SVC、PendSV、SysTick 是 Cortex-M 上 RTOS 的“铁三角”。
> 
> 理解它们的协作机制，就掌握了 FreeRTOS 调度的核心灵魂。

---

_笔记整理于 2025 年，基于 FreeRTOS v10.5 + Cortex-M4 架构_