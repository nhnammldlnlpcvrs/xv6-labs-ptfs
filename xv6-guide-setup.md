# HƯỚNG DẪN CHI TIẾT - XV6 PROJECT 2: PAGE TABLE

## MỤC LỤC
1. [Setup Môi Trường](#1-setup-môi-trường)
2. [Git Workflow](#2-git-workflow)
3. [Implementation Phần 1: Speed up System Calls](#3-implementation-phần-1-speed-up-system-calls)
4. [Implementation Phần 2: Print Page Table](#4-implementation-phần-2-print-page-table)
5. [Testing](#5-testing)
6. [Submission](#6-submission)
7. [Troubleshooting](#7-troubleshooting)

---

## 1. SETUP MÔI TRƯỜNG

### 1.1 Ubuntu 24 (Khuyến nghị)

```bash
# Update system
sudo apt-get update && apt-get upgrade

# Install required tools
sudo apt-get install git build-essential gdb-multiarch \
    qemu-system-misc gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu
```

### 1.2 Windows (WSL 2)

**Bước 1: Cài WSL 2**
```powershell
# Mở PowerShell với quyền Administrator
wsl --install
wsl --set-default-version 2
```

**Bước 2: Cài Ubuntu 24.04**
- Mở Microsoft Store
- Tìm "Ubuntu 24.04"
- Click Install

**Bước 3: Kiểm tra version**
```powershell
wsl -l -v
# Phải thấy VERSION = 2
```

**Bước 4: Mở Ubuntu và cài tools**
```bash
sudo apt-get update && sudo apt-get upgrade
sudo apt-get install git build-essential gdb-multiarch \
    qemu-system-misc gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu
```

**Lưu ý WSL:**
- File của bạn nằm ở: `\\wsl$\Ubuntu-24.04\home\<username>\`
- Có thể access từ Windows Explorer bằng cách gõ `\\wsl$` vào address bar

### 1.3 macOS

```bash
# Install Xcode command line tools
xcode-select --install

# Install Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install RISC-V toolchain
brew tap riscv/riscv
brew install riscv-tools

# Update PATH (thêm vào ~/.bashrc hoặc ~/.zshrc)
echo 'export PATH=$PATH:/usr/local/opt/riscv-gnu-toolchain/bin' >> ~/.zshrc

# Install QEMU
brew install qemu
```

### 1.4 Kiểm tra cài đặt

```bash
# Kiểm tra QEMU
qemu-system-riscv64 --version
# Kết quả: QEMU emulator version 7.2.0 (hoặc cao hơn)

# Kiểm tra GCC
riscv64-linux-gnu-gcc --version
# Kết quả: riscv64-linux-gnu-gcc ... 10.3.0 (hoặc cao hơn)
```

---

## 2. GIT WORKFLOW

### 2.1 Clone repo xv6 từ MIT

```bash
# Clone repo
git clone git://g.csail.mit.edu/xv6-labs-2024
cd xv6-labs-2024

# Checkout branch pgtbl
git fetch
git checkout pgtbl
make clean
```

### 2.2 Tạo GitHub repo của bạn và kết nối

**Bước 1: Tạo repo trên GitHub**
- Vào github.com
- Click "New repository"
- Tên repo: `xv6-project2` (hoặc tên khác)
- **KHÔNG** chọn "Initialize with README"
- Click "Create repository"

**Bước 2: Add remote của bạn**
```bash
# Add remote của bạn (thay YOUR_USERNAME và YOUR_REPO_NAME)
git remote add myrepo https://github.com/YOUR_USERNAME/YOUR_REPO_NAME.git

# Kiểm tra remotes
git remote -v
# Kết quả:
# origin    git://g.csail.mit.edu/xv6-labs-2024 (fetch)
# origin    git://g.csail.mit.edu/xv6-labs-2024 (push)
# myrepo    https://github.com/YOUR_USERNAME/YOUR_REPO_NAME.git (fetch)
# myrepo    https://github.com/YOUR_USERNAME/YOUR_REPO_NAME.git (push)
```

### 2.3 Workflow khi code

```bash
# Sau khi code xong các file
git add .
git commit -m "Implement speed up system calls and print page table"

# Push lên GitHub của bạn
git push myrepo pgtbl

# Nếu gặp lỗi authentication, có thể dùng personal access token:
# Settings > Developer settings > Personal access tokens > Generate new token
# Khi push, dùng token làm password
```

### 2.4 Tạo patch file (để submit)

```bash
# Tạo patch so với original pgtbl branch
git diff origin/pgtbl > StudentID1_StudentID2_StudentID3.patch

# Hoặc nếu muốn xem tất cả changes từ khi checkout
git diff origin/pgtbl HEAD > StudentID1_StudentID2_StudentID3.patch

# Xem nội dung patch
cat StudentID1_StudentID2_StudentID3.patch
```

**Giải thích:**
- `origin/pgtbl`: branch gốc từ MIT
- `HEAD`: commit hiện tại của bạn
- Patch file sẽ chứa tất cả thay đổi bạn làm

---

## 3. IMPLEMENTATION PHẦN 1: SPEED UP SYSTEM CALLS

### 3.1 Tóm tắt yêu cầu

Tăng tốc system call `getpid()` bằng cách:
- Map một read-only page tại địa chỉ `USYSCALL`
- Lưu PID của process vào page này
- Userspace có thể đọc PID trực tiếp mà không cần kernel crossing

### 3.2 Hiểu về vấn đề

**Cách cũ (chậm):**
```
User program → trap vào kernel → kernel trả PID → return to user
```

**Cách mới (nhanh):**
```
User program → đọc trực tiếp từ shared page → có PID ngay
```

### 3.3 Code Implementation

#### File 1: kernel/proc.h

Thêm field `usyscall` vào struct `proc`:

```c
// Per-process state
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // wait_lock must be held when using this:
  struct proc *parent;         // Parent process

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct usyscall *usyscall;   // THÊM DÒNG NÀY - data shared with userspace
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
};
```

#### File 2: kernel/proc.c - Function allocproc()

Sửa function `allocproc()` để allocate usyscall page:

```c
static struct proc*
allocproc(void)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++) {
    acquire(&p->lock);
    if(p->state == UNUSED) {
      goto found;
    } else {
      release(&p->lock);
    }
  }
  return 0;

found:
  p->pid = allocpid();
  p->state = USED;

  // Allocate a trapframe page.
  if((p->trapframe = (struct trapframe *)kalloc()) == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // ═══════════════════════════════════════════════════════════
  // THÊM ĐOẠN NÀY - Allocate a usyscall page
  // ═══════════════════════════════════════════════════════════
  if((p->usyscall = (struct usyscall *)kalloc()) == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }
  // Initialize usyscall page with PID
  p->usyscall->pid = p->pid;
  // ═══════════════════════════════════════════════════════════

  // An empty user page table.
  p->pagetable = proc_pagetable(p);
  if(p->pagetable == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // Set up new context to start executing at forkret,
  // which returns to user space.
  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;

  return p;
}
```

#### File 3: kernel/proc.c - Function proc_pagetable()

Map usyscall page vào user page table:

```c
pagetable_t
proc_pagetable(struct proc *p)
{
  pagetable_t pagetable;

  // An empty page table.
  pagetable = uvmcreate();
  if(pagetable == 0)
    return 0;

  // map the trampoline code (for system call return)
  // at the highest user virtual address.
  // only the supervisor uses it, on the way
  // to/from user space, so not PTE_U.
  if(mappages(pagetable, TRAMPOLINE, PGSIZE,
              (uint64)trampoline, PTE_R | PTE_X) < 0){
    uvmfree(pagetable, 0);
    return 0;
  }

  // map the trapframe page just below the trampoline page, for
  // trampoline.S.
  if(mappages(pagetable, TRAPFRAME, PGSIZE,
              (uint64)(p->trapframe), PTE_R | PTE_W) < 0){
    uvmunmap(pagetable, TRAMPOLINE, 1, 0);
    uvmfree(pagetable, 0);
    return 0;
  }

  // ═══════════════════════════════════════════════════════════
  // THÊM ĐOẠN NÀY - map the usyscall page below the trapframe
  // PTE_R: readable, PTE_U: user accessible
  // Không có PTE_W vì read-only
  // ═══════════════════════════════════════════════════════════
  if(mappages(pagetable, USYSCALL, PGSIZE,
              (uint64)(p->usyscall), PTE_R | PTE_U) < 0){
    uvmunmap(pagetable, TRAPFRAME, 1, 0);
    uvmunmap(pagetable, TRAMPOLINE, 1, 0);
    uvmfree(pagetable, 0);
    return 0;
  }
  // ═══════════════════════════════════════════════════════════

  return pagetable;
}
```

#### File 4: kernel/proc.c - Function proc_freepagetable()

Unmap usyscall page khi free page table:

```c
void
proc_freepagetable(pagetable_t pagetable, uint64 sz)
{
  uvmunmap(pagetable, TRAMPOLINE, 1, 0);
  uvmunmap(pagetable, TRAPFRAME, 1, 0);
  uvmunmap(pagetable, USYSCALL, 1, 0);  // THÊM DÒNG NÀY
  uvmfree(pagetable, sz);
}
```

#### File 5: kernel/proc.c - Function freeproc()

Free usyscall page khi free process:

```c
static void
freeproc(struct proc *p)
{
  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;
  
  // ═══════════════════════════════════════════════════════════
  // THÊM ĐOẠN NÀY - Free usyscall page
  // ═══════════════════════════════════════════════════════════
  if(p->usyscall)
    kfree((void*)p->usyscall);
  p->usyscall = 0;
  // ═══════════════════════════════════════════════════════════
  
  if(p->pagetable)
    proc_freepagetable(p->pagetable, p->sz);
  p->pagetable = 0;
  p->sz = 0;
  p->pid = 0;
  p->parent = 0;
  p->name[0] = 0;
  p->chan = 0;
  p->killed = 0;
  p->xstate = 0;
  p->state = UNUSED;
}
```

### 3.4 Giải thích chi tiết

**Tại sao phải có USYSCALL page?**
- Thay vì mỗi lần gọi `getpid()` phải trap vào kernel (chậm)
- User code có thể đọc trực tiếp từ một page được share (nhanh)

**Permission bits: `PTE_R | PTE_U`**
- `PTE_R`: Page có thể đọc (Read)
- `PTE_U`: User mode có thể access
- KHÔNG có `PTE_W`: User không thể ghi (Read-only)

**Lifecycle của usyscall page:**
1. `allocproc()`: Allocate page và store PID
2. `proc_pagetable()`: Map page vào virtual address USYSCALL
3. User code đọc PID từ USYSCALL address
4. `proc_freepagetable()`: Unmap page khi process exit
5. `freeproc()`: Free physical memory

---

## 4. IMPLEMENTATION PHẦN 2: PRINT PAGE TABLE

### 4.1 Tóm tắt yêu cầu

Viết function `vmprint()` để:
- In ra cấu trúc page table của process
- Hiển thị PTE (Page Table Entry) với format:
  - Depth (số dấu `..`)
  - Index
  - PTE value
  - Physical address

### 4.2 Hiểu về RISC-V Page Table

**3-level page table:**
```
Page Table Level 2 (512 entries)
  └─> Page Table Level 1 (512 entries)
       └─> Page Table Level 0 (512 entries)
            └─> Physical Page
```

**PTE (Page Table Entry):**
- Contains physical page number + flags
- Valid bit: PTE_V
- Permission bits: PTE_R, PTE_W, PTE_X, PTE_U

### 4.3 Code Implementation

#### File 1: kernel/defs.h

Thêm prototype của `vmprint()`:

```c
// vm.c
void            kvminit(void);
void            kvminithart(void);
void            kvmmap(pagetable_t, uint64, uint64, uint64, int);
int             mappages(pagetable_t, uint64, uint64, uint64, int);
pagetable_t     uvmcreate(void);
void            uvmfirst(pagetable_t, uchar *, uint);
uint64          uvmalloc(pagetable_t, uint64, uint64, int);
uint64          uvmdealloc(pagetable_t, uint64, uint64);
int             uvmcopy(pagetable_t, pagetable_t, uint64);
void            uvmfree(pagetable_t, uint64);
void            uvmunmap(pagetable_t, uint64, uint64, int);
void            uvmclear(pagetable_t, uint64);
pte_t *         walk(pagetable_t, uint64, int);
uint64          walkaddr(pagetable_t, uint64);
int             copyout(pagetable_t, uint64, char *, uint64);
int             copyin(pagetable_t, char *, uint64, uint64);
int             copyinstr(pagetable_t, char *, uint64, uint64);
void            vmprint(pagetable_t);  // THÊM DÒNG NÀY
```

#### File 2: kernel/vm.c

Thêm 2 functions: `vmprint_helper()` và `vmprint()`:

```c
// ═══════════════════════════════════════════════════════════
// THÊM 2 FUNCTIONS NÀY VÀO CUỐI FILE kernel/vm.c
// ═══════════════════════════════════════════════════════════

// Helper function để recursively print page table
// level: độ sâu trong tree (1 = top level, 2 = middle, 3 = bottom)
void
vmprint_helper(pagetable_t pagetable, int level)
{
  // Mỗi page table có 512 PTEs (2^9)
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    
    // Chỉ print PTE nếu valid bit được set
    if(pte & PTE_V){
      // Print indentation based on level
      for(int j = 0; j < level; j++){
        printf(" ..");
      }
      
      // Print PTE information
      // %d: index in page table
      // %p: PTE value (64-bit hex)
      // %p: physical address extracted from PTE
      printf("%d: pte %p pa %p\n", i, pte, PTE2PA(pte));
      
      // Nếu PTE trỏ tới page table level thấp hơn (không phải leaf page)
      // thì recurse
      // Leaf page có ít nhất một trong các bit: R, W, X
      // Non-leaf page chỉ có V bit và không có R, W, X
      if((pte & (PTE_R|PTE_W|PTE_X)) == 0){
        // PTE này trỏ tới page table ở level thấp hơn
        uint64 child = PTE2PA(pte);
        vmprint_helper((pagetable_t)child, level + 1);
      }
    }
  }
}

// Main function để print page table
void
vmprint(pagetable_t pagetable)
{
  printf("page table %p\n", pagetable);
  vmprint_helper(pagetable, 1);
}
```

#### File 3: kernel/exec.c

Gọi `vmprint()` khi init process được exec:

```c
int
exec(char *path, char **argv)
{
  char *s, *last;
  int i, off;
  uint64 argc, sz = 0, sp, ustack[MAXARG], stackbase;
  struct elfhdr elf;
  struct inode *ip;
  struct proghdr ph;
  pagetable_t pagetable = 0, oldpagetable;
  struct proc *p = myproc();

  // ... existing code ...

  // ═══════════════════════════════════════════════════════════
  // THÊM ĐOẠN NÀY - In page table của process đầu tiên (init)
  // Đặt TRƯỚC "return argc;" ở cuối function
  // ═══════════════════════════════════════════════════════════
  if(p->pid==1)
    vmprint(p->pagetable);
  // ═══════════════════════════════════════════════════════════

  return argc;

 bad:
  if(pagetable)
    proc_freepagetable(pagetable, sz);
  if(ip){
    iunlockput(ip);
    end_op();
  }
  return -1;
}
```

### 4.4 Giải thích chi tiết

**Cách hoạt động của vmprint_helper:**

1. **Loop qua 512 PTEs:** Mỗi page table có 512 entries
2. **Check valid bit:** Chỉ print nếu `PTE_V` được set
3. **Print indentation:** Số dấu `..` = level (depth)
4. **Print PTE info:**
   - Index: vị trí trong page table
   - PTE value: giá trị 64-bit của entry
   - PA: physical address (extract từ PTE bằng macro `PTE2PA`)

5. **Recurse nếu cần:**
   - Nếu `(pte & (PTE_R|PTE_W|PTE_X)) == 0`: đây là non-leaf PTE
   - Non-leaf PTE trỏ tới page table ở level thấp hơn
   - Recurse xuống level đó

**Output mẫu:**
```
page table 0x0000000087f6b000
..0: pte 0x0000000021fd9c01 pa 0x0000000087f67000
.. ..0: pte 0x0000000021fd9801 pa 0x0000000087f66000
.. .. ..0: pte 0x0000000021fda01b pa 0x0000000087f68000
.. .. ..1: pte 0x0000000021fd9417 pa 0x0000000087f65000
```

**Giải thích output:**
- `..0`: Level 2, index 0
- `.. ..0`: Level 1, index 0
- `.. .. ..0`: Level 0, index 0 (leaf page)

---

## 5. TESTING

### 5.1 Build và chạy xv6

```bash
# Build xv6
make qemu

# Kết quả sẽ thấy:
# - Page table được in ra khi boot (từ vmprint)
# - Prompt $
```

### 5.2 Test từng phần

**Test phần 1 (Speed up system calls):**
```bash
$ pgtbltest
```

Kết quả mong đợi:
```
ugetpid_test starting
ugetpid_test: OK
```

**Test phần 2 (Print page table):**

Khi boot xv6, bạn sẽ thấy output như:
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

### 5.3 Test tổng thể

```bash
$ make grade
```

Kết quả mong đợi cho phần page table:
```
== Test pte printout == 
pte printout: OK 
== Test answers-pgtbl.txt == 
answers-pgtbl.txt: OK 
== Test count copyin == 
count copyin: OK 
== Test usertests == 
usertests: OK
```

### 5.4 Thoát QEMU

```
Ctrl-A X
```

Giải thích: Nhấn `Ctrl` và `A` cùng lúc, sau đó nhấn `X`

---

## 6. SUBMISSION

### 6.1 Tạo patch file

```bash
# Đảm bảo đã commit tất cả changes
git add .
git commit -m "Complete page table implementation"

# Tạo patch file
git diff origin/pgtbl > 2312001_2312002_2312003.patch

# Kiểm tra patch file
cat 2312001_2312002_2312003.patch
```

### 6.2 Tạo clean version của xv6

```bash
# Clean build artifacts
make clean

# Tạo zip file
cd ..
zip -r xv6_page_table.zip xv6-labs-2024
```

### 6.3 Tạo cấu trúc folder submission

```bash
# Tạo folder structure
mkdir -p page_table

# Copy files
cp xv6-labs-2024/2312001_2312002_2312003.patch page_table/
cp xv6_page_table.zip page_table/

# Tạo report (dùng Word/Google Docs rồi export PDF)
# Copy vào page_table/2312001_2312002_2312003_Report.pdf

# Zip final submission
zip -r 2312001_2312002_2312003.zip page_table/
```

### 6.4 Nội dung Report

Report nên bao gồm:

1. **Tóm tắt implementation:**
   - Phần 1: Giải thích cách implement USYSCALL page
   - Phần 2: Giải thích cách implement vmprint

2. **Câu hỏi trong đề:**
   
   **Q1: Which other xv6 system call(s) could be made faster using this shared page?**
   
   Trả lời:
   - `getppid()`: Lấy parent PID
   - `uptime()`: Lấy thời gian hệ thống đã chạy
   - Các system call read-only khác không thay đổi state

   **Q2: Explain what each leaf page logically contains and its permission bits:**
   
   Phân tích từng leaf page trong output vmprint:
   - Entry 0, 1, 2, 3: Code và data của process
   - Entry 509, 510: User stack
   - Entry 511: Trapframe/Trampoline
   - Permission bits: R (read), W (write), X (execute), U (user)

3. **Vấn đề gặp phải và cách giải quyết**

4. **Test results**

---

## 7. TROUBLESHOOTING

### 7.1 Compilation Errors

**Error: `undefined reference to vmprint`**
```
Solution: Kiểm tra đã thêm prototype vào kernel/defs.h chưa
```

**Error: `'struct proc' has no member named 'usyscall'`**
```
Solution: Kiểm tra đã thêm field vào kernel/proc.h chưa
```

### 7.2 Runtime Errors

**Error: `panic: freewalk: leaf`**
```
Nguyên nhân: Quên unmap usyscall page trong proc_freepagetable()
Solution: Thêm uvmunmap(pagetable, USYSCALL, 1, 0);
```

**Error: `pgtbltest fails`**
```
Nguyên nhân: 
1. Chưa initialize p->usyscall->pid
2. Sai permission bits (phải là PTE_R | PTE_U)
3. Quên free usyscall page

Solution: Kiểm tra lại từng bước trong allocproc(), proc_pagetable(), freeproc()
```

### 7.3 Git Issues

**Error: `Permission denied (publickey)`**
```bash
# Solution: Dùng HTTPS thay vì SSH
git remote set-url myrepo https://github.com/YOUR_USERNAME/YOUR_REPO.git

# Hoặc setup SSH key:
ssh-keygen -t ed25519 -C "your_email@example.com"
cat ~/.ssh/id_ed25519.pub  # Copy và add vào GitHub Settings > SSH Keys
```

**Error: `Updates were rejected`**
```bash
# Solution: Pull trước khi push
git pull myrepo pgtbl --rebase
git push myrepo pgtbl
```

### 7.4 WSL Issues

**Error: `cannot execute binary file`**
```
Nguyên nhân: Đang chạy Windows binary trong WSL
Solution: Compile lại trong WSL Ubuntu
```

**File permission issues:**
```bash
# Fix permissions
chmod +x script_name.sh
```

---

## PHỤ LỤC: MACROS QUAN TRỌNG

### kernel/riscv.h

```c
// Extract physical address from PTE
#define PTE2PA(pte) (((pte) >> 10) << 12)

// PTE flags
#define PTE_V (1L << 0)  // Valid
#define PTE_R (1L << 1)  // Read
#define PTE_W (1L << 2)  // Write
#define PTE_X (1L << 3)  // Execute
#define PTE_U (1L << 4)  // User can access
```

### kernel/memlayout.h

```c
// Virtual addresses
#define USYSCALL (TRAPFRAME - PGSIZE)
#define TRAPFRAME (TRAMPOLINE - PGSIZE)
#define TRAMPOLINE (MAXVA - PGSIZE)

// Page size
#define PGSIZE 4096
```

---

## CHECKLIST TRƯỚC KHI SUBMIT

- [ ] Code compile thành công (`make clean && make qemu`)
- [ ] `pgtbltest` pass
- [ ] Thấy page table được in ra khi boot
- [ ] Đã tạo patch file đúng format
- [ ] Đã tạo clean zip của xv6
- [ ] Đã viết report PDF
- [ ] Folder structure đúng theo yêu cầu
- [ ] Filename đúng format: StudentID1_StudentID2_StudentID3.zip
- [ ] Đã push code lên GitHub của mình (backup)

---

**Good luck! 🚀**

**Tham khảo thêm:**
- xv6 book Chapter 3: https://pdos.csail.mit.edu/6.S081/2024/xv6/book-riscv-rev4.pdf
- RISC-V privileged spec: https://riscv.org/technical/specifications/