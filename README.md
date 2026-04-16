# AOS Lab3 — Noarr Structures 論文研討與重現

**論文：** *Approach to Optimized Parallel Traversal of Regular Data Structures*  
**論文作者：** Jiří Klepl, Adam Šmelko, Lukáš Rozsypal, Martin Kruliš  

---

## 論文簡介

本次重現的論文提出以 C++ 為基礎的高效能運算抽象框架 **Noarr Structures**，是一個 Header-only Library。其核心是透過抽象化的資料結構與遍歷（traversal）機制，能在不修改核心演算法的情況下切換記憶體佈局（如 row-major、column-major、Z-order 等）。

**研究重點：**
1. 提供統一的資料結構建模方式（layout-agnostic structures）
2. 透過 Noarr Traversers 實現彈性且平行化的資料存取模式
3. 驗證此方法在 PolyBench 基準測試中能保持良好效能與可重現性

---

## 研究背景與動機

現代記憶體與儲存系統在高效能運算環境下面臨嚴重的 memory bottleneck problems：

| 問題 | 說明 |
|------|------|
| Cache miss | 快取未命中問題 |
| Memory Latency | 記憶體存取延遲 |
| Memory Bandwidth | 記憶體頻寬瓶頸 |
| Data Locality | 資料存取缺乏良好局部性 |

傳統資料結構在面對大型或高維度資料時，常無法達到最佳效能。Noarr 的核心改善方式：**不是只加快運算，而是改變資料存取方式**。

---

## 核心概念

Noarr 將資料的結構與其在記憶體中的實際配置進行**解耦（decoupling）**，開發者不需改變程式邏輯，就能改變資料在記憶體中的佈局方式。

| 特性 | 說明 |
|------|------|
| 抽象層設計 | Abstraction Layer |
| 彈性資料定位 | Flexible Layout Mapping |
| 多維資料結構支援 | Multi-dimensional Support |
| 統一建模 | Layout-agnostic structures |
| 彈性遍歷 | Noarr Traversers 平行化存取 |
| 效能驗證 | PolyBench 基準測試 |

與傳統 Array 的差異：
- 傳統：資料結構固定
- Noarr：資料排布可動態調整，依照不同演算法需求提高快取命中率

---

## 實驗環境建置

### 虛擬機配置

在 VMware 中建立 Ubuntu 22.04 LTS 虛擬機，並進行系統初始化：

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y build-essential cmake git python3 python3-pip
```

### 磁碟空間擴充（遇到空間不足時）

> **重要提醒：** 在進行大型專案編譯前，務必確保有足夠的磁碟空間，建議預留至少 20GB 以上。

```bash
sudo pvdisplay
sudo pvresize /dev/sda3
sudo vgdisplay
sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
```

---

## 重現步驟

### 第一階段：PolyBenchC-Noarr 實驗

#### STEP 1. 下載原始碼

```bash
mkdir ~/noarr_project
cd ~/noarr_project
git clone https://github.com/jiriklepl/ParCo2024-artifact.git
```

#### STEP 2. 編譯 CPU 實驗

```bash
cd ~/noarr_project/ParCo2024-artifact/PolybenchC-Noarr
vim CMakeLists.txt
```

在 `project(...)` 下方加入：

```cmake
add_definitions(-DSMALL_DATASET -DDATA_TYPE_IS_FLOAT)
```

```bash
mkdir build
cd build
cmake ..
make -j2
```

#### STEP 3. 執行 CPU 實驗

```bash
ls
./2mm   # 每個可執行檔都可以嘗試執行
```

---

### 第二階段：Noarr Structures 函式庫驗證

#### STEP 1. 下載原始碼

```bash
cd ~/noarr_project
git clone https://github.com/ParaCoToUl/noarr-structures.git
cd ~/noarr_project/noarr-structures/examples/matrix
```

#### STEP 2. 建立 build 資料夾並編譯

```bash
mkdir build && cd build
cmake ..
make -j2
./matrix
```

支援使用者選擇矩陣佈局（rows / columns / z_curve）：

```bash
# row-major 布局、4x4 矩陣
./matrix rows 4

# column-major 布局、8x8 矩陣
./matrix columns 8

# Z-curve 布局、8x8 矩陣
./matrix z_curve 8
```

#### STEP 3. 執行官方單元測試（Tests）

```bash
cd ~/noarr_project/noarr-structures/tests
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build --config Debug
ctest --test-dir build -C Debug -V
```

**測試結果：**
- Tests: **289**
- Failures: **0**
- Assertions: **3,486,650**
- **100% tests passed, 0 tests failed out of 1**

---

### 第三階段：Z-Curve 矩陣乘法實作

將 Z-curve 邏輯整合至 Noarr 矩陣運算系統：

#### 1. 修改 `matrix.cpp`

引入 Z-curve 標頭檔：

```cpp
#include <noarr/structures/structs/zcurve.hpp>
```

移除或註解第 20-22 行的舊版 Z-curve 定義，避免衝突。

#### 2. 修正 `main` 函式

- Z-curve 要求矩陣大小必須是 2 的次方且 ≥ 1
- 定義 `matrix_z_tag` 作為 Z-curve 專用結構標記
- 呼叫 `matrix_demo` 時加入第三個參數 `true` 標記 Z-curve 模式

#### 3. 更新 `matrix_demo` 函式

```cpp
template<class Structure>
void matrix_demo(std::size_t size, Structure structure,
                 bool is_z_curve = false)
```

#### 4. 修改 `noarr_matrix_functions.hpp`

核心 `noarr_matrix_multiply` 函式使用 `noarr::traverser` 取代傳統 `for` 迴圈，並根據 `is_z_curve` 標記選擇遍歷順序：

```cpp
// Z-CURVE 遍歷
constexpr std::size_t MAX_SIZE = 1024;
base_t
    .order(merge_zcurve<'n','m','z'>::maxlen_alignment<MAX_SIZE,1>())
    | compute_kernel;
```

#### 5. 重新編譯與執行

```bash
cmake --build .
./matrix z_curve 8
```

---

### 第四階段：Syntax for Traversers 與 Parallel Execution

#### 設定環境（關掉 CUDA）

```bash
cd ~/PMAM2024-artifact/running-examples
```

替換 `CMakeLists.txt` 內容（使用不含 CUDA 的版本）：

```cmake
cmake_minimum_required(VERSION 3.10)
include(FetchContent)
project(NoarrPolybenchExamples VERSION 0.0.1 LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED True)

if(NOT DEFINED NOARR_STRUCTURES_BRANCH)
    set(NOARR_STRUCTURES_BRANCH PMAM2024)
endif()

FetchContent_Declare(
    Noarr
    GIT_REPOSITORY https://github.com/jiriklepl/noarr-structures.git
    GIT_TAG ${NOARR_STRUCTURES_BRANCH}
)
FetchContent_MakeAvailable(Noarr)
include_directories(${Noarr_SOURCE_DIR}/include)

if (DEFINED ENV{TBBROOT}) link_directories($ENV{TBBROOT}/lib) endif ()

add_executable(matmul matmul.cpp)
add_executable(matmul-blocked matmul.cpp)
target_compile_definitions(matmul-blocked PRIVATE MATMUL_BLOCKED=1)
```

#### 編譯

```bash
mkdir -p build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
cmake --build . -j"$(nproc)"
```

#### 產生測試資料

```bash
cd ~/PMAM2024-artifact/running-examples
mkdir -p data
chmod +x gen-text.sh gen-matrices.py

# Histogram 用資料（亂數 bytes）
./gen-text.sh data/input1.bin 1000000

# Matmul 用資料（隨機矩陣）
./gen-matrices.py data/input2.bin 512
```

#### 執行 Traverser 版本比較

```bash
./matmul ../data/input2.bin 512 > ../data/matmul_base.txt
./matmul-blocked ../data/input2.bin 512 > ../data/matmul_blocked.txt
./matmul-factored ../data/input2.bin 512 > ../data/matmul_factored.txt
```

| 版本 | 技術重點 | Cache 效率 | 記憶體存取 | 效能 |
|------|---------|-----------|-----------|------|
| baseline | 三層迴圈 | 差 | N 次 load + N 次 store | 最慢 |
| blocked | 分塊（tiling） | 佳 | N 次 load + N 次 store | 快於 baseline |
| factored | 暫存器累積（因式分解） | 佳 | 1 次讀 + 1 次寫 | 最快 |

#### Parallel Execution — Histogram

```bash
# 單執行緒版
./histogram-cpp ../data/input1.bin $(stat -c%s ../data/input1.bin) > ../data/histo_cpu.txt

# TBB 平行版（tbb::parallel_reduce，每個 thread 有 local histogram）
./histogram-cpp-parallel ../data/input1.bin $(stat -c%s ../data/input1.bin) > ../data/histo_tbb.txt
```

---

### 第五階段：GEMM 效能分析實驗

> **注意：** 此階段需要實體機才能使用 `perf stat` 監看 cache hit/miss，虛擬機無法觀測快取效能指標。

```bash
sudo apt install -y linux-tools-common linux-tools-generic
cd ~/gemm_experiment
cmake -S . -B build
cmake --build build -j
sudo ./build/gemm_experiment
```

比較三種矩陣乘法實作：

| 實作 | 說明 |
|------|------|
| Baseline | 傳統 C++ 三重迴圈，row-major 順序，加入小優化快取 A[i,k] |
| Noarr Simple | 使用 Noarr Traverser 搭配 `hoist` 改變迴圈順序（i-k-j），無 tiling |
| Noarr Tiled | 加入 `strip_mine(16)` 進行三維度 tiling，切成 16×16 小區塊 |

**效能結果（N=512）：**

| 版本 | 平均執行時間 |
|------|------------|
| Noarr Simple | 101 ms |
| Noarr Tiled | 13.4 ms |
| 效能提升 | **7.5x** |

**規模分析：**

- **小矩陣（N=256、512）：** Noarr Tiled 反而比 Baseline 更快。16×16×16 block 完全放進快取，幾乎無 cache miss。
- **大矩陣（N=1024）：** 抽象開銷超過快取利益。需要 262,144 個 tile，每個 tile 都產生狀態推導、offset 計算、bag indexing 開銷，效能反而下降約 2 倍。

---

## 實驗結果總覽

| 測試項目 | 說明 | 結果 |
|---------|------|------|
| 編譯 PolyBenchC-Noarr | 成功編譯並生成多個可執行 benchmark | 通過 |
| 範例執行 Noarr Structures | 成功執行矩陣範例（rows/columns/z_curve） | 通過 |
| 官方單元測試（ctest） | 共 289 測試案例，全數通過 | 通過（289/289）|
| Z-Curve 矩陣乘法 | 成功整合 Z-curve 遍歷至矩陣運算 | 通過 |
| Traverser 效能比較 | factored > blocked > baseline | 完成 |
| Parallel Execution | TBB 版本伸縮性優於單執行緒 | 完成 |
| GEMM 效能分析 | Tiled 版在 N≤512 時提升 7.5x | 完成 |

---

## 研究結論

1. Noarr 的程式轉換機制可以在不需要人工修改迴圈結構的前提下，自動產生效能良好的程式碼。
2. 結構化的資料布局改善 cache locality。
3. 透過暫存器累積減少記憶體 load/store 次數。
4. **Traversal 與 layout 解耦**可提升彈性，且可直接支援 GPU / 多核心，為 autotuning / AI 優化鋪路。
5. Noarr 框架證明「演算法本體不變，只靠改變 traversal 就可調整效能」的優點，但需注意規模限制（大型矩陣 N=1024 時抽象開銷可能超過優化效益）。

---

## 未來展望

- 整合 GPU 加速
- 應用於 AI / Machine Learning
- 測試更大規模資料
- 應用於資料庫與雲端系統
- 自動搜尋最佳 layout（機器學習 + autotuning）

---

## 參考文獻

- ACM 論文：https://dl.acm.org/doi/10.1145/3649169.3649247
- ScienceDirect 論文：https://www.sciencedirect.com/science/article/pii/S0167819124000346
- GitHub Source Code (noarr-structures)：https://github.com/ParaCoToUl/noarr-structures
- GitHub Source Code (ParCo2024 artifact)：https://github.com/jiriklepl/ParCo2024-artifact
