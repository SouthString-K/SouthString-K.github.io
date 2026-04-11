# Claude Code Windows 安装教程

这篇只整理 Windows 版本的核心安装步骤，只保留 `Node.js`、`Git for Windows` 和 `Claude Code` 的安装方法与命令。

## 1. 环境要求

根据 Anthropic 官方文档，Windows 安装 Claude Code 需要：

- Windows 10 及以上
- Node.js 18+
- Git for Windows

官方文档：

- https://docs.anthropic.com/en/docs/claude-code/setup
- https://docs.anthropic.com/en/docs/claude-code/overview

## 2. 安装 Node.js

先安装 `Node.js`，建议直接安装官方 LTS 版本。

下载地址：

https://nodejs.org/

安装完成后，在 PowerShell 中检查版本：

```bash
node -v
npm -v
```

如果能正常输出版本号，就说明 Node.js 和 npm 已经安装成功。

## 3. 安装 Git for Windows

Claude Code 在 Windows 下推荐配合 `Git for Windows` 使用，尤其是 `Git Bash`。

下载地址：

https://git-scm.com/download/win

安装完成后检查：

```bash
git --version
```

如果命令可以输出版本号，就说明 Git 已经安装成功。

## 4. 安装 Claude Code

在 PowerShell 或 Git Bash 中执行下面的命令：

```bash
npm install -g @anthropic-ai/claude-code
```

安装完成后检查版本：

```bash
claude --version
```

再执行一次安装检查：

```bash
claude doctor
```

## 5. 启动 Claude Code

进入你的项目目录后运行：

```bash
cd your-project-directory
claude
```

## 6. Git Bash 路径识别失败时的处理

如果你在 Windows 下使用的是原生环境，并且 Claude Code 没有正确找到 `Git Bash`，可以手动指定 `bash.exe` 路径：

```powershell
$env:CLAUDE_CODE_GIT_BASH_PATH="C:\Program Files\Git\bin\bash.exe"
```

设置后再重新运行：

```bash
claude
```

## 7. 核心命令汇总

```bash
node -v
npm -v
git --version
npm install -g @anthropic-ai/claude-code
claude --version
claude doctor
cd your-project-directory
claude
```
