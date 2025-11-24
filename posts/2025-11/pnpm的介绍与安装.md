---
title: pnpm的介绍与安装
filename: 20251124-pnpm的介绍与安装
tags:
  - Javascript
  - Node.js
  - 包管理
  - pnpm
categories:
  - 前端技术
date: 2025-11-24
description: 介绍 npm 与 pnpm 的作用和差异，梳理 pnpm 的优化点，并给出在不同系统上的安装方法。
articleGPT: 本文对比 npm 与 pnpm 的工作方式和优势，解释 pnpm 如何通过内容寻址存储与严格 node_modules 结构带来性能和一致性提升，并给出 macOS、Windows、Linux 的安装步骤与常用命令。
top:
share: true
delete: false
---

## npm 与 pnpm 的作用
- 它们都是 Node.js 生态的包管理器，用来安装、更新、发布依赖，并维护 `package.json` 与锁文件。
- npm 是 Node 官方自带的默认工具；pnpm 是兼容 npm 语法的替代方案，主打更快安装、更少磁盘占用与更严格的依赖解析。

## pnpm 相比 npm 的主要优化
- **内容寻址的全局 store**：相同版本的包只在全局 store 下载一次，再通过硬链接/符号链接到项目，显著减少磁盘占用。
- **严格的 `node_modules` 结构**：pnpm 不把所有依赖平铺，避免“幽灵依赖”（未声明却能被 require）；依赖关系更符合规范。
- **安装速度更快**：并行下载 + 复用缓存；重复安装、CI 场景受益明显。
- **Workspaces/Monorepo 友好**：内置工作区支持，跨包依赖用软链接直连源代码，调试发布都更顺畅。
- **锁文件稳定**：`pnpm-lock.yaml` 记录精确来源，配合严格解析能减少“本地好、线上挂”的差异。

![pnpm 优势示意](/images/posts/2025-11/1.png)

## npm vs pnpm 快速对比
- 磁盘占用：pnpm < npm（多项目共享同一版本的缓存）。
- 安装速度：pnpm 通常更快，尤其是重复安装或大仓库。
- 依赖隔离：pnpm 更严格，能及早暴露未声明依赖。
- 语法兼容：常用命令基本一致（`install/add/remove/run` 等），切换成本低。

## 在不同系统上安装 pnpm
前提：已安装 Node.js（建议 LTS/≥16.13 以使用 corepack；多数项目推荐 Node 18/20）。

### 通用：Corepack（推荐）
```bash
corepack enable                  # 让 Node 自带的 corepack 管理包管理器
corepack prepare pnpm@latest --activate
pnpm -v
```

### macOS
- Homebrew：`brew install pnpm`  
- npm 全局：`npm install -g pnpm`

### Windows
- winget：`winget install pnpm.pnpm`
- Scoop：`scoop install pnpm`
- Chocolatey：`choco install pnpm`
- 已装 Node 时也可用 corepack 方法。

### Linux
- corepack：同上（适合已装官方 Node）。
- npm 全局：`npm install -g pnpm`
- 官方安装脚本：`curl -fsSL https://get.pnpm.io/install.sh | sh -`（需要网络与执行权限）。

## 常用命令速览
- 初始化：`pnpm init`
- 安装依赖：`pnpm install`
- 安装/删除单个包：`pnpm add react`，`pnpm add -D typescript`，`pnpm remove lodash`
- 运行脚本：`pnpm run build`，`pnpm test`
- 工作区（monorepo）：`pnpm install --filter 包名`，`pnpm run build --filter 包名`
- 清理缓存：`pnpm store prune`

## 迁移与使用提示
- 直接替换命令：将原先的 `npm install`、`npm run` 换为 `pnpm install`、`pnpm run`，保持 `package.json` 不变。
- 关注“幽灵依赖”报错：如发现依赖缺失，显式写入 `dependencies`/`devDependencies` 后再安装。
- CI 缓存：缓存 `~/.pnpm-store`（或 pnpm 指定的 store 路径）可进一步加速。
