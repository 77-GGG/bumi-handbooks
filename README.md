# Bumi 机器人中文文档

与 [Bumi SDK](https://github.com/Noetix-Robotics/noetix_sdk_bumi) 配套的三本中文 LaTeX 手册：一本讲如何把 Bumi 运动策略部署到 Ubuntu 笔记本，一本讲硬件认识、日常操作与 SDK 接口的整体使用，一本讲如何在虚拟机里装好 Ubuntu 并与 Bumi 直连通信。

| 手册 | 阅读版 | Markdown 全文 | 源码 | 篇幅 |
| --- | --- | --- | --- | --- |
| Bumi 机器人笔记本外挂部署手册 | [bumi-deploy-handbook.pdf](bumi-deploy-handbook.pdf) | [bumi-deploy-handbook.md](bumi-deploy-handbook.md) | [deploy-handbook/](deploy-handbook/) | 40 页 |
| Bumi 机器人综合使用手册 | [bumi-user-manual.pdf](bumi-user-manual.pdf) | [bumi-user-manual.md](bumi-user-manual.md) | [user-manual/](user-manual/) | 39 页 |
| Bumi 机器人虚拟机连接配置手册 | [bumi-vm-setup-handbook.pdf](bumi-vm-setup-handbook.pdf) | [bumi-vm-setup-handbook.md](bumi-vm-setup-handbook.md) | [vm-setup-handbook/](vm-setup-handbook/) | 19 页 |

三本手册都可以直接下载根目录下的 PDF 阅读；需要改内容时进入对应子目录重新编译。每本手册另有一份 Markdown 全文，内容与 PDF 一致，方便全文检索、在线预览和交给 AI 阅读。

## 怎么选

- 手上已经有 Bumi 机器人，只想把官方运动策略跑起来：先读 **部署手册**，从终端、Python、Conda 讲起，一路到 SDK 编译、网线连接、手柄配置和程序启动。
- 刚拿到机器人，或者准备做 SDK 开发：读 **综合使用手册**，覆盖部件与版本、开关机与 APP、网络、SDK 与 Docker 环境、HighController / LowController / MediaController、手柄与 ONNX 策略、音视频与相机采集，以及故障排查。
- 手上只有一台笔记本，想在虚拟机里准备 Ubuntu 环境连接机器人：读 **虚拟机连接配置手册**，从安装虚拟机（Windows x86\_64 用 VMware 与 amd64 镜像，MacOS（Apple Silicon）用 Parallels Desktop 及其自带的 arm 镜像）讲到桥接网络、静态地址、放行 DDS 端口，直到跑通示例程序。

前两本的 Conda 环境统一为 \`bumi_deploy\`，接口核对基线为 Bumi SDK 3.1.0。实机调试请按正文要求使用吊架，并完成停控与稳定支撑后再退出程序。

## 编译

三个子目录各自是独立可编译的 LaTeX 工程，需要带中文支持的 TeX Live（\`xelatex\`、\`latexmk\`；已在 TeX Live 2025 下验证）。

\`\`\`sh
cd deploy-handbook   && latexmk main.tex
cd ../user-manual    && latexmk main.tex
cd ../vm-setup-handbook && latexmk main.tex
\`\`\`

输出在各自子目录的 \`_build/main.pdf\`。更新根目录的阅读版：

\`\`\`sh
cp deploy-handbook/_build/main.pdf    bumi-deploy-handbook.pdf
cp user-manual/_build/main.pdf        bumi-user-manual.pdf
cp vm-setup-handbook/_build/main.pdf  bumi-vm-setup-handbook.pdf
\`\`\`

Windows PowerShell 对应命令：

\`\`\`powershell
Copy-Item deploy-handbook\_build\main.pdf    bumi-deploy-handbook.pdf
Copy-Item user-manual\_build\main.pdf        bumi-user-manual.pdf
Copy-Item vm-setup-handbook\_build\main.pdf  bumi-vm-setup-handbook.pdf
\`\`\`

## Markdown 全文

每本手册的 Markdown 全文由对应的 LaTeX 源码逐节转换而来，章节结构与 PDF 相同，附有可跳转的目录，与 PDF 一样放在仓库根目录：

- [bumi-deploy-handbook.md](bumi-deploy-handbook.md) —— 部署手册，9 章 + 3 个附录
- [bumi-user-manual.md](bumi-user-manual.md) —— 综合使用手册，10 章 + 3 个附录
- [bumi-vm-setup-handbook.md](bumi-vm-setup-handbook.md) —— 虚拟机连接配置手册，第零章 + 5 章 + 2 个附录
- [deploy-handbook/quickstart-walkthrough.md](deploy-handbook/quickstart-walkthrough.md) —— 部署手册成书前的分步速查（9 步流程 + FAQ）

正文、代码块和表格都按原样保留；PDF 中的示意图改用 Mermaid 或纯文本框图表达。改动 LaTeX 正文后，Markdown 需要同步更新。

## 目录

\`\`\`text
.
├── bumi-deploy-handbook.pdf        部署手册阅读版
├── bumi-user-manual.pdf            综合手册阅读版
├── bumi-vm-setup-handbook.pdf      虚拟机连接配置手册阅读版
├── bumi-deploy-handbook.md         部署手册 Markdown 全文
├── bumi-user-manual.md             综合手册 Markdown 全文
├── bumi-vm-setup-handbook.md       虚拟机连接配置手册 Markdown 全文
├── deploy-handbook/                部署手册 LaTeX 源码
├── user-manual/                    综合手册 LaTeX 源码
├── vm-setup-handbook/              虚拟机连接配置手册 LaTeX 源码
└── LICENSE                         MIT
\`\`\`

## 许可

三本手册自行编写的内容与排版均采用 **MIT**。保留的第三方材料按各自上游许可：Noetix SDK 示例沿用 BSD 3-Clause，官方部件图不适用 MIT，具体范围见各子目录的 \`THIRD_PARTY_NOTICES.md\`。

## 发布

每次改动编译出的 PDF 都会作为附件发到 [Releases](https://github.com/77-GGG/bumi-handbooks/releases)。
