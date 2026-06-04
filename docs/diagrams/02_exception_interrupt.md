# 2. Xử lý Exception & Interrupt

> **Mục đích:** Cho thấy luồng từ khi hardware báo hiệu exception/interrupt đến
> khi handler xử lý xong và return về user mode.

## 2.1. Tổng quan các loại exception

```mermaid
flowchart LR
    subgraph TRIGGER["Tác nhân"]
        HW_IRQ["Hardware Interrupt<br/>Timer 10ms · UART RX"]
        SW_SVC["System Call<br/>svc #0 từ user code"]
        HW_FAULT["Hardware Fault<br/>NULL deref · prefetch abort<br/>undefined instruction"]
    end

    subgraph CPU_ACT["CPU Response"]
        IRQ_MODE["IRQ mode<br/>CPSR.M=0x12<br/>SP←IRQ stack<br/>LR←PC+4"]
        SVC_MODE["SVC mode<br/>CPSR.M=0x13<br/>SP←SVC stack<br/>LR←PC+4"]
        ABT_MODE["ABT mode<br/>CPSR.M=0x17<br/>SP←ABT stack<br/>LR←PC+8/4"]
    end

    subgraph HANDLER["Kernel Handler"]
        IRQ_H["handle_irq<br/>irq_dispatch<br/>schedule"]
        SVC_H["handle_svc<br/>syscall_dispatch<br/>schedule"]
        FAULT_H["handle_data_abort<br/>handle_prefetch_abort<br/>handle_undefined"]
    end

    subgraph OUTCOME["Kết quả"]
        RESUME["return về user<br/>ldmfd + rfefd"]
        KILL["kill process<br/>state→DEAD<br/>schedule"]
        PANIC["kernel panic<br/>halt forever"]
    end

    HW_IRQ --> IRQ_MODE --> IRQ_H
    SW_SVC --> SVC_MODE --> SVC_H
    HW_FAULT --> ABT_MODE --> FAULT_H

    IRQ_H --> RESUME
    SVC_H --> RESUME
    FAULT_H -->|"fault từ USR"| KILL
    FAULT_H -->|"fault từ kernel"| PANIC
```

## 2.2. IRQ path — chi tiết

Đường đi của 1 timer IRQ từ lúc GIC báo hiệu đến khi schedule và return.

```mermaid
flowchart TB
    subgraph STEP1["1. Hardware → CPU"]
        H1["Timer (SP804/DMTIMER2) fire sau 10ms"]
        H2["GIC assert IRQ line → CPU"]
        H3["CPU: nếu CPSR.I=0, chấp nhận IRQ<br/>CPSR.I←1 (mask lại)<br/>chuyển sang IRQ mode<br/>SP←IRQ stack · LR←PC+4"]
    end

    subgraph STEP2["2. Vector table"]
        V1["VBAR → offset 0x18 = IRQ vector"]
        V2["ldr pc, _vec_irq<br/>→ exception_entry_irq"]
    end

    subgraph STEP3["3. Entry stub (trampoline)"]
        E1["sub lr, lr, #4<br/>LR ← địa chỉ instruction bị ngắt"]
        E2["srsdb sp!, #0x13<br/>push {LR_irq, SPSR_irq} lên SVC stack"]
        E3["cps #0x13<br/>chuyển sang SVC mode"]
        E4["stmfd sp!, {r0-r12, lr}<br/>push 14 GPRs lên SVC stack"]
        E5["bl handle_irq"]
    end

    subgraph STEP4["4. C handler"]
        C1["handle_irq()"]
        C2["irq_dispatch()<br/>→ active_intc→ops→get_active<br/>→ irq_table[n]()  (timer_irq)"]
        C3["timer_irq()<br/>→ chip→ops→irq (ack hardware)<br/>→ timer_tick()"]
        C4["timer_tick()<br/>→ tick_count++<br/>→ user_handler = scheduler_tick"]
        C5["scheduler_tick()<br/>→ need_reschedule = 1"]
        C6["schedule()<br/>nếu need_reschedule:<br/>→ walk ring tìm next READY<br/>→ context_switch(prev, next)"]
    end

    subgraph STEP5["5. Return về user"]
        R1["ldmfd sp!, {r0-r12, lr}<br/>restore 14 GPRs từ SVC stack"]
        R2["rfefd sp!<br/>pop {PC, CPSR} từ SVC stack<br/>→ return về user mode<br/>→ CPSR.I ← 0 (enable IRQ)"]
    end

    STEP1 --> STEP2 --> STEP3 --> STEP4 --> STEP5

    style STEP3 fill:#fff3cd,stroke:#ffc107
    style STEP5 fill:#d4edda,stroke:#28a745
```

**Ghi chú thiết kế quan trọng:**

- **Trampoline pattern:** IRQ stack chỉ giữ vài register trong 3-4 instruction.
  `srsdb` push thẳng `{LR_irq, SPSR_irq}` sang SVC stack của process hiện tại
  — tránh dùng IRQ stack cho logic phức tạp.
- **Không nested IRQ:** CPSR.I bị set ngay khi vào IRQ mode. Handler chạy với
  IRQ tắt → không lo IRQ stack bị overwrite.
- **Schedule tail:** `handle_irq` gọi `schedule()` sau `irq_dispatch()` —
  nếu timer tick vừa set `need_reschedule`, process được swap trước khi `rfefd`
  return. Process mới nhận `rfefd` trỏ sang stack của nó.

## 2.3. Fault isolation

```mermaid
flowchart TB
    FAULT["Data Abort / Prefetch Abort / Undefined Instruction"]
    CHECK{"ctx→spsr & 0x1F == 0x10?<br/>(fault từ USR mode?)"}

    USER_PATH["user_fault_kill:"] --> KILL_PROC
    KERNEL_PATH["kernel panic:"]

    subgraph USER_KILL["Process bị kill — kernel sống"]
        direction TB
        U1["uart_printf tên + loại fault"]
        U2["current→state = TASK_DEAD"]
        U3["scheduler_request_resched()"]
        U4["schedule() → context_switch sang process khác"]
        U5["Process còn lại (counter, runaway) vẫn chạy bình thường"]
    end

    subgraph KERNEL_PANIC["Kernel bug — halt thật"]
        direction TB
        K1["uart_printf DFSR/DFAR + dump_context"]
        K2["halt_forever: wfi loop"]
    end

    FAULT --> CHECK
    CHECK -->|"YES"| USER_KILL
    CHECK -->|"NO"| KERNEL_PANIC
```

Đây là cơ chế cô lập lỗi cốt lõi: user process A crash → kernel + process B/C
tiếp tục chạy. Kernel giữ toàn quyền kiểm soát.
