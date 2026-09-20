# CALL-E Payment Result Guard

付款类电话结果的证据校验与业务状态边界。交付版本 v0.2。

## 查看材料

- [项目说明](PROJECT.md)：产品理解、问题、用户、方案、取舍及 AI 使用。
- [版本迭代与测试过程](ITERATIONS.md)：实际观察、调整依据和待补验证事项。
- [运行与演示](RUN.md)：补丁应用、运行路径与复现步骤。
- [交付内容核对](CHECKLIST.md)：提交材料对应关系。
- [验证报告](validation-report.json)及 [核心](core.log)、[kept](kept.log)、[Recover](recover.log)测试日志。
- [可直接查看的代码补丁](payment-result-guard.patch)。

本仓库是个人交付仓库；完整补丁面向 awesome-phone-call-agents 的基线
`707122340774e3d63d5ce87c643695afc50d53cd`，尚未向上游发起 PR。
源码通过完整补丁提供，应用到指定上游基线后即可查看并运行。

## 当前验证范围

本地 307 项测试已有执行日志；三个脱敏电话样本可离线回放。
P03 已取得完整否认句；P01 待补完整承诺句，P02 待核查金额差异。
模拟、本地适配回放、实际双 AI 脚本电话和真实宿主集成分别记录。
默认演示无需 API key，不拨号、不执行支付。未宣称真实业务收益或生产集成已验证。

核心、回放源码以及 kept / Recover 改动均包含在完整补丁中。请从 RUN.md 开始运行。
