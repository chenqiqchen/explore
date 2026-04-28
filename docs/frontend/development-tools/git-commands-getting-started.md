---
title: 【git】指令场景实战：单分支与多分支协作流程
date: 2026-04-28
tags: [git, 开发工具, 版本控制, 团队协作]
category: 开发工具
description: 面向新手的 Git 实战入门指南，重点讲透单分支与多分支两种公司常见协作场景，从拉取代码到提交合并完整走一遍。
---

# 【git】指令场景实战：单分支与多分支协作流程

## 一、先理解 Git 在做什么

一句话版本：**Git 是一个版本管理工具，用来记录代码的每次变化。**


## 二、第一次使用 Git，要先做的两件事

### 1）配置用户名和邮箱

```bash
git config --global user.name "你的名字"
git config --global user.email "你的邮箱"
```

它们会写进提交记录里，方便团队追踪修改来源。

### 2）查看配置是否生效

```bash
git config --list
```

## 三、两种开始方式： `clone` 和 `init`

### 方式 A：拉取远程已有项目

```bash
git clone <仓库地址>
```

例如：

```bash
git clone https://github.com/your-org/your-repo.git
```

### 方式 B：本地新项目

```bash
git init
```

在当前目录初始化一个 Git 仓库。

## 四、场景一：单分支协作（直接在团队分支上开发）

这个场景常见于小团队或内部项目：大家都在同一个长期分支上协作（例如 `dev`）。

### 目标

拿到仓库，拉最新代码，完成一个小需求并提交。

### 推荐流程

#### 第 1 步：先查看当前有哪些分支

```bash
git branch -a
```

#### 第 2 步：切换到团队分支并拉取最新代码，避免“在旧代码上开发”

```bash
git switch dev
git pull origin dev
```

#### 第 3 步：暂存并提交

```bash
git add .
git commit -m "feat: 完成登录页按钮交互优化"
```

#### 第 4 步：推送到远程团队分支

```bash
git push origin dev
```

### 单分支场景的关键提醒

- 养成“先 `pull` 再开发”的习惯
- 不要长时间不拉最新代码，冲突可能会越来越难解

## 五、场景二：多分支协作（主流团队工作流）

这个场景更常见于规范化团队：`main/master` 保持稳定，每个需求在独立分支完成，再合并回主干。

### 目标

从主干拿最新代码，创建自己的功能分支，完成开发后合并回目标分支。

### 推荐流程

#### 第 1 步：先更新主干代码

```bash
git switch main
git pull origin main
```

#### 第 2 步：创建并切换到功能分支

```bash
git switch -c feature/login-form
```

分支名建议包含业务语义，如 `feature/`、`fix/` 前缀。

#### 第 3 步：暂存并提交

```bash
git add .
git commit -m "feat: 新增登录表单校验逻辑"
```

一个完整功能可以拆成多个小提交，方便 review 和回滚。

#### 第 4 步：推送分支并建立远程跟踪

```bash
git push -u origin feature/login-form
```

`-u` 只需第一次使用，后续直接 `git push` 即可。

#### 第 5 步：合并代码（两种常见方式）

方式 A（推荐）：在代码平台发起 PR / MR，由评审后合并。  
方式 B（本地演示）：切回目标分支后手动合并：

```bash
git switch main
git pull origin main
git merge feature/login-form
git push origin main
```

#### 第 6 步：合并完成后清理分支

```bash
git branch -d feature/login-form
git push origin --delete feature/login-form
```

本地和远程都清理掉已合并分支，分支列表会更清爽。

## 六、两个场景都通用的常见问题

### 1）`pull` 或 `merge` 冲突了

说明同一代码区域被不同提交修改。处理方法：手动解决冲突文件后，再 `add` 和 `commit`。

### 2）常见撤销操作（建议记住这 3 条）

```bash
# 取消暂存
git restore --staged <文件>

# 丢弃工作区改动（单个文件）
git restore <文件>

# 丢弃当前目录改动
git restore .
```

### 3）当前改动没做完，但要临时切走处理其他事情

```bash
git stash
git switch 其他分支
# 处理完回来
git switch 原分支
git stash pop
```

## 七、总结：先练熟“场景流程”，再背命令

最重要的不是记住多少个命令，而是跑通完整流程。

- 单分支：`branch -a -> switch/pull -> 开发 -> add/commit -> push`
- 多分支：`main pull -> 新建分支 -> 开发提交 -> push 分支 -> 合并 -> 清理分支`

当这两套流程熟练后，你再学习 rebase、cherry-pick等会轻松很多。
