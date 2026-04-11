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

## 6. 一些常用终端命令

安装完成之后，除了直接运行 `claude`，还有几条比较常用的命令可以顺手记住。

### 初始化项目说明文件

如果你希望 Claude Code 在进入项目时自动读取项目约定、开发规范和说明文档，可以执行：

```bash
/init
```

根据 Anthropic 官方命令文档，`/init` 用来为当前项目初始化 `CLAUDE.md` 指南文件。这样后续在这个项目里使用 Claude Code 时，它会更容易理解你的项目背景和工作方式。

### 查看或管理 IDE 集成状态

如果你在 Windows 上同时使用 VS Code 之类的编辑器，可以执行：

```bash
/ide
```

这个命令用于管理 IDE 集成并查看当前状态，适合在命令行和编辑器联动使用时检查配置是否正常。

### 查看当前配置

如果你想看看 Claude Code 当前的设置，比如主题、模型或者一些偏好配置，可以执行：

```bash
/config
```

### 清空当前会话

如果当前上下文太长，或者你想重新开始一个新的会话，可以执行：

```bash
/clear
```

这个命令会清空当前对话历史，等价于重新开始当前终端会话。

### 检查安装状态

除了前面的版本检查，平时也很常用：

```bash
claude doctor
```

这个命令可以帮助你检查安装、配置和环境是否正常。

## 7. Git Bash 路径识别失败时的处理

如果你在 Windows 下使用的是原生环境，并且 Claude Code 没有正确找到 `Git Bash`，可以手动指定 `bash.exe` 路径：

```powershell
$env:CLAUDE_CODE_GIT_BASH_PATH="C:\Program Files\Git\bin\bash.exe"
```

设置后再重新运行：

```bash
claude
```

## 8. 核心命令汇总

```bash
node -v
npm -v
git --version
npm install -g @anthropic-ai/claude-code
claude --version
claude doctor
cd your-project-directory
claude
/init
/ide
/config
/clear
```
