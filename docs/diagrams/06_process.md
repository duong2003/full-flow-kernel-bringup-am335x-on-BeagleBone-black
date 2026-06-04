# 6. Process Management

> **Mục đích:** Cho thấy cấu trúc PCB và cách 3 process được tổ chức trong bộ nhớ.

## 6.1. Process Control Block

```mermaid
flowchart TB
    subgraph PCB["process_t — 1 process"]
        direction TB
        subgraph CTX["proc_context_t ctx (offset 0)"]
            direction LR
            CTX_BODY["r0..r12  (13 × 4 = 52 bytes)<br/>sp_svc   (kernel stack pointer)<br/>lr_svc<br/>spsr     (CPSR để restore)<br/>sp_usr   (banked USR SP)<br/>lr_usr   (banked USR LR)"]
        end

        PID["pid: 0 / 1 / 2"]
        STATE["state: READY / RUNNING / BLOCKED / DEAD"]
        NAME["name: 'counter' / 'runaway' / 'shell'"]
        PGD["pgd: VA pointer → L1 table 16KB"]
        PGD_PA["pgd_pa: PA của L1 table → ghi vào TTBR0"]
        KSTACK["kstack_base: kernel stack 8KB"]
        KSTACK_SIZE["kstack_size: 8192"]
        USER_ENTRY["user_entry: 0x40000000"]
        USER_STACK["user_stack_top: 0x40100000"]
        USER_PA["user_phys_base: PA slot 1MB riêng"]
    end

    style CTX fill:#fff3cd,stroke:#ffc107
```

## 6.2. 3 process trong hệ thống

```mermaid
flowchart TB
    subgraph P0["Process 0: counter (pid=0)"]
        direction LR
        P0_PCB["PCB"]
        P0_PGD["L1 table<br/>16KB"]
        P0_KS["Kernel stack<br/>8KB"]
        P0_US["User PA slot<br/>RAM_BASE+0x200000<br/>1MB"]
        P0_PCB --> P0_PGD
        P0_PCB --> P0_KS
        P0_PCB --> P0_US
    end

    subgraph P1["Process 1: runaway (pid=1)"]
        direction LR
        P1_PCB["PCB"]
        P1_PGD["L1 table<br/>16KB"]
        P1_KS["Kernel stack<br/>8KB"]
        P1_US["User PA slot<br/>RAM_BASE+0x300000<br/>1MB"]
        P1_PCB --> P1_PGD
        P1_PCB --> P1_KS
        P1_PCB --> P1_US
    end

    subgraph P2["Process 2: shell (pid=2)"]
        direction LR
        P2_PCB["PCB"]
        P2_PGD["L1 table<br/>16KB"]
        P2_KS["Kernel stack<br/>8KB"]
        P2_US["User PA slot<br/>RAM_BASE+0x400000<br/>1MB"]
        P2_PCB --> P2_PGD
        P2_PCB --> P2_KS
        P2_PCB --> P2_US
    end

    subgraph SHARED["Tài nguyên dùng chung"]
        direction LR
        BOOT_PGD["boot_pgd<br/>16KB"]
        EXC_STACKS["Exception stacks<br/>IRQ · ABT · UND · FIQ<br/>mỗi cái 0.5-1KB"]
        KERNEL_IMG["Kernel .text + .data + .bss<br/>~80KB"]
    end
```

## 6.3. Bố trí Physical Memory

```mermaid
flowchart TB
    subgraph PHYS["Physical RAM (QEMU: 128MB @ 0x70000000)"]
        direction TB
        KERNEL["0x70000000: Kernel image<br/>.text + .data + .bss<br/>(chứa boot_pgd + proc_pgd[3])<br/>~1.4 MB"]
        P0_MEM["0x70200000: Process 0 user<br/>counter binary + .bss + stack<br/>1 MB"]
        P1_MEM["0x70300000: Process 1 user<br/>runaway binary + .bss + stack<br/>1 MB"]
        P2_MEM["0x70400000: Process 2 user<br/>shell binary + .bss + stack<br/>1 MB"]
        FREE["0x70500000+: Còn trống"]
    end

    style KERNEL fill:#f8d7da,stroke:#721c24
    style P0_MEM fill:#d4edda,stroke:#155724
    style P1_MEM fill:#d4edda,stroke:#155724
    style P2_MEM fill:#d4edda,stroke:#155724
    style FREE fill:#e2e3e5,stroke:#383d41
```

## 6.4. Virtual Memory view (mỗi process)

```mermaid
flowchart LR
    subgraph VA["Virtual Address (góc nhìn process)"]
        direction TB
        NULL_G["0x00000000: FAULT (NULL guard)"]
        USER_V["0x40000000: User code + data + stack<br/>1 MB — mỗi process map sang PA khác"]
        UNUSED["0x40100000 → 0xBFFFFFFF: Unmapped"]
        KERNEL_V["0xC0000000+: Kernel space<br/>map giống hệt cho mọi process<br/>(mirror từ boot_pgd)"]
        PERIPH_V["Peripherals: identity mapped<br/>QEMU: 0x10000000 / 0x1E000000<br/>BBB:  0x44E00000 / 0x48000000"]
    end

    style NULL_G fill:#f8d7da,stroke:#721c24
    style USER_V fill:#d4edda,stroke:#155724
    style KERNEL_V fill:#cce5ff,stroke:#004085
```

**Isolation thật sự:** Process A (counter) và process B (shell) cùng thấy user code
ở VA `0x40000000`, nhưng PA khác nhau → A ghi vào `0x40000000` không ảnh hưởng B.
MMU đảm bảo điều này qua per-process page table.
