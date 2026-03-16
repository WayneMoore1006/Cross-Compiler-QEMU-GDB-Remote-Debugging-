# Lab 2：Cross Compiler、QEMU 與 GDB Remote Debugging 實驗

# 📌 實驗簡介

本實驗主要介紹如何在 Linux 環境下：

- 建立 **ARM Cross Compiler 編譯環境**
- 使用 **GDB 進行 source-level debugging**
- 使用 **QEMU 模擬 ARM 程式執行**
- 利用 **GDB 與 QEMU 進行 remote debugging**
- 了解 **GDB Remote Serial Protocol** 的基本概念

---

# 🎯 實驗目的

1. 學習如何建立 ARM cross compiler 環境  
2. 了解如何使用 GDB 進行 source-level debugging  
3. 使用 QEMU 模擬 ARM 環境執行 cross compiled 程式  
4. 學習透過 remote debugging 方式，利用 GDB 連接 QEMU 進行除錯  
5. 了解 GDB remote serial protocol 的基本運作方式  

---

# 🖥 實驗環境

作業系統：

```
Linux (Ubuntu)
```

工作目錄：

```
/home/wayne/WORK
```

Cross Compiler：

```
/home/wayne/WORK/crossgcc2/bin/arm-linux-gnueabihf-gcc
```

Cross GDB：

```
/home/wayne/WORK/crossgcc2/bin/arm-linux-gnueabihf-gdb
```

---

# 📂 實驗檔案

本實驗會使用以下檔案：

```
test.c
test.exe
```

---

# 1️⃣ 建立測試程式

建立 `test.c`

```c
#include <math.h>
#include <stdio.h>

int main(void)
{
    int a;
    double b;

    a = 10;
    b = 20 + cos((double)a);
    printf("%f\n", b);

    return 0;
}
```

建立方式：

```bash
cd /home/wayne/WORK
nano test.c
```

---

# 2️⃣ 使用 ARM Cross Compiler 編譯

輸入：

```bash
/home/wayne/WORK/crossgcc2/bin/arm-linux-gnueabihf-gcc -static -g test.c -o test.exe -lm
```

參數說明：

| 參數 | 說明 |
|---|---|
| -static | 靜態連結 |
| -g | 加入除錯資訊 |
| -o | 輸出檔案名稱 |
| -lm | 連結 math library |

確認檔案：

```bash
file test.exe
```

若成功會看到：

```
ELF 32-bit LSB executable, ARM
```

---

# 3️⃣ 安裝 QEMU

安裝 ARM user emulator

```bash
sudo apt-get update
sudo apt-get install qemu-user
```

確認是否安裝成功：

```bash
qemu-arm -help
```

---

# 4️⃣ 使用 QEMU 執行 ARM 程式

```bash
qemu-arm ./test.exe
```

預期輸出：

```
19.160928
```

因為程式中：

```
b = 20 + cos(10)
```

---

# 5️⃣ 啟動 QEMU GDB Server

讓 QEMU 等待 GDB 連線

```bash
qemu-arm -g 12345 ./test.exe
```

說明：

| 參數 | 說明 |
|---|---|
| -g 12345 | 在 port 12345 啟動 gdbserver |

Terminal 會停住等待 GDB 連線，這是 **正常現象**。

---

# 6️⃣ 使用 GDB Remote Debugging

開啟 **另一個 Terminal**

```bash
cd /home/wayne/WORK
/home/wayne/WORK/crossgcc2/bin/arm-linux-gnueabihf-gdb test.exe
```

進入 GDB 後輸入：

```gdb
target remote localhost:12345
break main
continue
```

---

# 7️⃣ 常用 GDB 指令

```gdb
next
print a
print b
```

簡寫版本：

```gdb
n
p a
p b
c
q
```

指令表：

| 指令 | 說明 |
|---|---|
| break main | 設定中斷點 |
| continue | 繼續執行 |
| next | 執行下一行 |
| step | 進入函式 |
| print | 印出變數 |
| quit | 離開 GDB |

---

# 8️⃣ 完整實驗流程

Terminal 1：

```bash
cd /home/wayne/WORK
qemu-arm -g 12345 ./test.exe
```

Terminal 2：

```bash
cd /home/wayne/WORK
/home/wayne/WORK/crossgcc2/bin/arm-linux-gnueabihf-gdb test.exe
```

GDB 內：

```gdb
target remote localhost:12345
break main
continue
next
print a
next
print b
```

---

# 9️⃣ Remote Serial Protocol

當 GDB 使用：

```gdb
target remote localhost:12345
```

時，GDB 與 QEMU 之間會透過 **Remote Serial Protocol (RSP)** 溝通。

封包格式：

```
$packet-data#checksum
```

這個 ASCII protocol 可以用來：

- 設定 breakpoint
- 讀取暫存器
- 讀寫記憶體
- 控制程式執行

---

# 🔍 問題與討論

## 1️⃣ What is JTAG？

JTAG（Joint Test Action Group）是一種硬體除錯與測試介面標準，標準為 **IEEE 1149.1**。

在嵌入式系統開發中，JTAG 可以讓開發者透過硬體連接直接控制 CPU。

JTAG 主要用途：

- 硬體除錯（Hardware Debugging）
- 晶片測試（Boundary Scan）
- 韌體下載（Firmware Programming）
- 檢查 CPU 暫存器與記憶體

透過 JTAG，開發者可以：

- 暫停 CPU
- 設定 breakpoint
- 單步執行程式
- 檢查系統內部狀態

因此 JTAG 是嵌入式開發中非常重要的硬體除錯工具。

---

## 2️⃣ What is GDB server？What is GDB stub？Difference？

### GDB server

GDB server 是一個在 **目標系統上執行的程式**，負責接收 GDB 的除錯指令並控制程式執行。

例如：

- 設定 breakpoint
- 讀取暫存器
- 讀取記憶體
- 控制程式繼續或停止

在本實驗中：

```
qemu-arm -g 12345 ./test.exe
```

QEMU 就提供了 **gdbserver 功能**。

---

### GDB stub

GDB stub 是嵌入在 **目標程式或系統內部的除錯程式碼**。

它會實作 **GDB remote serial protocol**，並在程式發生 breakpoint 或 exception 時與 GDB 溝通。

常見函式：

```
set_debug_traps()
breakpoint()
```

通常使用在：

- bare-metal 系統
- bootloader
- 作業系統 kernel

---

### GDB server 與 GDB stub 差異

| 項目 | GDB Server | GDB Stub |
|---|---|---|
| 型態 | 獨立程式 | 內嵌程式碼 |
| 執行位置 | 目標系統 | 程式內部 |
| 使用場景 | Linux / QEMU | Bare-metal |
| 功能 | 轉接 GDB 指令 | 直接處理 GDB 指令 |

簡單理解：

- **GDB Server = 外部除錯服務**
- **GDB Stub = 程式內建除錯機制**

兩者都可以讓 GDB 進行 remote debugging。

---

# 🧠 實驗心得

透過本次實驗，我學會如何建立 ARM cross compiler 編譯環境，並利用 QEMU 在 x86 Linux 環境中模擬執行 ARM 程式。此外，我也學會使用 GDB 進行 remote debugging，透過 `target remote` 指令連接 QEMU，並利用 breakpoint、next、print 等指令觀察程式執行過程。

本實驗也讓我更了解 JTAG、GDB server、GDB stub 以及 Remote Serial Protocol 的概念，對嵌入式系統除錯流程有更完整的理解。

---

# 📚 指令速查表

```bash
cd /home/wayne/WORK

# 編譯
/home/wayne/WORK/crossgcc2/bin/arm-linux-gnueabihf-gcc -static -g test.c -o test.exe -lm

# 執行
qemu-arm ./test.exe

# 啟動 gdb server
qemu-arm -g 12345 ./test.exe

# 啟動 gdb
/home/wayne/WORK/crossgcc2/bin/arm-linux-gnueabihf-gdb test.exe
```

GDB 指令：

```gdb
target remote localhost:12345
break main
continue
next
print a
print b
quit
```

---

# ✅ 結論

本實驗成功完成：

- ARM Cross Compilation
- QEMU ARM 模擬執行
- GDB source-level debugging
- Remote debugging
- 理解 JTAG、GDB server、GDB stub 與 Remote Serial Protocol

這些技術是 **嵌入式系統開發與除錯的重要工具**。
