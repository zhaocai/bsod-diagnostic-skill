# bsod-diagnostic-skill

[English](README.md) | 简体中文

> **安装：** 直接把这句话发给你的 AI —— *"请安装技能
> https://github.com/YOUR_NAME/bsod-diagnostic-skill，装好后告诉我怎么用。"*

一个用于排查 Windows 蓝屏（BSOD）的 AI Agent 技能，采用微软官方标准调试流程：
事件查看器 → minidump → WinDbg/cdb → `!analyze -v` + 微软符号服务器，
并把调试结果转换成大白话结论和可执行的修复建议。

## 功能

- 从系统事件日志确认蓝屏（BugCheck 1001 / Kernel-Power 41）
- 定位对应的 minidump 转储文件（`C:\Windows\Minidump`），即使被系统清理
  删除了也有兜底方案
- 自动找到调试器（Windows SDK / 商店版 WinDbg），并处理 WindowsApps
  权限限制和扩展 DLL 缺失等常见坑
- 运行 `!analyze -v`（微软符号服务器），解读停止代码、失败桶、调用栈
- 对 0x9F 电源 IRP 蓝屏用 `!devobj` / `!irp` 挖出真正的责任驱动
  （而不只是失败桶里显示的模块名）
- 用大白话解释结果，列出历史蓝屏记录和分级修复建议

## 安装

### opencode

把 `bsod-diagnostic-skill` 文件夹复制到以下任一位置：

- 全局：`~/.config/opencode/skills/bsod-diagnostic-skill/`
- 项目：`.opencode/skills/bsod-diagnostic-skill/`

或者发布后通过 `opencode.json` 中的 URL 注册：

```json
{
  "skills": {
    "urls": ["https://example.com/skills/"]
  }
}
```

### Claude Code 及其他兼容 Anthropic SKILL.md 的 agent

```powershell
Copy-Item -Recurse .\bsod-diagnostic-skill "$env:USERPROFILE\.claude\skills\"
```

安装后重启 agent 即可生效。skill 就是一个文件夹里的 `SKILL.md`，任何兼容
SKILL.md 的 agent 都能加载。

## 使用方法

你只需要描述现象，例如"我刚才蓝屏了，帮我看看为什么"。skill 会引导 agent
完成：确认蓝屏时间 → 分析转储 → 给出大白话结论和修复方案。

## 运行要求

- Windows 10/11，PowerShell 5.1+（以管理员权限运行）
- 首次分析需联网下载微软符号（msdl.microsoft.com），耗时 1–5 分钟
- 建议提前安装调试器：Windows SDK "Debugging Tools for Windows" 或商店版
  WinDbg（skill 会自动寻找，也可自动从商店版 WinDbg 复制出可用的 cdb.exe）

## 说明与局限

- **数据不出本机。** 只读取本地事件日志和转储文件，不会上传到任何地方。
- Windows 内核三诊转储只包含寄存器和栈，部分深层内存分析（`!devobj`/`!irp`）
  可能不可用；skill 会如实说明，而不是瞎猜。
- 单次蓝屏通常是**驱动/软件问题，不是硬件坏了**。硬件级信号（如 0x124 WHEA）
  会被谨慎对待，绝不会凭一次事件就断定硬件损坏。

## 许可证

MIT，可自由使用、修改和分发，详见 [LICENSE](LICENSE)。

## 贡献

欢迎提交 PR。向 [`examples/`](examples/) 添加新案例时请附上停止代码、
`!analyze -v` 的失败桶 ID 和 `!irp` 栈信息。
