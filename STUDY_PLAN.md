# 毫米波雷达学习指南（study 分支）

> 用法：这是路线图 + 进度表。完成一项就把 `[ ]` 改成 `[x]`，并在末尾「进度总表」更新日期。
> 每学完一个阶段 commit 一次本文件，学习轨迹就自动沉淀在 git 历史里。
>
> - 原仓库 `upstream`：StZhang-B/PKU-Millimeter-Wave-Radar-Tutorial（只读，同步更新用）
> - 学习分支 `study`：push 到 `origin`（hello-wangjj/Copy_PKU-Millimeter-Wave-Radar-Tutorial）
> - **永远不在 main 上直接改东西**，注释和实验都在 study 分支做

---

## 0. Git 工作流程

```bash
# 日常循环：加注释/改代码 → 提交 → 推送
git add -A
git commit -m "阶段2-ca_cfar: 推导门限因子α=N(Pfa^(-1/N)-1)"
git push

# 原作者更新了怎么办（当前没更新过，2025-04-19 后静止）
git fetch upstream
git merge upstream/main

# （可选）学完一个阶段后，在 MOT 主仓库记录 submodule 指针
cd D:/UGit/MOT && git add sub/PKU-Millimeter-Wave-Radar-Tutorial && git commit -m "chore(submodule): PKU教程 study 阶段X 完成"
```

**commit 信息约定**：`阶段N-主题: 一句话说清学到了什么/修了什么`。修 bug 单独提交，写清证据。

---

## 1. 仓库全景

整条链路：**ADC 原始数据 → 点云**

```
Chirp发射 → 混频得到中频信号 → 数据立方体(通道×chirp×采样点)
   → ADC维FFT=距离 ──┐
   → Chirp维FFT=速度 ├── 2D/3D谱 → CFAR检测 → 峰值凝聚 → DOA测角 → 点云(R,V,θ)
   → 天线维FFT=角度 ─┘
```

| 文件夹 | 角色 | 需要硬件 |
|---|---|---|
| `Basic Mmwave Signal Processing` | 理论核心：23页教程PDF + TI原理PDF + 329行完整仿真 | ❌ |
| `IWR1843+DCA1000 ADC Raw Data Capture` | 硬件采集教程（装软件、连板子、抓bin） | ✅ 跳过 |
| `基于点云的人体定位demo` | 最工程化的完整流水线（自带信号仿真） | ❌ |
| `基于热力图的人体定位demo` | 实测数据→Range-Azimuth热力图（与Basic/demo.m几乎相同） | ✅ |
| `生命体征检测demo` | 相位提取→解缠→带通滤波→呼吸/心跳频率 | ✅ |
| `手势识别demo` | 空壳，只有一个外链README | — |

**事实**：仓库不含任何 .bin 实测数据，三个实测 demo 只能读代码学思路；两份仿真代码覆盖完整链路，是学习主体。

---

## 2. 环境准备

- [ ] MATLAB R2020b 或更高（README 作者用的 2020b）
- [ ] Signal Processing Toolbox（`hamming`/`hanning`/`findpeaks`/`fdesign` 需要）
- [ ] Communications Toolbox（点云demo的 `awgn` 需要；没有就把 `generateSignal.m:61` 的 `awgn` 行注释掉，跑无噪声仿真）
- 验证：`FMCW_simulation.m` 直接跑通出图，环境就算就绪

---

## 3. 阶段 1：原理筑基（2-3 天）

**材料**：`Basic Mmwave Signal Processing/毫米波雷达感知_tutorial_张粟桐.pdf`（23页）+ `毫米波雷达传感器基础知识_TI.pdf`

**目标**：合上 PDF 能徒手推导以下内容，推导写进 `NOTES.md`：

- [ ] 中频信号模型：为什么 `s_IF = cos(2π(S·2d/c)·t + ...)`，对 ADC 维做 FFT 为什么直接得到距离谱（PDF 第 9-10 页）
- [ ] 三个分辨率公式及物理来源：
  - 距离 `dR = c/2B`（B 是有效带宽，不是扫频带宽——对照 `generateParameter.m:20-21` 里 `Bandwidth` 与 `BandwidthValid` 的区别）
  - 速度 `dV = λ/2NT`（N 为 chirp 数）
  - 角度分辨率由虚拟阵列孔径决定
- [ ] TDM-MIMO：3TX×4RX 如何时分复用拼出 12 通道等效虚拟阵（PDF 第 8 页）
- [ ] 最大不模糊速度 `Vmax = λ/4T` 怎么来的（`FMCW_simulation.m:22` 有现成算式）
- [ ] 背下 tutorial 第 15 页的 **7 种数据形式**：4D点云、RD热力图、距离-时间图、微多普勒（速度-时间）图、距离-方位角图、距离-俯仰角图、4D热力图——它们是全仓库的输出物词汇表

**自查**：能向别人解释「为什么距离维 FFT 用 `fft`、速度维要 `fftshift`、角度维也要 `fftshift`」？

---

## 4. 阶段 2：仿真主线（3-5 天）

**材料**：`Basic Mmwave Signal Processing/FMCW_simulation.m`（329行，仓库灵魂）+ `ca_cfar.m`

**方法**：逐段跑，每个 figure 对应链路一环。代码分段：

| 行号 | 段落 | 干什么 |
|---|---|---|
| 3-36 | 参数 | 77GHz/150MHz/1TX8RX，所有分辨率公式在此落地 |
| 38-96 | 目标 | 3个目标：50m@-15°@+10m/s、100m@10°@-15m/s、25m@30°@-15m/s |
| 98-165 | 信号 | 几何法：算双程时延→差拍相位→叠加3目标回波 |
| 187-203 | 2D FFT | RD 谱（ fft2 + 沿 doppler 维 fftshift ） |
| 205-219 | CA-CFAR | 2D 窗口式 CFAR + 检测点凝聚 |
| 221-243 | FFT测角 | 天线维 FFT → Range-Angle 图 + 单目标 DOA 谱 |
| 247-281 | MUSIC | 协方差矩阵特征分解 → 噪声子空间 → 伪谱 |
| 283-330 | 点云/RA-MUSIC | 谱峰→坐标；逐距离bin扫MUSIC |

- [ ] 跑通全流程，每个 figure 截图存到 `notes/` 并用一句话注释它验证了什么
- [ ] 精读 `ca_cfar.m`，在注释里推导门限因子 `α = N(Pfa^(-1/N) - 1)`（提示：Rayleigh/指数噪声模型下 CA-CFAR 的虚警概率公式）
- [ ] **改参数实验**（每条记一行结论到 NOTES.md）：
  - [ ] 带宽 150MHz→75MHz：两个同距离目标还能分开吗？验证 `dR=c/2B`
  - [ ] chirp 数 256→128：速度分辨率变化；把目标速度加到 `Vmax` 之外看速度模糊（折叠）
  - [ ] 两个目标放到同一距离不同角度：FFT 测角 vs MUSIC 的分辨率对比——理解超分辨的意义
- [ ] 搞懂坑#3（见第 7 节）：为什么 3 目标仿真里 MUSIC 取 1 个信号特征向量也能工作

**自查**：能画出从 `mixed{i,j}` 到点云的数据流转图（形状标注）吗？

---

## 5. 阶段 3：工程流水线（3-5 天）

**材料**：`基于点云的人体定位demo/` 全部 10 个文件，入口 `main.m`（112行）

**精读顺序**（按数据流）：

1. `generateParameter.m` — 76.5GHz/512chirps/3目标表；注意 `doaMethod` 开关
2. `generateSignal.m` — **正仿真**模型（直接写相位），对比阶段2的几何时延法：两种建模范式差异写进 NOTES
3. `RDfftMatrix.m` / `RAfftMatrix.m` — 加窗（为什么用 hanning）+ 两维 FFT 的维度方向
4. `chan_Accumulate.m` — 为什么是**非相干**积累（取模后相加）？什么时候必须相干？
5. `generateCfarParameter.m` + `cfar.m` — **两步分离式 CFAR**（先 doppler 维、后 range 维）vs 阶段2的 2D 窗口式：算力/性能取舍
6. `peakSearch.m` — 4邻域局部极大值，为什么 CFAR 后还要凝聚
7. `doa.m` — 三种测角：DBF(扫描加权) / FFT(补零1024点) / Capon(最小方差)，改 `doaMethod=1/2/3` 对比谱形与精度
8. `main.m` — 串联：注意 bin 索引→物理量的换算（坑#5）

- [ ] 每个文件头部加注释块：**本文件在链路中的位置 + 输入/输出张量形状**（如 `12×256×512`）
- [ ] 修坑#1（cfar.m 分支变量），把 SO/GO-CFAR 补全并对比 CA 的检测差异
- [ ] 修坑#5（off-by-one），画 bin→速度映射图验证
- [ ] **作者思考题**（main.m 第2行）：把 `DCA1000_Read_Data` 的输出接进本流水线替换 `generateSignal`——数据形状怎么对齐？（这题做完，实测数据随时能接入）
- [ ] 多帧运行（`Frame>1`），输出连续点云序列，观察 3 个目标的轨迹是否符合参数表中设置的运动

**自查**：随机遮住一个函数，能凭数据形状和链路位置重写它的大致逻辑吗？

---

## 6. 阶段 4：应用 demo + 衔接 MOT（按兴趣选做）

### 6a. 生命体征检测（推荐，微动思想独家）

**材料**：`生命体征检测demo/main.m`（282行）

链路：距离FFT → 找能量最大 range bin（人体所在）→ 取该 bin 相位时间序列 → 手动解缠 → 相位差分 → 两个 Chebyshev 带通（0.1-0.5Hz 呼吸 / 0.8-1.9Hz 心跳）→ FFT 谱峰搜频率

- [ ] 推导：微动 Δd 引起相位差 `Δφ = 4πΔd/λ`，77GHz 下 λ≈3.9mm——胸腔毫米级位移为何可测
- [ ] 搞懂 `chebyshev_IIR.m` 滤波器参数表（fdatool 生成，读通一边即可）
- [ ] 坑#4：手动解缠 vs MATLAB `unwrap()` 输出对比，注释差异
- [ ] 思考：`fs=20`（第145行）这个"相位信号采样率"由什么决定？（答案在 chirp 采样结构里）

### 6b. 热力图 demo / 手势识别（顺带）

- `基于热力图demo/demo.m` 与 Basic 的 `demo.m` 几乎相同，读一处即可；顺手看坑#2
- 手势识别只有外链：看 tutorial PDF 第 19 页思路（三张热力图作三通道输入CNN）即可，不必深入

### 6c. 衔接 MOT 主业（本仓库的最终意义）

- [ ] 把阶段 3 输出的点云 (R,V,θ) 转换为 (x,y)：`x=R·sinθ, y=R·cosθ`
- [ ] 多帧点云序列导出成 MOT 格式（frame, x, y, v），喂给 `D:/UGit/MOT` 里的 AB3DMOT / mmWave_radar_tracking 跑跟踪
- [ ] 打通后记录：雷达前端（本仓库）→ 跟踪器（主业）的全链路数据形状变化

---

## 7. 坑清单（读懂验证器）

学的时候逐个验证、修复、提交。能独立发现并解释这些坑 = 真懂了。

| # | 位置 | 问题 | 验证方式 | 状态 |
|---|---|---|---|---|
| 1 | `基于点云demo/cfar.m:60` | range维CFAR分支写成 `elseif dopplerMethod==2`，复制粘贴bug，SO/GO分支永远不可达 | 补全 rangeMethod 分支后对比检测点数 | [ ] 未修 |
| 2 | `Basic/demo.m:45`（热力图demo同） | 方位角FFT只取 `temp(1:8)`，但参数是3TX4RX=12通道 | 对照12通道 vs 8通道的Range-Azimuth图差异 | [ ] 未修 |
| 3 | `FMCW_simulation.m:266` | MUSIC 只取1个信号特征向量 `Qs=Q(:,1)`；3目标为何仍工作？ | 想清楚：3目标分处不同(R,V)bin，单bin内确为单源 | [ ] 未验证 |
| 4 | `生命体征demo/main.m:121-129` | 手动解缠 `+2π` 分支缺 `n=n+1`，且与 `unwrap()` 行为有差异 | 两法输出画在同一张图对比 | [ ] 未修 |
| 5 | `基于点云demo/main.m:98` | 速度解算 `(speedBin-dopplerBin/2-1)*speedRes` 疑似 off-by-one | 画 bin→速度映射直线验证 | [ ] 未修 |

---

## 8. 进度总表

| 阶段 | 内容 | 状态 | 完成日期 | 证据（commit） |
|---|---|---|---|---|
| 0 | Git 工作流就绪 | ✅ | 2026-11-15 | study 分支已建立 |
| 1 | 原理筑基（PDF+推导） | ⬜ | | |
| 2 | 仿真主线（FMCW_simulation） | ⬜ | | |
| 3 | 工程流水线（点云demo） | ⬜ | | |
| 4a | 生命体征 | ⬜ | | |
| 4c | 衔接 MOT | ⬜ | | |

---

*本指南由 ZCode 协助生成于 2026-11-15，基于对仓库全部 1685 行代码与教程 PDF 的通读。*
