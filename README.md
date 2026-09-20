# CALL-E Payment Result Guard

**让付款类电话结果在进入业务记录之前，先接受证据检查和状态资格判断。**

当 CALL-E 返回一段转写和结构化结论时，应用仍需要回答两个问题：**客户原话是否充分支持这个结论？有证据的结论，又最多能推进到哪一步？** 本项目围绕这两个问题，提供共享校验核心、kept / Recover 薄适配器，以及离线测试与电话样本回放。

交付版本 **v0.2**，定位为有限范围的社区参考实现。源码以[完整代码补丁](payment-result-guard.patch)提供；本仓库同时保存项目说明、版本过程和验证证据。默认演示无需 API key，不拨号、不支付。

## 为什么做这个项目

电话结束、字段完整、模型给出较高置信度，并不意味着可以直接更新业务状态。付款场景中，细小的语义差别会改变后续处理方式：

| 客户表达或提取结果 | 业务上需要区分的含义 | 本项目关注的检查 |
| --- | --- | --- |
| “审批通过后再付” | 附条件计划，而非无条件承诺 | 是否保留审批条件 |
| 金额字段写 1250，原话却否认可承诺日期 | 字段与证据存在冲突 | 结论是否得到原话支持 |
| 金额写成 `1k` | 不能沿用不明确的数字清洗结果 | 是否属于支持的金额表达 |
| “我已经付款” | 客户声明，尚需独立核实 | 不将电话声明升级为到账事实 |
| “可以重试扣款” | 客户意向，不代表扣款成功 | 将后续处理限定为建议 |

在 kept 的本地合成输入探测中，`1k` 曾被记录为 100 最小单位，附条件金额和矛盾原话也能进入承诺记录。加入校验后，这三类指定输入分别进入金额歧义、条件计划或证据冲突处理，不再直接构建 Promise。详细对照见[基线结果](baseline-probes.json)和[改动后结果](after-probes.json)。

这些结果证明特定消费层输入的行为变化，没有证明线上问题发生频率、真实损失或回款率改善。

## 它与 CALL-E 的关系

CALL-E 上游负责 SDK、认证、API、通话执行和 Provider 控制。本项目位于社区应用消费电话结果的环节：复用已有电话能力，检查结构化结果与原话的关系，并输出适用的业务状态。

```text
CALL-E 通话结果与逐轮转写
          ↓
宿主原有的鉴权、业务对象绑定和策略检查
          ↓
场景薄适配器 → 共用证据校验核心
          ↓
证据状态 + 业务状态 + 原因码 + 适用的原话引用
          ↓
宿主记录承诺、保存建议，或进入人工复核
```

身份与对象绑定由宿主负责，调度由宿主 Scheduler 负责，实际支付和到账由独立业务系统及授权流程确认。核心的所有输出均保持 `execution_authorized=false`。

## 两个应用如何复用

| 应用 | 业务场景 | 接入位置 | 结果使用边界 |
| --- | --- | --- | --- |
| **kept / Python** | 应收账单付款承诺跟踪 | 原有绑定和策略检查之后，构建 Promise 之前 | 限定表达且证据充分时可记录承诺；条件、矛盾或缺证据进入复核；不证明到账 |
| **Recover / TypeScript** | 订阅扣款异常后的客户意向跟进 | 鉴权及权威回查之后，保存建议之前 | 通过本地 Python 子进程复用核心；明确重试意向最多支持建议，不执行扣款 |

共用的是证据判断与状态契约，各应用保留自身数据、账本和策略。Recover 原有的 advisory、`recoveredCents=0` 等保护继续保留，不计为本项目新增功能。

直接价值是帮助开发者在两个不同技术栈的应用中复用检查，为指定异常输入提供可复现、可解释的处理方式。对运营人员而言，原因码能为后续复核提供线索；本版没有新增运营界面，也尚未测量人工处理效率的变化。更多用户分析与取舍见 [PROJECT.md](PROJECT.md)。

## 当前交付内容

- **共享核心**：接收业务上下文、来源范围、结构化结论和转写，分别输出证据状态、业务状态及原因。
- **两个薄适配器及最小宿主接入**：复用 kept 的 SDK 模拟与账本路径，以及 Recover 的实际路由和 Python 调用路径。
- **42 个基础合成案例**：付款承诺 26 个、恢复意向 16 个，覆盖正常表达、条件、矛盾和异常边界。
- **3 个脱敏脚本电话回放**：原样保留虚构台词与异常金额，分开报告通话完成、目标句取得和核心输出。
- **离线 harness 与代码补丁**：默认不触发真实电话或资金动作，支持本地复现与差异查看。

应用补丁后，核心位于 `apps/python/payment-result-guard/`；宿主改动位于 `apps/python/kept/` 和 `apps/typescript/recover/`。本仓库不是完整上游源码副本，需要先应用补丁再运行。

## 验证结果怎样理解

| 验证层 | 已有结果与证据 | 范围 |
| --- | --- | --- |
| 核心回归 | [63 项测试](core.log) | 有限规则、CLI 和回放检查 |
| kept 本地路径 | [195 项测试](kept.log) | 原 SDK、模拟 transport、capture 和临时 ledger |
| Recover 本地路径 | [49 项测试](recover.log) | 实际 POST 和 Python 子进程；供应商与数据库边界使用替身 |
| 合计 | **307 项测试**；[机器可读报告](validation-report.json) | 不等于 307 个真实电话 |
| 补丁应用 | [基线应用与代码树核对](patch-verification.json) | 指定上游版本的完整补丁 |
| 电话样本回放 | [三个脱敏样本的输出](scripted-replay.json) | 重放既有输入，不重新拨号或重新执行语音识别 |

此前实际双 AI 系统话术测试的覆盖状态为：

- **P01 明确承诺**：待补完整承诺句，不能使用调用方的示例代替接听方证据。
- **P02 条件承诺**：已取得 “If approved” 条件句；预设 100.00 与转写/提取的 100100 存在差异，待核查金额音频。
- **P03 否认承诺**：完整否认句已取得，平台提取为 `refused`。

三项通话都返回过任务完成标记，但样本取得情况不同。v0.2 因此将回归复现与样本覆盖分开呈现。核心对这些 `unclear/refused` 结果会提前转复核，不能据此宣称核心独立识别了条件或金额异常，也不能把脚本电话视为自然客户对话的整体准确率。逐轮过程见 [ITERATIONS.md](ITERATIONS.md)。

## 如何开始运行

需要 **Python 3.11+**；验证 Recover 还需要 **Node 22**。先下载本仓库的 [payment-result-guard.patch](payment-result-guard.patch)，在正常克隆的 `awesome-phone-call-agents` 干净工作副本中执行：

```bash
git switch --detach 707122340774e3d63d5ce87c643695afc50d53cd
python3 scripts/check_branch_name.py --branch feat/payment-result-guard
git switch -c feat/payment-result-guard
git apply --check /absolute/path/payment-result-guard.patch
git apply /absolute/path/payment-result-guard.patch
```

将示例中的补丁路径替换为实际下载路径。补丁只应用一次，不覆盖已有未提交工作。随后在应用补丁后的仓库根目录运行：

```bash
python3.11 -m venv ../guard-venv
source ../guard-venv/bin/activate
python -m pip install -e apps/python/payment-result-guard
python -m payment_result_guard --demo
python apps/python/payment-result-guard/scripts/replay.py
```

demo 展示合成重试意向如何保持建议状态；普通回放返回 `regression_passed=true`，表示观察结果可复现，同时保留 `targets_complete=false`，提示目标样本仍有待补事项。

可选的 `--require-targets` 覆盖检查模式返回退出码 1，用于提示 P01/P02 的补充验证事项。该输出与代码回归结果分别记录。安装依赖可能联网，默认演示及回放不要求真实凭据。完整宿主安装与验证命令见 [RUN.md](RUN.md)。

## 文档与证据导航

| 文件 | 建议用途 |
| --- | --- |
| [PROJECT.md](PROJECT.md) | 深入了解问题依据、目标用户、具体价值、实现和关键取舍 |
| [RUN.md](RUN.md) | 应用补丁、安装、演示及完整验证 |
| [ITERATIONS.md](ITERATIONS.md) | 查看从初版到脚本电话再到 v0.2 的过程与待补事项 |
| [CHECKLIST.md](CHECKLIST.md) | 核对交付内容及对应材料 |
| [payment-result-guard.patch](payment-result-guard.patch) | 查看完整源码差异并应用到上游基线 |
| [validation-report.json](validation-report.json) | 查看各组件执行结果及合成语料明细 |
| [SHA256SUMS](SHA256SUMS) | 核对公开材料的文件摘要 |

## 使用边界与后续方向

当前自动接受范围限定为部分 en-US、USD 数字金额和明确日期表达。口语数字、多语言、相对日期、复杂语义及拒绝分类仍有覆盖限制。Recover 的 Python 子进程依赖本地解释器，尚未验证 Edge/serverless 部署。真实宿主业务写入、支付执行和生产运行不在当前已验证范围内。

默认运行不产生电话或资金副作用，公开材料不含凭据、真实号码或原始录音。已有真实电话是在用户明确授权下逐项进行；停止查询或关闭页面不代表取消已提交电话。

下一步先补齐 P01 样本、核查 P02 金额来源，再以新增回归为前提评估语义扩展，并单独验证 Recover 的真实意向场景。当前是个人交付仓库，尚未向上游创建 PR；可在个人 Fork 应用完整补丁后继续社区协作。
