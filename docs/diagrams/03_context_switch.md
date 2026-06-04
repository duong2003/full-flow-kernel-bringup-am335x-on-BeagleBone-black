# 3. Context Switch

> **Mục đích:** Cho thấy chi tiết những gì diễn ra bên trong `context_switch(prev, next)`
> — hàm assembly quan trọng nhất để chuyển đổi giữa các process.

## 3.1. Tổng quan

```mermaid
flowchart TB
    subgraph SAVE["PHA 1: Lưu prev"]
        S1["push {r4-r11, lr}<br/>→ lưu 9 word lên prev→kernel stack"]
        S2["str sp, [prev + ctx.sp_svc]<br/>→ ghi lại đỉnh stack hiện tại"]
        S3["cps #0x1F (SYS mode)<br/>→ truy cập banked USR registers"]
        S4["str SP_usr, [prev + ctx.sp_usr]<br/>str LR_usr, [prev + ctx.lr_usr]"]
        S5["cps #0x13 (về SVC mode)"]
    end

    subgraph SWAP["PHA 2: Swap address space"]
        W1["đọc next→pgd_pa"]
        W2["TTBR0 ← next→pgd_pa | 0x4A<br/>(page table walk attributes)"]
        W3["TLBIALL — flush TLB toàn bộ"]
        W4["ICIALLU — invalidate I-cache<br/>(cùng VA 0x40000000 → PA khác)"]
        W5["DSB + ISB — barrier"]
    end

    subgraph LOAD["PHA 3: Load next"]
        L1["cps #0x1F (SYS mode)<br/>→ load banked USR registers của next"]
        L2["ldr SP_usr, [next + ctx.sp_usr]<br/>ldr LR_usr, [next + ctx.lr_usr]"]
        L3["cps #0x13 (về SVC mode)"]
        L4["ldr sp, [next + ctx.sp_svc]<br/>→ swap sang kernel stack của next"]
        L5["pop {r4-r11, lr}<br/>→ 9 word từ next→kernel stack"]
        L6["bx lr<br/>→ resume tại nơi next đã bị dừng<br/>(hoặc ret_from_first_entry nếu first-time)"]
    end

    SAVE --> SWAP --> LOAD

    style SAVE fill:#fff3cd,stroke:#ffc107
    style SWAP fill:#cce5ff,stroke:#004085
    style LOAD fill:#d4edda,stroke:#28a745
```

## 3.2. First-time entry vs Preempted resume

```mermaid
flowchart TB
    subgraph FIRST["First-time entry: context_switch(NULL, proc0)"]
        F1["prev = NULL → skip SAVE"]
        F2["SWAP: TTBR0 ← proc0→pgd_pa<br/>TLBIALL + ICIALLU"]
        F3["LOAD: SP_svc ← proc0→ctx.sp_svc<br/>(trỏ vào initial frame 25 word)"]
        F4["pop {r4-r11, lr}<br/>lr ← ret_from_first_entry<br/>(được ghi sẵn trong initial frame)"]
        F5["bx lr → ret_from_first_entry"]
        F6["ret_from_first_entry:<br/>ldmfd sp!, {r0-r12, lr}<br/>rfefd sp!<br/>→ pop PC=0x40000000, CPSR=0x10<br/>→ USR mode!"]
    end

    subgraph PREEMPTED["Preempted resume: context_switch(p0, p1)"]
        P1["SAVE: push {r4-r11, lr} lên p0→kernel stack<br/>lưu SP_usr, LR_usr"]
        P2["SWAP: TTBR0 ← p1→pgd_pa<br/>flush TLB + I-cache"]
        P3["LOAD: SP_svc ← p1→ctx.sp_svc<br/>(trỏ vào frame do lần SAVE trước để lại)"]
        P4["pop {r4-r11, lr}<br/>lr ← địa chỉ trong handle_irq/schedule<br/>(nơi p1 bị preempt lần trước)"]
        P5["bx lr → quay về schedule()<br/>→ schedule return → handle_irq return<br/>→ ldmfd + rfefd → USR mode"]
    end

    FIRST ~~~ PREEMPTED
```

**Điểm khéo:** Cùng 1 code path `context_switch` dùng cho cả first-time entry và
preempted resume. Sự khác biệt nằm ở nội dung kernel stack frame chứ không ở code.
First-time entry có `lr = ret_from_first_entry` (pre-built), còn preempted resume
có `lr` trỏ vào giữa `schedule()` (do lần SAVE trước tự push vào).

## 3.3. Kernel stack frame layout

```mermaid
flowchart LR
    subgraph INIT["Initial frame (first-time)"]
        direction TB
        I1["r4..r11 = 0  (8 words)"]
        I2["lr = ret_from_first_entry"]
        I3["─── kernel-resume frame ───"]
        I4["r0..r12 = 0  (13 words)"]
        I5["svc_lr = 0"]
        I6["pc = 0x40000000  (user entry)"]
        I7["cpsr = 0x10  (USR mode)"]
        I8["─── IRQ-exit frame ───"]
    end

    subgraph SAVED["Saved frame (preempted)"]
        direction TB
        S1["r4..r11 (giá trị thật)"]
        S2["lr (địa chỉ resume thật)"]
        S3["─── kernel-resume frame ───"]
        S4["r0..r12 (giá trị thật)"]
        S5["svc_lr (giá trị thật)"]
        S6["pc (địa chỉ user bị ngắt)"]
        S7["cpsr (CPSR lúc bị ngắt)"]
        S8["─── IRQ-exit frame ───"]
    end
```

- **Kernel-resume frame (9 word):** `context_switch` epilogue pop `{r4-r11, lr}` từ đây.
- **IRQ-exit frame (16 word):** `ret_from_first_entry` (first-time) hoặc
  `ldmfd + rfefd` (preempted) drain frame này để trở về USR mode.
- Tổng: 25 word = 100 byte trên đỉnh kernel stack 8 KB.
