# 5. System Call

> **Mục đích:** Cho thấy luồng từ user code gọi `svc #0` đến kernel dispatch,
> xử lý, và return về user mode.

## 5.1. Tổng quan luồng syscall

```mermaid
sequenceDiagram
    participant APP as User Code (USR)
    participant LIBC as libc wrapper
    participant HW as CPU (SVC mode)
    participant VEC as vectors.S
    participant ENTRY as exception_entry.S
    participant HANDLER as handle_svc
    participant DISPATCH as syscall_dispatch
    participant IMPL as sys_* handler

    APP->>LIBC: write(1, "hello", 5)
    LIBC->>LIBC: mov r7, #SYS_WRITE (0)<br/>mov r0, #1 (fd)<br/>mov r1, buf<br/>mov r2, #5 (len)
    LIBC->>HW: svc #0

    HW->>HW: CPSR.M ← SVC (0x13)<br/>CPSR.I ← 1 (mask IRQ)<br/>LR_svc ← PC+4<br/>SP ← SP_svc (kernel stack)

    HW->>VEC: VBAR + 0x08 = SVC vector
    VEC->>ENTRY: ldr pc, _vec_svc<br/>→ exception_entry_svc

    ENTRY->>ENTRY: stmfd sp!, {r0-r12, lr}<br/>mrs r0, spsr<br/>stmfd sp!, {r0}<br/>mov r0, sp (ctx pointer)

    ENTRY->>HANDLER: bl handle_svc (r0 = ctx)
    HANDLER->>DISPATCH: syscall_dispatch(ctx)

    DISPATCH->>DISPATCH: num = ctx→r[7] (SYS_WRITE=0)
    DISPATCH->>DISPATCH: validate num < NR_SYSCALLS
    DISPATCH->>DISPATCH: fn = syscall_table[num]

    DISPATCH->>IMPL: ret = fn(ctx→r[0], ctx→r[1], ctx→r[2], ctx→r[3])

    IMPL->>IMPL: valid_user_ptr(buf, len)? ✓
    IMPL->>IMPL: for i in 0..len: uart_putc(buf[i])
    IMPL-->>DISPATCH: return 5 (bytes written)

    DISPATCH->>DISPATCH: ctx→r[0] = ret (5)

    HANDLER->>HANDLER: schedule()<br/>(xử lý nếu sys_yield/sys_exit<br/>đã set need_reschedule)

    ENTRY->>ENTRY: ldmfd sp!, {r0} → SPSR<br/>ldmfd sp!, {r0-r12, pc}^<br/>→ restore regs + return USR<br/>→ r0 = 5 (return value)

    APP->>APP: tiếp tục chạy với r0 = 5
```

## 5.2. Dispatch table

```mermaid
flowchart TB
    SVC["handle_svc(ctx)"]
    DISPATCH["syscall_dispatch(ctx)"]
    CHECK{"num = ctx→r[7]<br/>num < NR_SYSCALLS (8)?<br/>table[num] != NULL?"}

    subgraph TABLE["syscall_table[8]"]
        direction LR
        S0["[0] sys_write"]
        S1["[1] sys_getpid"]
        S2["[2] sys_yield"]
        S3["[3] sys_exit"]
        S4["[4] sys_read"]
        S5["[5] sys_ps"]
        S6["[6] sys_kill"]
        S7["[7] sys_ticks"]
    end

    subgraph VALIDATE["Pointer validation (nếu có)"]
        VAL["valid_user_ptr(buf, len)<br/>◉ addr >= USER_VIRT_BASE<br/>◉ addr+len <= USER_VIRT_BASE+1MB<br/>◉ không overflow"]
    end

    RETURN["ctx→r[0] = return value<br/>(dương = success<br/>âm = error code)"]
    ERROR["ctx→r[0] = E_BADCALL (-1)"]
    FAULT["return E_FAULT (-2)"]

    SVC --> DISPATCH
    DISPATCH --> CHECK
    CHECK -->|"yes"| TABLE
    CHECK -->|"no"| ERROR

    TABLE -->|"write / read / ps"| VALIDATE

    VALIDATE -->|"valid"| RETURN
    VALIDATE -->|"invalid"| FAULT

    TABLE -->|"getpid / yield / exit / kill / ticks"| RETURN
```

## 5.3. ABI — thanh ghi

| Register | Hướng | Vai trò |
|----------|-------|---------|
| `r7` | Input | Syscall number (0..7) |
| `r0` | Input | Argument 0 |
| `r1` | Input | Argument 1 |
| `r2` | Input | Argument 2 |
| `r3` | Input | Argument 3 |
| `r0` | Output | Return value |

## 5.4. Danh sách syscall

| # | Name | Args | Returns | Blocking? |
|---|------|------|---------|-----------|
| 0 | `write` | fd, buf, len | bytes written | Không |
| 1 | `getpid` | — | pid (0..2) | Không |
| 2 | `yield` | — | 0 | Không |
| 3 | `exit` | — | không return | Không |
| 4 | `read` | fd, buf, len | bytes read | **Có** — block nếu ring rỗng |
| 5 | `ps` | buf, size | bytes written | Không |
| 6 | `kill` | pid | 0 / E_BADCALL | Không |
| 7 | `ticks` | — | tick_count | Không |

`E_BADCALL = -1`, `E_FAULT = -2`. Các giá trị âm được user libc map sang errno.
