# 运行、查看与提交方式

## 快速离线演示

需要 Python 3.11+。在本目录创建独立环境并安装本地核心：

```bash
python3.11 -m venv .venv
source .venv/bin/activate
python -m pip install ./代码与补丁/payment-result-guard
python -m payment_result_guard --demo
python 代码与补丁/payment-result-guard/scripts/replay.py
python 代码与补丁/payment-result-guard/scripts/replay.py --require-targets
```

安装依赖可能联网；演示和回放不需要 API key，不会拨号或支付。普通回放应退出 0、regression_passed=true；样本覆盖检查模式返回 1、targets_complete=false，提示 P01 待补完整承诺句、P02 待核查金额差异；该状态与回归结果分开记录。核心 demo 的合成重试意向始终只产生建议，不授权执行。

## 查看及应用完整补丁

主补丁为 `代码与补丁/payment-result-guard.patch`。可直接用文本编辑器查看；基线为上游 `707122340774e3d63d5ce87c643695afc50d53cd`。在正常克隆的 awesome-phone-call-agents 中：

```bash
git switch --detach 707122340774e3d63d5ce87c643695afc50d53cd
python3 scripts/check_branch_name.py --branch feat/payment-result-guard
git switch -c feat/payment-result-guard
git apply --check /absolute/path/payment-result-guard.patch
git apply /absolute/path/payment-result-guard.patch
```

请使用干净工作副本；以上命令不是覆盖已有未提交工作的指令。完整补丁只应用一次，不需要再叠加旧版本或增量补丁。已在独立 Git 索引完成基线应用与代码树核对，记录见 `代码与补丁/patch-verification.json`。

## 两个宿主与完整验证

应用补丁后，按 apps/python/payment-result-guard/README.md 安装 Python 3.11+、kept 开发依赖与 Node 22 / Recover 依赖，然后运行：

```bash
python apps/python/payment-result-guard/scripts/verify.py --output ../guard-report
python scripts/validate_repository.py
```

当前结果为 307 项测试通过，见 `验证证据/harness/`。宿主检查使用本地模拟边界，未连接真实支付或数据库服务。本轮未重跑全库 lint/build；旧版曾记录 Recover 原有 lint 问题，不宣称全仓库所有检查无误。

## 如后续采用 GitHub 呈现

当前选择 Patch 形式交付，尚无 PR 链接。若需要 GitHub，可在个人 Fork 的正常上游克隆中应用补丁、提交并推送功能分支，再创建 Draft PR，正文使用 `代码与补丁/PR-description.md`。不要推送本地快照导入的原始提交历史。

对方可在 PR 的 Files changed 查看代码差异，在说明中了解运行方式；也可直接下载此包查看补丁并运行。无需先被社区接受或合并。本次仅整理材料，未执行上传或发送。
