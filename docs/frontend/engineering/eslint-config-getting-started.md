---
title: ESLint 配置文件入门指南：从 0 到 1 搭建可维护的前端规范
date: 2026-04-17
tags: [eslint, 工程化, 前端规范, 代码质量]
category: 工程化
description: 一篇讲透 ESLint 配置文件的入门指南，覆盖新版 Flat Config、规则分层、与 Prettier 协作及团队落地实践。
---

# ESLint 配置入门指南

## 前言

刚接触 ESLint 时，很多同学都会有同一个感受：  
**规则太多、配置太杂、报错看不懂。**


---

## 一、先搞懂：ESLint 配置文件到底在做什么？

一句话解释：

官方版：查找并修复JavaScript代码中的问题

简单版：解决代码质量的问题

ESLint 的核心作用把团队约定写进工具里，自动帮你做这几件事：

1. **统一风格**：比如分号、引号、import 顺序
2. **兜底错误**：比如未使用变量、重复定义、潜在 bug
3. **约束坏味道**：比如滥用 `any`、随意 `console`

---

## 二、ESLint 配置文件有哪些形式？

目前你会遇到两种主流写法：

### 1）传统写法（Legacy）

- `.eslintrc.js`
- `.eslintrc.json`
- `.eslintrc.yml`

### 2）新版写法（Flat Config，推荐）

- `eslint.config.js`（或 `eslint.config.mjs`）

从 ESLint 9 开始，官方主推 Flat Config。  
如果你是新项目，建议直接用 `eslint.config.js`，后续迁移成本更低。

---

## 三、从 0 到 1：最小可用配置（Flat Config）

### 第一步：使用官方命令初始化

```bash
npm init @eslint/config@latest
```

执行后会进入交互式问答，按项目实际情况选择即可。  
如果你是 React + TypeScript 项目，建议优先选择：

- 使用 `JavaScript modules (import/export)`
- 启用 TypeScript
- 按需开启 React 相关能力
- 最终生成 `eslint.config.js`（Flat Config）

### 第二步：在生成配置上做增量调整

```js
import js from "@eslint/js";
import globals from "globals";
import reactPlugin from "eslint-plugin-react";
import reactHooksPlugin from "eslint-plugin-react-hooks";
import tseslint from "typescript-eslint";

export default [
  {
    ignores: ["dist/**", "node_modules/**", "coverage/**"],
  },
  js.configs.recommended,
  ...tseslint.configs.recommended,
  {
    files: ["**/*.{js,mjs,cjs,ts,tsx,jsx}"],
    languageOptions: {
      ecmaVersion: "latest",
      sourceType: "module",
      globals: {
        ...globals.browser,
        ...globals.node,
      },
    },
    plugins: {
      react: reactPlugin,
      "react-hooks": reactHooksPlugin,
    },
    rules: {
      "no-console": ["warn", { allow: ["warn", "error"] }],
      "@typescript-eslint/no-unused-vars": [
        "warn",
        { argsIgnorePattern: "^_", varsIgnorePattern: "^_" },
      ],
      "react/react-in-jsx-scope": "off",
      "react-hooks/rules-of-hooks": "error",
      "react-hooks/exhaustive-deps": "warn",
    },
    settings: {
      react: {
        version: "detect",
      },
    },
  },
];
```

> 这份配置的思路是：先通过 `npm init @eslint/config@latest` 快速生成可用配置，  
> 再在官方推荐规则之上按项目情况“增量覆盖”。

## 四、重点关注 `rules` 对象

在真实项目里，你最常改的就是 `rules`。  
`rules` 的结构很简单：

- 对象 **key**：规则名（比如 `no-unused-vars`）
- 对象 **value**：规则的限制级别

常见级别如下：

- `"off"` 或 `0`：关闭此规则
- `"warn"` 或 `1`：警告，不影响代码运行
- `"error"` 或 `2`：错误，通常会在校验或构建阶段拦截

示例：

```js
rules: {
  "no-unused-vars": 2, // 禁止定义未使用的变量
  "no-var": 2, // 不允许使用 var 定义变量
}
```

### 从编辑器报错反推规则名

实际开发时，多留意代码下方的黄色、红色波浪线。  
把鼠标移到报错代码上，通常会看到一个提示框，里面会带上 ESLint 规则名（例如 `no-undef`、`@typescript-eslint/no-explicit-any`）。

拿到规则名后，你就可以在 `rules` 对象里精确配置它：

- 想先放行：改成 `"off"` 或 `"warn"`
- 想严格约束：改成 `"error"`

更多规则说明可参考 ESLint 中文文档：<https://eslint.nodejs.cn/>

---

## 五、把 ESLint 真正落到团队流程

如果你只在本地偶尔跑一次 lint，规范很快会失效。  
建议按下面这套“三级兜底”来落地：

- 本地手动可执行（开发者随时自检）
- 提交前自动拦截（问题不进仓库）
- CI 再兜底校验（保证分支质量）

### 1）在 `package.json` 增加脚本

```json
{
  "scripts": {
    "lint": "eslint .",
    "lint:fix": "eslint . --fix"
  }
}
```

日常开发建议先跑：

```bash
npm run lint
```

如果是可自动修复的问题，再执行：

```bash
npm run lint:fix
```

### 2）在提交前自动检查

可配合 `lint-staged + husky`，只检查本次提交文件，速度和体验都更好。  
这样可以把问题挡在 `git commit` 之前，避免把明显错误提交到远端。

可参考这类配置思路：

```json
{
  "lint-staged": {
    "*.{js,jsx,ts,tsx}": ["eslint --fix"]
  }
}
```

### 3）在 CI 中执行 `npm run lint`

在 GitHub Actions / GitLab CI 等流水线里加一条 lint 步骤，作为最终兜底。  
这样即使有人跳过了本地检查，CI 也能统一拦截。

这样做的价值是：把“靠记忆遵守规范”变成“工具自动兜底”。

---

## 六、和 Prettier 的关系：分工，不冲突

**两者职责不同。**

- **Prettier**：只管格式（排版、美化）
- **ESLint**：管质量（潜在问题、代码约束）

最佳实践是“格式交给 Prettier，质量交给 ESLint”，避免重复劳动。
