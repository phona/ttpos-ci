# ttpos-ci

TTPOS 统一构建仓库。接收业务仓库的 dispatch 事件，执行 CI/CD 流水线。

## 架构

```
业务仓库 (dispatch.yml)          构建仓库 (ci-go.yml / ci-flutter.yml)
┌─────────────────────┐          ┌──────────────────────────────────┐
│  PR / Push 事件      │          │  init: 读取 ci-env, 算 merge-base │
│  ↓                  │ dispatch │  ↓                               │
│  repository_dispatch ├─────────→  lint / unit-test / integration   │
│  (event_type + payload)        │  ↓                               │
└─────────────────────┘          │  sonarqube                       │
                                 └──────────────────────────────────┘
```

**职责划分**：
- **业务仓库**：定义*怎么跑*（`make ci-*` targets）
- **构建仓库**：定义*怎么调度*（workflow 编排、环境准备、状态回报）

## 业务仓库 CI 契约

业务仓库通过 Makefile 暴露以下标准接口，构建仓库统一调用。

### 必须实现的 targets

| Target | 说明 | 环境变量 |
|--------|------|----------|
| `ci-env` | 输出 CI 配置（`KEY=VALUE` 格式到 stdout） | - |
| `ci-setup` | 安装依赖、准备环境 | - |
| `ci-lint` | 代码检查 | `BASE_REV` — merge-base SHA，增量扫描基准；未设置则全量 |
| `ci-unit-test` | 单元测试，覆盖率输出到 `coverage/` | - |
| `ci-integration-test` | 集成测试，覆盖率输出到 `coverage/` | `BUILD_ID` — 隔离并发运行 |
| `ci-build` | 构建产物 | `VERSION` — 最终镜像 tag；`REF_NAME` — 分支/tag 上下文；`SHA` — 构建提交 |

### ci-env 输出规范

```makefile
ci-env:
	@echo "GO_VERSION=1.23"
	@echo "NEEDS_DOCKER=true"
	@echo "NEEDS_SUBMODULES=true"
```

| Key | 说明 | 示例 |
|-----|------|------|
| `GO_VERSION` | Go 版本 | `1.23` |
| `NEEDS_DOCKER` | 是否需要 Docker（集成测试） | `true` / `false` |
| `NEEDS_SUBMODULES` | 是否需要 git submodules | `true` / `false` |

### 覆盖率输出契约

测试覆盖率文件输出到 `coverage/` 目录，使用 Go 原生格式（`.out`）。SonarQube 通过业务仓库的 `sonar-project.properties` 中 `sonar.go.coverage.reportPaths` 配置读取。

业务仓库负责：
- 在 `ci-unit-test` 和 `ci-integration-test` 中生成覆盖率文件到 `coverage/`
- 在 `sonar-project.properties` 中声明覆盖率文件路径

构建仓库负责：
- 收集 `coverage/` 目录作为 artifact
- 将 artifact 传递给 SonarQube 扫描步骤

### 环境变量传递

构建仓库在 init 阶段计算以下值，以环境变量形式传给业务仓库的 make targets：

| 变量 | 来源 | 说明 |
|------|------|------|
| `BASE_REV` | `git merge-base` (PR 基线与 HEAD 的公共祖先 SHA) | 增量扫描基准，仅 PR 时设置；未设置时业务仓库应全量扫描 |
| `BUILD_ID` | `github.run_id` | 隔离并发运行的集成测试环境 |
| `VERSION` | `client_payload.version` / `workflow_dispatch.inputs.version`，为空时由 `ttpos-ci` 计算 | 最终镜像 tag，传给 `ci-build` 时必须非空 |
| `REF_NAME` | `client_payload.ref_name` / `workflow_dispatch.inputs.ref_name` | 分支/tag 上下文，仅用于日志或业务仓库本地兜底 |
| `SHA` | `client_payload.sha` / `workflow_dispatch.inputs.sha` | 构建提交 SHA |

业务仓库的 make target 应通过 `$BASE_REV` 环境变量读取，不应假设 git 远程分支可用：

```makefile
# 正确：读环境变量，有则增量，无则全量
ci-lint:
	cd main && golangci-lint run $${BASE_REV:+--new-from-rev=$$BASE_REV}

# 错误：依赖 git 上下文
ci-lint:
	cd main && golangci-lint run --new-from-rev=origin/dev
```

## Dispatch Payload 规范

业务仓库的 `dispatch.yml` 发送以下 payload：

```json
{
  "sha": "完整 commit SHA",
  "pr_number": "PR 编号（非 PR 时为空）",
  "base_ref": "目标分支名",
  "head_ref": "源分支名",
  "ref_name": "分支/tag 名",
  "event_name": "触发事件类型",
  "event_action": "触发动作（可选）",
  "source_repo": "owner/repo",
  "version": "最终镜像 tag",
  "images": ["需要构建的镜像名"]
}
```

### 镜像版本契约

构建仓库负责计算默认镜像版本号，并通过 `VERSION` 传给业务仓库的 `make ci-build`。业务仓库也可以在 `client_payload.version` 中显式指定版本；一旦指定，构建仓库会原样使用该值。

版本号规则：

- PR / 分支构建：`<safe-branch-name>-<short-sha>`，其中分支名会按 Docker tag 规则清洗；非法字符替换为 `-`，开头非法字符移除，并保留空间给短 SHA
- tag 构建：`<tag-name>`

业务仓库的 `ci-build` 必须使用非空 `VERSION` 作为镜像 tag。`REF_NAME` 只表示触发来源上下文，不应作为最终版本号来源或兜底来源。

后向兼容：

- 旧业务仓库没有发送 `version` 时，构建仓库会根据 `ref_name` / `event_name` / checkout 后的 `HEAD` 计算默认 `VERSION`
- 新版构建仓库传给业务仓库 `ci-build` 的 `VERSION` 必须非空，业务仓库不需要再保留版本兜底逻辑
- 在 `ttpos-ci` 手动触发 `build.yml` 时，`version` 是可选的镜像 tag override；不填写则由 `ttpos-ci` 根据 `ref_name` 和 checkout 后的短 SHA 自动计算

| event_type | 触发条件 | 说明 |
|------------|----------|------|
| `ci-go` | `pull_request` | 触发 Go CI 流水线 |
| `build` | `push tags` / `workflow_dispatch` / CI 成功后触发 | 触发构建发布流水线 |

手动触发 `build.yml` 时，`images` 输入使用 JSON 数组字符串，例如 `["ttpos-server-go"]`。留空数组 `[]` 表示交给业务仓库 `ci-build` 构建全部镜像。

手动验证 tag 打包时，将 `ref_type` 设置为 `tag`，`ref_name` 填 tag 名，`version` 留空即可验证自动版本为原始 tag 名。

## Workflows

| 文件 | 说明 |
|------|------|
| `ci-go.yml` | Go 业务仓库 CI（lint → test → sonarqube → build dispatch） |
| `ci-flutter.yml` | Flutter 前端仓库 CI |
| `build.yml` | Docker 镜像构建与发布 |
