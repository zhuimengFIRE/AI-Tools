# Cursor 分享

## 1. 什么是Cursor 
Cursor 是由 4 位麻省理工学院毕业生在 2022 年创立的一款 AI 编程 IDE。本质上，它是在 VSCode 的基础上深度集成 AI 能力的智能编程环境。

---

## 2. Cursor 可以做什么
那么，它和普通 AI（如 ChatGPT、DeepSeek）有什么区别？

- 能够理解整个项目上下文，而不仅仅是单段代码
- 可以自动创建文件和项目结构
- 按照既有代码风格生成代码
- 支持代码审查（Code Review）
- 支持自动 Debug 和问题修复

 简单来说：**它不仅是“会写代码”，而是“会参与开发”**

---

## 3. Cursor 的模式

### 3.1 Agent 模式（最常用）
- 输入需求后，自动完成代码编写、修改等任务
- 适合快速开发功能
- 几乎可以当作“AI 程序员”

---

### 3.2 Plan 模式
- 用于制定和管理开发计划
- 提供交互式编辑器，可以直接修改计划
- 确认后，Cursor 会按照计划执行代码修改

👉 适合：复杂需求、多步骤开发

---

### 3.3 Debug 模式
- 基于运行时信息 + 人工验证的调试机制
- 能稳定修复复杂 Bug
- 对于传统 AI 难以解决的问题效果更好

👉 适合：疑难 Bug、复杂问题定位

---

### 3.4 Ask 模式
- 类似普通 AI 问答模式
- 用于知识查询、问题咨询

👉 适合：日常提问、学习

---

## 4. Cursor 进阶

### 4.1 设置项目规则（非常重要 ⚠️）

项目规则会直接影响 AI 生成代码的质量。

我做过对比：
- ❌ 没有规则 → 代码混乱、不可控
- ✅ 有规则 → 输出稳定、风格统一

👉 一个好的项目规则应包含：

#### （1）项目结构规范
- 文件夹命名规则
- 模块划分（如：Modules / Services / Utils）
- 分层结构（MVC / MVVM / Clean Architecture）

#### （2）代码规范
- 命名规范（变量 / 方法 / 类）
- 注释规范
- 代码风格（缩进、换行等）
- 是否使用某些框架（如 SnapKit、Moya）

#### （3）文档规范
- 接口文档格式
- README 编写规则
- 注释说明要求

#### （4）测试规范
- 是否必须写单元测试
- 测试覆盖率要求
- 测试文件命名规则

---

### ✅ 示例：项目规则（示例模板）

```txt
You are working on an iOS project using Swift.

Project Structure:
- Follow MVVM architecture
- Modules are organized under /Modules
- Shared components under /Common

Code Style:
- Use camelCase for variables and functions
- Use PascalCase for class names
- Prefer SwiftUI, fallback to UIKit if necessary
- Use SnapKit for layout

Networking:
- Use Moya + SwiftyJSON for API layer
- All network requests must be encapsulated

Documentation:
- All public methods must include comments
- Use MARK to separate sections

Testing:
- Write unit tests for ViewModel
- Test files must end with Tests.swift