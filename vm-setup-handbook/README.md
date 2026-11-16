# Bumi 机器人虚拟机连接配置手册

面向没有现成 Ubuntu 笔记本的读者：先在虚拟机里装好 Ubuntu，再把虚拟机桥接到机器人所在的直连网段，配置静态地址并放通通信端口，最后跑通示例程序验证虚拟机与 Bumi 的连接。

**[阅读 PDF](../bumi-vm-setup-handbook.pdf)** · **[Markdown 全文](../bumi-vm-setup-handbook.md)** · [MIT 许可证](LICENSE) · [来源与第三方声明](THIRD_PARTY_NOTICES.md)

本手册是 [Bumi 机器人中文文档](../README.md) 合集之一，源码位于仓库的 `vm-setup-handbook/` 子目录。仓库根目录的 `../bumi-vm-setup-handbook.md` 是与 PDF 内容一致的 Markdown 全文，便于全文检索和 AI 阅读。原始素材 `虚拟机连接Bumi网络配置步骤.md` 保留在本目录，正文整理自这份记录。

操作环境分两种：Windows x86\_64 主机使用 **VMware** 与 **Ubuntu 22.04.5 amd64** 桌面版镜像，MacOS（Apple Silicon）主机使用 **Parallels Desktop** 及其自带的 arm 版镜像。两套环境的桥接与静态地址配置方法一致，只在安装步骤上分开说明。

## 内容

| 章节 | 主题 |
| --- | --- |
| 第零章 | 按主机架构选择 VMware / Parallels Desktop 与对应镜像，安装 Ubuntu |
| 第 1 章 | 目标地址与本机地址、直连条件下的整体配置流程 |
| 第 2—4 章 | 虚拟机桥接网络、netplan 静态地址、ufw 放行 DDS 端口 |
| 第 5 章 | ping 与示例程序验证通信 |
| 附录 | 常见故障排查表、恢复原配置 |

网络参数：本机 `192.168.55.101/24`，目标机器人 `192.168.55.102`。

## 编译 PDF

安装带有中文支持的 TeX Live，确保 `xelatex` 和 `latexmk` 可用。本项目已在 TeX Live 2025 下编译验证，中文使用 Fandol 字体，代码使用 Latin Modern Mono；流程图使用 TikZ 绘制。

进入本目录（`vm-setup-handbook/`）后运行：

```sh
latexmk -xelatex -interaction=nonstopmode -halt-on-error main.tex
```

输出为 `_build/main.pdf`，中间文件全部留在 `_build/`（`latexmkrc` 里设置了 `$pdf_mode = 5` 与 `$out_dir = '_build'`）。在 Overleaf 中也可直接以 `main.tex` 为主文件编译。

Windows PowerShell 下若没有 `latexmk`，可重复运行 `xelatex`：

```powershell
xelatex -interaction=nonstopmode -halt-on-error main.tex
xelatex -interaction=nonstopmode -halt-on-error main.tex
```

## 目录

```text
.
├── main.tex                   主文件、标题页与章节装配
├── preamble.tex               字体、页眉、提示框、代码与表格
├── frontmatter/               前言
├── chapters/                  第零章与 5 章正文
├── appendices/                2 个附录
├── docs/                      阅读版 PDF 存档
├── latexmkrc                  编译设置
├── licenses/                  第三方许可证原文
├── THIRD_PARTY_NOTICES.md      来源与第三方材料范围
├── 虚拟机连接Bumi网络配置步骤.md  原始配置记录
└── LICENSE                    MIT
```

修改正文时编辑对应章节；修改页眉、字体和提示框时编辑 `preamble.tex`。所有 TeX 文件通过本目录内的相对路径引用，工程可独立编译。编译好的阅读版放在仓库根目录的 `../bumi-vm-setup-handbook.pdf`。

## 来源与整理范围

- 项目内的原始配置记录 `虚拟机连接Bumi网络配置步骤.md`
- [Bumi SDK](https://github.com/Noetix-Robotics/noetix_sdk_bumi)：示例程序 `example_policy_keyboard` 与 `[DEBUG]: init finish` 输出的来源

正文按操作顺序重新组织：把原记录里的十个步骤拆成桥接网络、静态地址、放行端口、验证通信四章，补充了 Windows x86\_64 与 MacOS（Apple Silicon）两种主机的安装说明，故障表整理为附录。配置参数、命令与输出均保持原样。

## 许可与维护

本项目自行编写的内容与排版采用 **MIT**。SDK 相关示例保留上游 **BSD 3-Clause** 条件，范围与完整声明见 [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md)。PDF 编译记录不作为实机测试记录。

## 修改文档

正文在对应章节文件中修改，公共版式在 `preamble.tex` 中修改。提交修改前重新编译并检查 PDF，把 `_build/main.pdf` 覆盖到仓库根目录的 `../bumi-vm-setup-handbook.pdf`；编译缓存保留在 `_build/`。
