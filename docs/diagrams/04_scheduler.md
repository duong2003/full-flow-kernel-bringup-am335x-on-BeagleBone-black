# 4. Scheduler

> **Mục đích:** Cho thấy state machine của process và thuật toán round-robin
> preemptive scheduler.

## 4.1. Process state machine

```mermaid
stateDiagram-v2
    READY: READY<br/>sẵn sàng chạy
    RUNNING: RUNNING<br/>đang chiếm CPU
    BLOCKED: BLOCKED<br/>chờ UART input
    DEAD: DEAD<br/>đã exit / kill / fault

    [*] --> READY: process_init_all()

    READY --> RUNNING: schedule() chọn
    RUNNING --> READY: timer IRQ preempt<br/>hoặc sys_yield()
    RUNNING --> BLOCKED: sys_read()<br/>chưa có data
    BLOCKED --> READY: UART RX IRQ<br/>scheduler_wake_reader()
    RUNNING --> DEAD: sys_exit() / sys_kill()<br/>hoặc user fault
    READY --> DEAD: sys_kill() bởi pid khác
    BLOCKED --> DEAD: sys_kill() bởi pid khác
    DEAD --> [*]: scheduler skip vĩnh viễn
```

| State | Ý nghĩa | Còn trong run queue? |
|-------|---------|---------------------|
| `READY` | Sẵn sàng chạy, chờ được schedule pick | Có |
| `RUNNING` | Đang chiếm CPU (chỉ 1 process) | Có |
| `BLOCKED` | Chờ UART input — bị loại khỏi run queue | Không |
| `DEAD` | Đã exit/kill/fault — scheduler skip | Không |

## 4.2. Thuật toán round-robin

```mermaid
flowchart TB
    START["schedule() được gọi<br/>từ handle_irq hoặc handle_svc"]

    CHECK_NEED{"need_reschedule?"}
    NOOP["return ngay<br/>(không cần switch)"]

    HINT{"wake_hint != NULL<br/>và hint != current?"}
    HINT_SWITCH["perform_switch(current, hint)<br/>→ ưu tiên process vừa được wake<br/>(tránh reader bị CPU hog bỏ đói)"]

    WALK["walk ring từ (current→pid + 1)<br/>tìm process READY hoặc RUNNING"]

    FOUND{"tìm thấy<br/>process khác<br/>runnable?"}
    SWITCH["perform_switch(prev, cand)<br/>1. prev→state = READY (nếu đang RUNNING)<br/>2. cand→state = RUNNING<br/>3. context_switch(prev, cand)"]
    KEEP["giữ current<br/>(không còn process nào khác<br/>để chạy)"]

    START --> CHECK_NEED
    CHECK_NEED -->|"no"| NOOP
    CHECK_NEED -->|"yes"| HINT

    HINT -->|"yes"| HINT_SWITCH
    HINT -->|"no"| WALK

    WALK --> FOUND
    FOUND -->|"yes"| SWITCH
    FOUND -->|"no"| KEEP

    style HINT_SWITCH fill:#cce5ff,stroke:#004085
    style SWITCH fill:#d4edda,stroke:#28a745
```

## 4.3. BLOCKED → READY wake-up path

```mermaid
sequenceDiagram
    participant USER as Shell (USR mode)
    participant KERNEL as Kernel (SVC mode)
    participant UART as UART RX IRQ

    USER->>KERNEL: sys_read(buf, len)
    KERNEL->>KERNEL: uart_rx_pop() = -1<br/>(ring buffer rỗng)
    KERNEL->>KERNEL: scheduler_block_on_input()<br/>current→state = BLOCKED<br/>blocked_reader = current<br/>need_reschedule = 1<br/>schedule() → context_switch

    Note over USER,KERNEL: Shell bị block — counter/runaway tiếp tục chạy

    UART->>KERNEL: Người dùng gõ phím → RX IRQ
    KERNEL->>KERNEL: uart_rx_irq()<br/>→ chip→ops→rx_irq<br/>→ uart_rx_push(c) vào ring buffer
    KERNEL->>KERNEL: scheduler_wake_reader()<br/>blocked_reader→state = READY<br/>wake_hint = blocked_reader<br/>need_reschedule = 1
    KERNEL->>KERNEL: schedule()<br/>→ ưu tiên wake_hint<br/>→ context_switch về shell

    USER->>KERNEL: Shell tiếp tục sys_read loop<br/>uart_rx_pop() = 'h' → copy vào buf<br/>return về user
```

**Điểm khéo:** `wake_hint` cho phép process vừa được wake (shell) nhận CPU ngay
trong lần `schedule()` tiếp theo, thay vì phải chờ hết vòng round-robin. Điều này
giữ cho interactive shell responsive ngay cả khi counter + runaway đều READY.
