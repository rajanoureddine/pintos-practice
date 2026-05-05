> **Personal project** — Raja Noureddine. This is a personal learning repo and is not intended for public support.

# 🖥️ Modified PintOS Starter Code

This repository contains a **modified PintOS starter code** that can be compiled and run on **Ubuntu 22.04 / 24.04**.  
It includes setup instructions, configuration steps, and test commands to verify your environment.

---

## 📺 Setup Video
*Coming soon*

---

## 🛠️ System Information

Check your Ubuntu version:

```bash
lsb_release -a
```

---

## 📦 Install Dependencies

Update package lists:

```bash
sudo apt update
```

Install required build tools and libraries:

```bash
sudo apt install -y build-essential pkg-config zlib1g-dev libglib2.0-dev libc6-dev autoconf libtool libsdl1.2-dev libx11-dev libxrandr-dev libxi-dev perl make git qemu
```

Additionally, install QEMU system emulator:

```bash
sudo apt install -y qemu-system-x86
```

---

## 📂 Clone Repository

```bash
git clone https://github.com/dktpt44/pintos.git
cd pintos
```

---

## ⚙️ Configuration Steps

### Step 1: Configure `pintos-gdb`
Open `src/utils/pintos-gdb`.  
Run the following command in your terminal to get the current path:

```bash
pwd
```

Set `GDBMACROS` equal to:

```
GDBMACROS=/path/to/pintos/src/misc/gdb-macros
```

Save the file (`Ctrl+S`).

---

### Step 2: Update `Pintos.pm`
Open `src/utils/Pintos.pm`.  
At **line 362**, change:

```
loader.bin
```

to:

```
/path/to/pintos/src/threads/build/loader.bin
```

---

### Step 3: Compile Utilities
```bash
cd src/utils
make
```

---

### Step 4: Update `pintos` Script
Open `src/utils/pintos`.  
At **line 260**, replace:

```
kernel.bin
```

with:

```
/path/to/kernel.bin
```

---

### Step 5: Update PATH
Append PintOS utilities to your PATH:

```bash
echo 'export PATH=/home/nyuad/pintos/src/utils:$PATH' >> ~/.bashrc
source ~/.bashrc
```

Verify:

```bash
cat ~/.bashrc | tail -n 5
```

---

### Step 6: Compile Kernel
```bash
cd src/threads
make
```

---

## ✅ Verify Setup

Run:

```bash
pintos -v -- -q run alarm-multiple
```

If you have GUI support:

```bash
pintos run alarm-multiple
```

---

## 🧪 Running Tests

Navigate to the `threads/build` directory.

### Batch of Alarm Tests
```bash
make check TESTS="tests/threads/alarm-single tests/threads/alarm-simultaneous tests/threads/alarm-zero tests/threads/alarm-negative" VERBOSE=1
```

### Single Test
```bash
make check TESTS="tests/threads/alarm-multiple" VERBOSE=1
```

### Priority Tests
```bash
make check TESTS="tests/threads/alarm-priority tests/threads/priority-fifo tests/threads/priority-change tests/threads/priority-preempt" VERBOSE=1
```

---

## ⚖️ MLFQS Tests

### Run Single Test
```bash
pintos -v -- -q -mlfqs run mlfqs-load-1
make check TESTS="tests/threads/mlfqs-load-1" VERBOSE=1
```

### Run All MLFQS Tests
```bash
make check TESTS="tests/threads/mlfqs-load-1 tests/threads/mlfqs-load-60 tests/threads/mlfqs-fair-2 tests/threads/mlfqs-nice-2 tests/threads/mlfqs-nice-10 tests/threads/mlfqs-recent-1 tests/threads/mlfqs-load-avg" VERBOSE=1
```

---

## 🎯 Summary

- Install dependencies  
- Clone and configure PintOS  
- Update paths in `pintos-gdb`, `Pintos.pm`, and `pintos`  
- Compile utilities and kernel  
- Verify with `pintos run`  
- Run tests (`make check`)  

