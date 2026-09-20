# 运行与演示

本仓库以完整代码 Patch 交付，无需压缩包。先将补丁应用到指定上游基线，再安装并运行。

## 应用源码补丁

在正常克隆的 awesome-phone-call-agents 干净工作副本中，下载本仓库的 payment-result-guard.patch，然后执行：

```bash
git switch --detach 707122340774e3d63d5ce87c643695afc50d53cd
python3 scripts/check_branch_name.py --branch feat/payment-result-guard
git switch -c feat/payment-result-guard
git apply --check /absolute/path/payment-result-guard.patch
git apply /absolute/path/payment-result-guard.patch
```

请勿在已有未提交改动上直接操作；补丁仅应用一次。应用后可查看核心、测试、两个宿主适配器和文档。完整补丁已在独立 Git 索引完成基线应用与代码树核对。

## 快速离线运行

需要 Python 3.11+，在应用补丁后的仓库根目录执行：

```bash
python3.11 -m venv ../guard-venv
source ../guard-venv/bin/activate
python -m pip install -e apps/python/payment-result-guard
python -m payment_result_guard --demo
python apps/python/payment-result-guard/scripts/replay.py
python apps/python/payment-result-guard/scripts/replay.py --require-targets
```

安装依赖可能联网；默认演示和回放无需 API key，不拨号、不支付。普通回放返回 0 表示观察结果可复现；覆盖检查返回 1、targets_complete=false，提示 P01 待补完整承诺句、P02 待核查金额差异。两者与业务状态分别记录。

## 两个宿主与完整验证

安装 Node 22，随后在同一仓库和 Python 环境执行：

```bash
python -m pip install -e 'apps/python/kept[dev]'
npm --prefix apps/typescript/recover ci
python apps/python/payment-result-guard/scripts/verify.py --output ../guard-report
python scripts/validate_repository.py
```

当前本地 307 项测试已有执行日志：[核心](core.log)、[kept](kept.log)、[Recover](recover.log)、[完整报告](validation-report.json)。三个电话样本的离线回放见 [scripted-replay.json](scripted-replay.json)。

宿主检查使用本地模拟边界，未连接真实支付或数据库服务。最近一轮未重跑全库 lint/build；旧版记录过 Recover 原有 lint 问题，不宣称全仓库所有检查无误。

## 社区贡献方式

这是个人交付仓库，目前没有上游 PR。完整补丁可用于个人 Fork 的功能分支，后续可创建 Draft PR；不要推送本地快照导入的历史。项目说明与迭代过程均可在本仓库直接阅读。
