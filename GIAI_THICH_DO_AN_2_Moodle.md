# GIẢI THÍCH CHI TIẾT ĐỒ ÁN 2

1. **Task 1**: Speed up system calls (USYSCALL)
2. **Task 2**: Print page table (vmprint)
3. **Task 3**: Large file

---

# TASK 1: SPEED UP SYSTEM CALLS (USYSCALL)

## 🎯 MỤC TIÊU

Tối ưu hóa system call `getpid()` bằng cách **loại bỏ việc chuyển đổi giữa user mode và kernel mode** (context switch).

---

## 📊 SO SÁNH TRƯỚC VÀ SAU

### ❌ TRƯỚC (Cách hiện tại - CHẬM)

```
User Program                    Kernel
    |                              |
    | getpid()                     |
    |----------------------------->|
    |   (trap vào kernel)          |
    |                              | sys_getpid()
    |                              | return myproc()->pid
    |<-----------------------------|
    |   (return về user)           |
    v                              v
```

**Vấn đề:** Mỗi lần gọi `getpid()` phải:
1. Chuyển từ user mode → kernel mode (trap)
2. Thực thi code trong kernel
3. Chuyển từ kernel mode → user mode (return)

→ **Tốn kém về performance!**

---

### ✅ SAU (Cách tối ưu - NHANH)

```
User Program                    Shared Memory (USYSCALL)
    |                                   |
    | ugetpid()                         |
    | đọc trực tiếp từ USYSCALL ------> | struct usyscall {
    | return u->pid                     |   int pid;
    |                                   | }
    v                                   v
```

**Lợi ích:** 
- Không cần trap vào kernel
- Chỉ đọc memory
- Kernel đã ghi sẵn PID vào vùng shared memory

---

## 🔧 CÁCH THỰC HIỆN

### Bước 1: Hiểu cách TRAPFRAME được xử lý (để làm tương tự)

Đọc hiểu `kernel/proc.c`, hàm `proc_pagetable()`

### Bước 2: Thêm USYSCALL mapping (tương tự TRAPFRAME)

**Cần làm 5 bước:**

#### A. Thêm field vào `struct proc` (kernel/proc.h)
Thêm pointer `struct usyscall *usyscall` vào struct proc.

#### B. Cấp phát memory trong `allocproc()` (kernel/proc.c)
Gọi `kalloc()` để cấp phát 1 page, khởi tạo `p->usyscall->pid = p->pid`.

#### C. Map vào page table trong `proc_pagetable()` (kernel/proc.c)
Gọi `mappages()` với địa chỉ `USYSCALL`, permission `PTE_R | PTE_U` (READ-ONLY).

#### D. Giải phóng memory trong `freeproc()` (kernel/proc.c)
Gọi `kfree((void*)p->usyscall)` và set `p->usyscall = 0`.

#### E. Unmap trong `proc_freepagetable()` (kernel/proc.c)
Gọi `uvmunmap(pagetable, USYSCALL, 1, 0)`.

---

## 🔍 CÁCH HOẠT ĐỘNG

### 1. Khi process được tạo (allocproc)

```
Kernel:
  1. Cấp phát 1 page physical memory cho usyscall
  2. Ghi PID vào usyscall->pid
  3. Map page này vào virtual address USYSCALL
  4. Set permission: READ-ONLY (PTE_R | PTE_U)
```

### 2. Khi user program gọi ugetpid()

```c
// user/ulib.c
int ugetpid(void)
{
  struct usyscall *u = (struct usyscall *)USYSCALL;
  return u->pid;  // ← Đọc trực tiếp từ memory!
}
```

**Không cần trap vào kernel!** Chỉ đọc memory thôi.

---

## 🎓 KIẾN THỨC CẦN NẮM

### 1. Page Table Entry (PTE) Flags

```c
#define PTE_V (1L << 0)  // Valid - entry có hiệu lực
#define PTE_R (1L << 1)  // Readable
#define PTE_W (1L << 2)  // Writable
#define PTE_X (1L << 3)  // Executable
#define PTE_U (1L << 4)  // User can access
```

**Ví dụ:**
- `PTE_R | PTE_W | PTE_U`: User có thể đọc và ghi
- `PTE_R | PTE_U`: User chỉ có thể đọc (READ-ONLY)
- `PTE_R | PTE_X`: Kernel có thể đọc và thực thi

### 2. Hàm quan trọng

```c
// Cấp phát 1 page (4096 bytes) physical memory
char *kalloc(void);

// Giải phóng 1 page
void kfree(void *pa);

// Map virtual address → physical address
int mappages(pagetable_t pagetable, uint64 va, uint64 size, 
             uint64 pa, int perm);

// Unmap virtual address
void uvmunmap(pagetable_t pagetable, uint64 va, 
              uint64 npages, int do_free);
```

---

# TASK 2: PRINT PAGE TABLE (vmprint)

## 🎯 MỤC TIÊU

Viết hàm `vmprint()` để in ra cấu trúc page table, giúp debug và hiểu rõ cách page table hoạt động.

## 📊 OUTPUT MẪU

Khi chạy `init` process, sẽ in ra:

```
page table 0x0000000087f6b000
 ..0: pte 0x0000000021fd9c01 pa 0x0000000087f67000
 .. ..0: pte 0x0000000021fd9801 pa 0x0000000087f66000
 .. .. ..0: pte 0x0000000021fda01b pa 0x0000000087f68000
 .. .. ..1: pte 0x0000000021fd9417 pa 0x0000000087f65000
 .. .. ..2: pte 0x0000000021fd9007 pa 0x0000000087f64000
 .. .. ..3: pte 0x0000000021fd8c17 pa 0x0000000087f63000
 ..255: pte 0x0000000021fda801 pa 0x0000000087f6a000
 .. ..511: pte 0x0000000021fda401 pa 0x0000000087f69000
 .. .. ..509: pte 0x0000000021fdcc13 pa 0x0000000087f73000
 .. .. ..510: pte 0x0000000021fdd007 pa 0x0000000087f74000
 .. .. ..511: pte 0x0000000020001c0b pa 0x0000000080007000
init: starting sh
```

## 🗺️ GIẢI THÍCH OUTPUT

### Hiểu cấu trúc Page Table 3 cấp

Page table giống như một **cây 3 tầng**:
- **Level 2 (Root)**: Tầng gốc - điểm bắt đầu
- **Level 1**: Tầng giữa - nhánh trung gian  
- **Level 0 (Leaf)**: Tầng lá - trỏ đến page thật chứa data

**Ghi nhớ:** Level càng cao (2) thì càng gần gốc, Level càng thấp (0) thì càng gần lá.

### Đọc output như thế nào?

Mỗi dòng có format:
```
..0: pte 0x0000000021fd9c01 pa 0x0000000087f67000
│ │     │                    │
│ │     │                    └──────────► Địa chỉ vật lý (Physical Address)
│ │     └───────────────────────────────► Giá trị PTE (chứa địa chỉ + quyền)
│ └─────────────────────────────────────► Số thứ tự entry (0-511)
└───────────────────────────────────────► Độ sâu (số dấu ".." = level)
```

**Quy tắc đọc:**
- `..` (1 cặp dấu chấm) = Level 2 (gốc)
- `.. ..` (2 cặp dấu chấm) = Level 1 (giữa)
- `.. .. ..` (3 cặp dấu chấm) = Level 0 (lá - page thật)

### Giải thích chi tiết một dòng

Lấy ví dụ dòng: `..0: pte 0x0000000021fd9c01 pa 0x0000000087f67000`

**Phân tích từng phần:**

1. **`..`** = Level 2 (tầng gốc)
2. **`0`** = Entry số 0 trong page table Level 2
3. **`pte 0x0000000021fd9c01`** = Giá trị PTE (Page Table Entry)
   - Chứa địa chỉ vật lý + các flags (quyền truy cập)
   - Giá trị này được lưu trong bộ nhớ
4. **`pa 0x0000000087f67000`** = Physical Address (địa chỉ vật lý)
   - Đây là địa chỉ thật trong RAM
   - Trỏ đến page table Level 1 (vì đây là Level 2, chưa phải leaf)

**Ý nghĩa:**
- Entry 0 ở Level 2 trỏ đến một page table Level 1 tại địa chỉ `0x87f67000`
- Đây KHÔNG phải page chứa data, mà là page table tiếp theo
- Để biết có phải leaf không, xem flags: nếu chỉ có `V` (Valid) thì là pointer, nếu có `R/W/X` thì là leaf

### Ví dụ cụ thể từ output:

```
 ..0: pte ... pa 0x...67000          ← Level 2, entry 0
 .. ..0: pte ... pa 0x...66000       ← Level 1, entry 0
 .. .. ..0: pte ... pa 0x...68000    ← Level 0, entry 0 = TEXT (code)
 .. .. ..1: pte ... pa 0x...65000    ← Level 0, entry 1 = DATA
 .. .. ..2: pte ... pa 0x...64000    ← Level 0, entry 2 = BSS
 .. .. ..3: pte ... pa 0x...63000    ← Level 0, entry 3 = HEAP
```

**Giải thích:**
- Entry 0 ở Level 2 → trỏ đến page table Level 1
- Entry 0 ở Level 1 → trỏ đến page table Level 0
- Entries 0,1,2,3 ở Level 0 → trỏ đến các page thật chứa code và data


### Phân biệt Leaf vs Non-Leaf

**Non-leaf (pointer đến page table con):**
- Chỉ có flag `V` (Valid)
- Không có `R`, `W`, `X`
- Ví dụ: `..0:` và `.. ..0:` trong output

**Leaf (page thật chứa data):**
- Có ít nhất 1 trong `R`, `W`, `X`
- Ví dụ: `.. .. ..0:` (có R/W/X)
### Permission Bits - Đọc quyền truy cập

Mỗi PTE có 10 bits flags (bits 0-9), quan trọng nhất là 5 bits đầu:

```
Bit:  4 3 2 1 0
      ─ ─ ─ ─ ─
      U X W R V
      │ │ │ │ └─► Valid (có hiệu lực)
      │ │ │ └───► Readable (đọc được)
      │ │ └─────► Writable (ghi được)
      │ └───────► Executable (chạy được)
      └─────────► User accessible (user truy cập được)
```

**Ví dụ đọc flags (từ đơn giản → phức tạp):**

```
0x01 = 0b00001 = V         → Chỉ valid (pointer đến page table Level tiếp theo)
0x17 = 0b10111 = U|W|R|V   → User có thể đọc/ghi (data, heap, stack)
0x1b = 0b11011 = U|X|W|R|V → User có thể đọc/ghi/chạy (code)
         ↑   ↑
         bit 4 bit 0 (đọc từ phải sang trái)
```

**Cách nhớ:**
- Có `X` (Executable) → Code (text segment)
- Không có `X` nhưng có `R|W|U` → Data (data, heap, stack)
- Chỉ có `V` → Pointer đến page table con (không phải page thật)

## 🔧 CÁCH THỰC HIỆN

### Bước 1: Thêm prototype vào `kernel/defs.h`

### Bước 2: Implement `vmprint()` trong `kernel/vm.c`


**Logic thực hiện:**

1. Duyệt qua 512 entries trong page table
2. Với mỗi entry valid (có bit V):
   - In indent (số cặp `..` = level)
   - In số thứ tự entry, giá trị PTE, và địa chỉ vật lý
   - Nếu không có bit R/W/X → đây là pointer đến page table con → gọi đệ quy xuống level tiếp theo
   - Nếu có bit R/W/X → đây là leaf page (page thật) → không đệ quy nữa

### Bước 3: Gọi trong `kernel/exec.c`

Thêm vào cuối hàm `exec()`, trước `return argc`: nếu `p->pid == 1` thì gọi `vmprint(p->pagetable)`.

## 💡 GỢI Ý

- Tham khảo hàm `freewalk()` trong `vm.c` - cũng duyệt page table đệ quy
- Dùng macro `PTE2PA(pte)` để lấy physical address từ PTE
- Dùng `%p` trong printf để in địa chỉ 64-bit hex


---

# TASK 3: LARGE FILE (Doubly-Indirect Block)

## 🎯 MỤC TIÊU

Tăng kích thước tối đa của file trong xv6 từ **268 blocks** lên **65803 blocks** bằng cách thêm hỗ trợ **doubly-indirect block**.

---

## 📊 TÌNH TRẠNG HIỆN TẠI (TRƯỚC KHI SỬA)

### Cấu trúc inode gốc

```
addrs[0]  → data block 0        ─┐
addrs[1]  → data block 1         │
addrs[2]  → data block 2         │ 12 direct blocks
...                              │
addrs[11] → data block 11       ─┘
addrs[12] → indirect block ──→ [0] → data block 12
                                [1] → data block 13
                                ...
                                [255] → data block 267
```

**Giới hạn:** 12 (direct) + 256 (singly-indirect) = **268 blocks** = 268 KB

### Các hằng số gốc (kernel/fs.h)

```c
#define NDIRECT 12                        // 12 direct blocks
#define NINDIRECT (BSIZE / sizeof(uint))  // 1024/4 = 256
#define MAXFILE (NDIRECT + NINDIRECT)     // 12 + 256 = 268
```

### Mảng addrs gốc

```c
// kernel/fs.h - struct dinode (on-disk)
uint addrs[NDIRECT+1];   // addrs[13] → 12 direct + 1 indirect

// kernel/file.h - struct inode (in-memory)
uint addrs[NDIRECT+1];   // addrs[13] → giống dinode
```

**Quan trọng:** Mảng `addrs[]` có đúng **13 phần tử** (index 0-12). Kích thước này KHÔNG ĐƯỢC THAY ĐỔI vì nó ảnh hưởng đến kích thước `struct dinode` trên disk.

---

## 📊 CẤU TRÚC SAU KHI SỬA

### Ý tưởng: Hy sinh 1 direct block → thêm 1 doubly-indirect block

```
addrs[0]  → data block 0        ─┐
addrs[1]  → data block 1         │
...                              │ 11 direct blocks (giảm từ 12 → 11)
addrs[10] → data block 10       ─┘
addrs[11] → singly-indirect block ──→ [0] → data block 11
                                       [1] → data block 12
                                       ...
                                       [255] → data block 266
addrs[12] → doubly-indirect block ──→ [0] → indirect block 0 ──→ [0] → data
                                       │                          [1] → data
                                       │                          ...
                                       │                          [255] → data
                                       [1] → indirect block 1 ──→ [0] → data
                                       │                          ...
                                       │                          [255] → data
                                       ...
                                       [255] → indirect block 255 ──→ ...
```

**Giới hạn mới:** 11 + 256 + 256×256 = 11 + 256 + 65536 = **65803 blocks**

---

## 🧮 TÍNH TOÁN CHI TIẾT

### Tại sao giảm NDIRECT từ 12 → 11?

Mảng `addrs[]` luôn có **NDIRECT + 1** phần tử. Nhưng bây giờ ta cần **2 slot** cho indirect:
- Slot cho singly-indirect: `addrs[NDIRECT]` = `addrs[11]`
- Slot cho doubly-indirect: `addrs[NDIRECT+1]` = `addrs[12]`

Vậy tổng phần tử = `NDIRECT + 2` = 11 + 2 = **13** → vẫn giữ nguyên kích thước mảng!

```
Trước: addrs[13] = 12 direct + 1 indirect         = NDIRECT(12) + 1
Sau:   addrs[13] = 11 direct + 1 indirect + 1 dbl = NDIRECT(11) + 2
```

### Tính MAXFILE mới

```c
NDIRECT = 11
NINDIRECT = 256  (= BSIZE / sizeof(uint) = 1024 / 4)
NDBLINDIRECT = 256 * 256 = 65536

MAXFILE = NDIRECT + NINDIRECT + NDBLINDIRECT
        = 11 + 256 + 65536
        = 65803
```

---

## 🔧 CÁCH THỰC HIỆN TỪNG BƯỚC

### Bước 1: Sửa hằng số trong `kernel/fs.h`

**Code gốc:**
```c
#define NDIRECT 12
#define NINDIRECT (BSIZE / sizeof(uint))
#define MAXFILE (NDIRECT + NINDIRECT)
```

**Code mới:**
```c
#define NDIRECT 11
#define NINDIRECT (BSIZE / sizeof(uint))
#define MAXFILE (NDIRECT + NINDIRECT + NINDIRECT * NINDIRECT)
```

**Giải thích:**
- `NDIRECT` giảm từ 12 → 11 (hy sinh 1 direct block)
- `NINDIRECT` giữ nguyên = 256
- `MAXFILE` = 11 + 256 + 256×256 = 65803

---

### Bước 2: Sửa mảng `addrs[]` trong `kernel/fs.h` và `kernel/file.h`

**Trong `kernel/fs.h` - struct dinode (on-disk inode):**

```c
// Code gốc:
uint addrs[NDIRECT+1];   // khi NDIRECT=12 → addrs[13]

// Code mới:
uint addrs[NDIRECT+2];   // khi NDIRECT=11 → addrs[13] → CÙNG KÍCH THƯỚC!
```

**Trong `kernel/file.h` - struct inode (in-memory inode):**

```c
// Code gốc:
uint addrs[NDIRECT+1];

// Code mới:
uint addrs[NDIRECT+2];
```

**Tại sao kích thước không đổi?**
```
Trước: NDIRECT+1 = 12+1 = 13 phần tử
Sau:   NDIRECT+2 = 11+2 = 13 phần tử  ← GIỐNG NHAU!
```

Đây là điểm mấu chốt: ta giảm NDIRECT đi 1 để "nhường chỗ" cho doubly-indirect slot, nên tổng kích thước mảng vẫn là 13. Kích thước `struct dinode` trên disk không thay đổi → không cần format lại disk layout.

---

### Bước 3: Sửa hàm `bmap()` trong `kernel/fs.c`

Hàm `bmap(ip, bn)` nhận logical block number `bn` và trả về disk block address. Ta cần thêm xử lý cho doubly-indirect.

**Thêm phần doubly-indirect trước `panic`:**

```c
static uint
bmap(struct inode *ip, uint bn)
{
  uint addr, *a;
  struct buf *bp;

  // === Phần 1: Direct blocks (bn = 0..10) ===
  if(bn < NDIRECT){
    if((addr = ip->addrs[bn]) == 0){
      addr = balloc(ip->dev);
      if(addr == 0)
        return 0;
      ip->addrs[bn] = addr;
    }
    return addr;
  }
  bn -= NDIRECT;

  // === Phần 2: Singly-indirect blocks (bn = 0..255) ===
  if(bn < NINDIRECT){
    // Load indirect block, allocating if necessary.
    if((addr = ip->addrs[NDIRECT]) == 0){
      addr = balloc(ip->dev);
      if(addr == 0)
        return 0;
      ip->addrs[NDIRECT] = addr;
    }
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if((addr = a[bn]) == 0){
      addr = balloc(ip->dev);
      if(addr){
        a[bn] = addr;
        log_write(bp);
      }
    }
    brelse(bp);
    return addr;
  }
  bn -= NINDIRECT;

  // === Phần 3: Doubly-indirect blocks (bn = 0..65535) ===
  if(bn < NINDIRECT * NINDIRECT){
    // --- Bước 3a: Load doubly-indirect block ---
    // addrs[NDIRECT+1] chứa địa chỉ của doubly-indirect block

    ...

    // --- Bước 3b: Tìm singly-indirect block bên trong ---
    // bn / NINDIRECT = index vào doubly-indirect block
    // Ví dụ: bn=300 → 300/256 = 1 → lấy indirect block thứ 1

    ...

    // --- Bước 3c: Load singly-indirect block và tìm data block ---
    // bn % NINDIRECT = index vào singly-indirect block
    // Ví dụ: bn=300 → 300%256 = 44 → lấy data block thứ 44
    
    ...

  }

  panic("bmap: out of range");
}
```

#### Giải thích luồng doubly-indirect bằng sơ đồ:

```
Cho bn = 300 (logical block thứ 300 trong vùng doubly-indirect)

Bước 3a: Đọc doubly-indirect block từ addrs[NDIRECT+1]
         ┌─────────────────────────────────┐
         │ doubly-indirect block            │
         │ [0] → indirect block 0           │
         │ [1] → indirect block 1  ◄── 300/256 = 1
         │ [2] → indirect block 2           │
         │ ...                              │
         │ [255] → indirect block 255       │
         └─────────────────────────────────┘

Bước 3b: Đọc indirect block 1
         ┌─────────────────────────────────┐
         │ indirect block 1                 │
         │ [0] → data block                 │
         │ ...                              │
         │ [44] → data block  ◄── 300%256 = 44
         │ ...                              │
         │ [255] → data block               │
         └─────────────────────────────────┘

Bước 3c: Trả về địa chỉ data block tại index 44
```

---

### Bước 4: Sửa hàm `itrunc()` trong `kernel/fs.c`

Hàm `itrunc()` giải phóng tất cả blocks của file. Ta cần thêm code giải phóng doubly-indirect blocks.

**Thêm phần giải phóng doubly-indirect (trước `ip->size = 0`):**

```c
void
itrunc(struct inode *ip)
{
  int i, j, k;
  struct buf *bp, *bp2;
  uint *a, *a2;

  // === Giải phóng direct blocks ===
  for(i = 0; i < NDIRECT; i++){
    if(ip->addrs[i]){
      bfree(ip->dev, ip->addrs[i]);
      ip->addrs[i] = 0;
    }
  }

  // === Giải phóng singly-indirect blocks ===
  if(ip->addrs[NDIRECT]){
    bp = bread(ip->dev, ip->addrs[NDIRECT]);
    a = (uint*)bp->data;
    for(j = 0; j < NINDIRECT; j++){
      if(a[j])
        bfree(ip->dev, a[j]);
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[NDIRECT]);
    ip->addrs[NDIRECT] = 0;
  }

  // === Giải phóng doubly-indirect blocks ===
  
  ...

  ip->size = 0;
  iupdate(ip);
}
```

#### Giải thích luồng giải phóng:

```
Giải phóng doubly-indirect = 3 tầng (từ trong ra ngoài):

Tầng 3 (trong cùng): bfree(data blocks)
    ↑
Tầng 2 (giữa):       bfree(singly-indirect blocks)
    ↑
Tầng 1 (ngoài cùng): bfree(doubly-indirect block)

Thứ tự: Phải giải phóng data blocks TRƯỚC,
         rồi mới giải phóng indirect blocks,
         cuối cùng mới giải phóng doubly-indirect block.
         (Vì nếu free block cha trước, ta mất pointer đến block con)
```

**Lưu ý quan trọng:**
- Mỗi `bread()` phải có `brelse()` tương ứng (giải phóng buffer cache)
- Dùng 2 biến buffer riêng biệt: `bp` cho doubly-indirect, `bp2` cho singly-indirect
- Dùng 2 biến pointer riêng biệt: `a` cho doubly-indirect, `a2` cho singly-indirect

---

### Bước 5: Kiểm tra `kernel/param.h`

Trên branch `fs`, file đã được cấu hình sẵn:

```c
#ifdef LAB_FS
#define FSSIZE       200000  // size of file system in blocks
#endif
```

`FSSIZE = 200000` đủ lớn để chứa file 65803 blocks. Không cần sửa gì thêm.

---

## ⚠️ LƯU Ý QUAN TRỌNG

### 1. Luôn `brelse()` sau `bread()`
Mỗi lần gọi `bread()` để đọc block vào buffer cache, PHẢI gọi `brelse()` để giải phóng. Nếu quên → buffer cache bị đầy → kernel panic.

### 2. Chỉ `log_write()` khi thay đổi nội dung block
Trong `bmap()`, chỉ gọi `log_write(bp)` khi ta ghi địa chỉ mới vào block (allocate block mới). Nếu chỉ đọc thì không cần.

### 3. Allocate on demand
Giống `bmap()` gốc, chỉ allocate block khi cần (lazy allocation). Nếu `balloc()` trả về 0 (hết disk space), phải xử lý đúng cách.

### 4. Rebuild fs.img sau khi sửa
Vì thay đổi `NDIRECT` ảnh hưởng đến `mkfs`, cần chạy:
```bash
$ make clean
$ make qemu
```
`make clean` sẽ xóa `fs.img` cũ, `make` sẽ build lại `mkfs` với `NDIRECT` mới và tạo `fs.img` mới.

### 5. Kiểm tra kết quả
Chạy trong xv6:
```
$ bigfile
...................................................................................
wrote 65803 blocks
done; ok
$ usertests
...
ALL TESTS PASSED
```

---