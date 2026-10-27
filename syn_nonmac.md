# RMSNorm / RoPE / SwiGLU：仅用 DC 的简化评估

```text
请完成三个非 MAC 引擎的 DC-only 评估，仅获取面积 µm²、建模延迟 ns 和估计能耗 pJ。每个引擎在一次 dc_shell 会话内完成映射、DC 自带的基本 STA、面积与 Power Compiler 无波形功耗估计。不要启动 pt_shell、PrimePower 或 VCS，不做功能仿真，不寻找 DW 仿真模型，不生成 VCD/SAIF。

先读 README.md 和脚本，执行 python3 verify_upload.py。固定 nonmac_vector_eval 三个配置：
1. RMSNorm：OP=17、LANES=1024，运行长度 128/1024 共用一份硬件面积。
2. RoPE：OP=24、LANES=128，长度 128，cos/sin 是预给定输入。
3. SwiGLU：OP=23、LANES=1024，长度 1024，包含 gate/up 配对、preload 和完整 SiLU 门乘。
保持 RTL 规模，保留内部浮点乘加，不做 Attention QK/PV MAC 扣除。范围含引擎/包装/局部寄存器，不含外部 GSRAM/PHY、格式转换和 sin/cos 生成。

填 synopsys/site.json：SMIC SCC28NHKCP_HSC35P140_RVT_V0p2，RVT/SS/0.9V/125°C basic NLDM/NLPM，使用 DW 综合库/许可。确认 report_units、库面积单位及 Power Compiler 可用性。
保持 1ns 理想时钟；setup/hold uncertainty 50/20ps；clock/input slew 50ps；输入/输出 max/min delay 100/0ps；输出负载 5fF；max transition 200ps；仅 rst_n false path，无额外 FSM 假路径/multicycle，无 CTS/SPEF。

执行：
python3 synopsys/run_flow.py --stage all
python3 synopsys/collect_candidate.py results/smic28_dconly

默认共 3 次 DC、0 次 PT。每个引擎只做一次 compile -map_effort medium，DC 时序摘要完成后直接分析面积和功耗。保留汇总和各 10 条代表性 setup/hold 路径，完整链接、无黑盒及单位检查。保存网表/SDC，不做独立签核、百万行违例清单、多种子/多 N 扫描、空闲窗口、Fmax 搜索、重复闭合或全局 DDC 重载。
已有同源同参数同库同约束的完整网表优先复用，不重新 compile：
python3 synopsys/run_flow.py --stage analyze --output <已有结果根目录>
python3 synopsys/collect_candidate.py <已有结果根目录>
可加 --operation rmsnorm/rope/swiglu 单独执行。manifest 必须匹配，不可混用不同设计。

延迟沿用现有模型，本轮不重测周期：RMSNorm128/1024 各 10 拍、RoPE128 为 5 拍、SwiGLU1024 为 9 拍。按用户指定的 1ns/周期报告 10/5/9ns 建模延迟，即使 DC 时序失败也不按 WNS 或慢周期换算。DC 时序实际状态另列，不把模型值称为已闭合的硬件延迟。

功耗仅在 DC/Power Compiler 内用 set_switching_activity + report_power 获取。假设非时钟/复位输入每周期 0.1 次翻转、P(1)=0.5，clk 每周期 2 次翻转、rst_n 恒 1。按本版 DC help/man 核实 -period 语法和单位，可修正测量脚本并留 diff；不使用 PT 专属命令，不切换到 PT。
若 DC 功耗命令或许可不可用，保留面积/时序/建模延迟，功耗和能耗设 null/PENDING 并说明原因，不重复综合或虚构数值。
报告 internal/switching/dynamic/leakage/total W；动态和漏电单位逐项核对，收集器只把带明确单位的报告值统一成 W。解析失败时保留报告，人工核对后填值，不猜单位。
能耗按 E_pJ=P_W×模型时长ns×1000 换算，默认分别为 10/5/9ns。动态、漏电和总估计值分列，标 VECTORLESS_ASSUMED_ACTIVITY；没有命令/FSM/长度相关活动，不能称实际命令能耗。RMSNorm128/1024 共用一份通用引擎功耗且周期相同，粗估能耗会相同；面积只计一次。空闲动态功耗保持 null，不默认零，不升级 MEASURED。

交付简短 RESULTS_SUMMARY.md 和 reviewed JSON：四个长度 profile 的建模延迟及估计 pJ、三个引擎各一份面积/功耗、活动假设和 DC 时序实际状态。附关键面积/单位/链接/时序/功耗报告、源码/网表 SHA256 和脚本 diff。功能仿真标 SKIPPED_PER_USER_REQUEST，缺项 null/PENDING。不改 RTL、模拟器或 Excel，不自动导入校准，完成这些即交付。
```
