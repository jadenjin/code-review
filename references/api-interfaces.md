# 接口与步骤说明

本文档给出一个通用的 Gitea接口调用链路

## 总体链路

### 步骤 1：获取审查对象上下文

目的：拿到 PR 详情、变更文件、patch、compare 信息。

按事件类型选择接口：

#### 1.1 Pull Request 详情

- 方法：`GET`
- 接口：`{gitea.api_base_url}/repos/{owner}/{repo}/pulls/{pr_number}`
- 用途：获取 PR 基础信息、分支、状态

#### 1.2 Pull Request 文件变更

- 方法：`GET`
- 接口：`{gitea.api_base_url}/repos/{owner}/{repo}/pulls/{pr_number}/files`
- 用途：获取变更文件列表、patch、增删行

#### 1.3 单次提交详情

- 方法：`GET`
- 接口：`{gitea.api_base_url}/repos/{owner}/{repo}/git/commits/{commit_sha}`
- 用途：获取指定 commit 的变更明细

#### 1.4 Compare 差异

- 方法：`GET`
- 接口：`{gitea.api_base_url}/repos/{owner}/{repo}/compare/{base_ref}...{head_ref}`
- 用途：用于 push 或多提交汇总场景，拉取净变化

鉴权方式：

- Header：`Authorization: token {gitea.token}`

### 步骤 2：下载补充内容

目的：当 files 接口内容不足时，拉取原始文件或仓库内容补充上下文。

可选接口：

#### 2.1 获取文件内容

- 方法：`GET`
- 接口：`{gitea.api_base_url}/repos/{owner}/{repo}/contents/{file_path}?ref={head_ref_or_sha}`
- 用途：读取变更文件全文

#### 2.2 clone 仓库

- 方式：`git clone` 或 `git fetch`
- 地址：`{REPO_CLONE_URL}`
- 用途：在需要跨文件上下文、配置文件、依赖关系时本地分析

### 步骤 3：识别语言并加载对应审查方案

目的：针对不同文件套用不同规则。

处理规则：

- `.java` → 读取 `java-springboot-review.md`
- `.vue`, `.ts`, `.js`（位于 Vue 前端目录时）→ 读取 `vue-review.md`
- `.cpp`, `.cc`, `.cxx`, `.h`, `.hpp`（位于 Qt 工程时）→ 读取 `cpp-qt-review.md`

### 步骤 4：执行代码审查

目的：根据 patch 与上下文生成行级问题和总结。

输入：

- patch / diff
- 文件全文（可选）
- 审查变量模板中的运行参数
- 语言审查方案

输出：

- 审核结论：`approve` / `comment` / `request changes`
- 行级问题列表
- 最终总结
- 变量检查结果

### 步骤 5：回写 OpenClaw 审查结果

目的：把格式化后的结果提交到 OpenClaw 工作流。

### 步骤 6：回写 Gitea 评论 / Review

#### 6.1 PR 行级评论

- 方法：`POST`
- 接口：`{gitea.api_base_url}/repos/{owner}/{repo}/pulls/{pr_number}/reviews`
- 作用：创建带行级评论的审查，可精确标注到具体代码行

**重要：Gitea 1.25+ 使用 `new_position` 和 `old_position`，不是 `line`！**

完整请求体示例：

```json
{
  "commit_id": "c2fd9e173727b1a939a7ec8302d7f385f2e03a75",
  "event": "REQUEST_CHANGES",
  "body": "发现阻断问题，请修复后再次提交。",
  "comments": [
    {
      "path": "src/main/java/com/example/Service.java",
      "body": "[警告]\n位置：src/views/Home.vue : Line 12\n内容:直接操作了 DOM 节点，违反了 Vue 的响应式数据驱动原则。\n建议:\n// 改为使用 ref 绑定\nconst myElement = ref(null);",
      "new_position": 83
    },
    {
      "path": "src/main/java/com/example/Controller.java",
      "body": "[major] 路径遍历风险",
      "new_position": 47
    }
  ]
}
```

**参数说明：**
- `commit_id`: PR 的 head commit SHA（必需）
- `event`: `APPROVE` / `REQUEST_CHANGES` / `COMMENT` / `PENDING`
- `body`: Review 总体评论
- `comments`: 行级评论数组
  - `path`: 文件路径（必需）
  - `body`: 评论内容（必需）
  - `new_position`: 新文件中的行号（新增或修改的行）
  - `old_position`: 旧文件中的行号（删除的行，可选）
  - **注意：`new_position` 和 `old_position` 至少一个非 0**

**常见错误：**
- ❌ 使用 `line` 参数 → Gitea 会忽略，评论变成文件级别
- ❌ `new_position` 设为 0 → 评论无行号锚点
- ❌ 缺少 `commit_id` → API 拒绝请求

#### 6.2 PR Review 普通评论

- 方法：`POST`
- 接口：`{gitea.api_base_url}/repos/{owner}/{repo}/issues/{pr_number}/comments`
- 作用：发布总体中文总结

示例请求体：

```json
{
  "body": "[拒绝合并]\n审查摘要：发现 1 处严重事务逻辑缺失（Java）和 1 处内存泄露隐患（C++）\n合并建议：修复行级评论中标记为“错误”的代码后方可合并。"
}
```

## 接口使用原则

- 优先使用最小权限 token
- 优先读取 patch，只有 patch 不足时再拉文件全文或 clone
- 不要在日志、报告、评论中输出真实 token
