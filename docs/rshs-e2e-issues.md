# rshs 端到端测试问题与未迁移清单

本文档记录本项目与真实 WebDAV 服务器 [`rshs`](https://github.com/mogeko/rshs) 对接时发现的问题，以及从参考实现
（`references/webdav-client`）迁移真实测例时**刻意未迁移**的部分及原因。

> 说明：本文档纠正了早期草稿中的一处错误结论。旧草稿把“`Depth: 1` 只返回目录自身”归因为
> “发布镜像与参考源码不一致”，该结论不成立。同一个镜像 digest 下，真正原因见 3.1 节。

## 1. 测试概况

| 项目        | 内容                                                                       |
| ----------- | -------------------------------------------------------------------------- |
| 服务端镜像  | `ghcr.io/mogeko/rshs:latest`                                               |
| 镜像 digest | `sha256:8e9e8315fd358cb4ce6892c160e5536e0775ce0ece43036c96cb77a930fe6ecb`  |
| 客户端      | 本项目 Effect 实现（`src/`）                                               |
| 测试基座    | `tests/e2e/harness.ts`（Docker 容器生命周期 + 命名卷重置）                 |
| 迁移的测例  | `tests/e2e/rshs.test.ts`                                                   |
| 测试数据    | `tests/e2e/fixtures/serverContents/`（复制自参考库 `test/serverContents`） |
| 认证        | Basic Auth，`RSHS_USERS=admin:pass`                                        |

结果：

```text
vp check                      通过（格式、lint、类型均无错误）
vp test                       75 passed, 69 skipped   （默认跳过 e2e）
vp run test:e2e               69 passed
WEBDAV_E2E=1 vp test          144 passed              （75 单元 + 69 端到端）
```

运行方式：

```bash
vp run test:e2e     # 需要 Docker，harness 会自动启停 rshs 容器
```

## 2. 测试基座说明

`tests/e2e/harness.ts` 负责容器生命周期，两个 Docker 相关细节值得记录：

1. **必须关闭 seccomp**：rshs 通过 `io_uring` 批量执行 `statx`（见 `references/rshs/src/scandir.rs`）。
   Docker 默认 seccomp 配置会拦截 `io_uring` 系统调用，导致所有目录项被静默跳过。容器因此以
   `--security-opt seccomp=unconfined` 启动。
2. **不能直接挂载宿主机目录**：在 Docker Desktop for macOS 上，对 bind mount 递归删除并重建条目后，
   容器会保留过期的目录视图，后续请求返回 `ENOENT`。因此服务数据放在**命名卷**中，并用一个常驻
   `busybox` sidecar 在每个用例前把预置 fixture 复制回卷内。

正因如此，端到端用例统一通过 WebDAV API 断言变更结果，而不是断言宿主机文件系统。

## 3. rshs 服务端问题

### 3.1 (S-01) Docker 默认 seccomp 拦截 io_uring，导致目录列举为空

**现象**

```http
PROPFIND /sub1/
Depth: 1
```

rshs 返回 `207 Multi-Status`，但响应只包含目录自身，没有子项：

```xml
<D:multistatus xmlns:D="DAV:">
  <D:response>
    <D:href>/sub1/</D:href>
    ...
  </D:response>
</D:multistatus>
```

rshs 日志显示 `directory listing entry_count=0`、`PROPFIND ... entries=1`。

**根因**

rshs 在 Linux 上使用 `io_uring` 批量提交 `statx`（`references/rshs/src/scandir.rs` 的 `batch_linux`）。
Docker 自 24.0 起在默认 seccomp profile 中屏蔽了 `io_uring_setup` / `io_uring_enter` 等系统调用；
`batch_statx` 失败后，目录项被当作“元数据读取失败”而全部跳过，于是只剩 self。

**修复 / 规避**

启动容器时加入：

```bash
docker run --security-opt seccomp=unconfined ...
```

之后 `Depth: 1` 正常返回集合自身与直接子资源。本项目 `harness.ts` 已固化该参数。

**影响**

- 旧草稿据此得出的“镜像与参考源码版本不一致”结论**不成立**：同一 digest 下，仅 seccomp 配置不同即可复现/消除。
- 客户端的目录查询默认排除 self，因此受影响时会得到空数组；`search` 等基于目录读取的能力同样失效。

**复现命令**

```bash
# 默认配置：只返回 self
docker run --rm -p 8080:8080 -v "$PWD/data:/mnt/data" ghcr.io/mogeko/rshs:latest
curl -s -u admin:pass -X PROPFIND -H 'Depth: 1' http://127.0.0.1:8080/sub1/

# 关闭 seccomp：正常返回子项
docker run --rm --security-opt seccomp=unconfined -p 8080:8080 \
  -v "$PWD/data:/mnt/data" ghcr.io/mogeko/rshs:latest
curl -s -u admin:pass -X PROPFIND -H 'Depth: 1' http://127.0.0.1:8080/sub1/
```

### 3.2 (S-02) PUT 未实现 `If-None-Match: *`，`overwrite: false` 会静默覆盖

**现象**

客户端在 `overwrite: false` 时发送：

```http
PUT /notes.txt
If-None-Match: *
```

目标文件已存在，rshs 仍返回成功，客户端得到 `true`，文件内容被覆盖。

**根因**

参考源码 `references/rshs/src/handlers/http.rs` 的 `handle_put` 只检查目标是否存在，随后直接
`File::create`，既不读取也不校验 `If-None-Match`。

**影响**

调用方无法用 `putFileContents(path, data, { overwrite: false })` 保护已有文件，存在**数据被意外覆盖**的高风险。
“先 `HEAD` 再 `PUT`”无法替代，因为并发下仍有竞态。

**对应测试**

`tests/e2e/rshs.test.ts` → `putFileContents` → _does not protect existing files when overwrite is disabled
(rshs limitation)_。该用例固化当前（不安全）行为，使风险可见而不是让它“静默通过”。

### 3.3 (S-03) GET 未实现 `Range`

**现象**

```http
GET /alrighty.jpg
Range: bytes=0-24999
```

rshs 返回 `200 OK` 和完整文件（`Content-Length: 52130`），没有 `206 Partial Content` 与 `Content-Range`。

**根因**

参考源码 `handle_get_head` 在文件响应路径直接以 `200 OK` + 完整文件大小构造响应，从文件开头创建
`ReaderStream`，未解析 `Range`。源码顶部 “Accepts `Range` requests” 注释与实现不一致。

**影响**

客户端 `createReadStream` 在带 `range` 时严格要求 `206`，因此会以 `HttpStatusError(200)` 失败。
这一严格行为本身更安全（不会把完整文件误当成范围结果）。

**对应测试**

`tests/e2e/rshs.test.ts` → `createReadStream` → _fails when the server ignores the Range header (rshs limitation)_。

### 3.4 UNLOCK 无效 token 返回 `403`

参考实现（进程内 `webdav-server`）返回 `409 Conflict`；rshs 返回 `403 Forbidden`。客户端如实映射为
`HttpStatusError(403)`。对应测试：`lock / unlock` → _fails unlocking with an invalid token_。

### 3.5 仅支持 Basic Auth

rshs 只实现 Basic Auth（`--user` / `RSHS_USERS`）。参考库的 Digest 用例无法在 rshs 上运行；Bearer token
请求返回 `401`，客户端归类为 `AuthenticationError`。对应测试：`authentication` → _rejects Bearer tokens
because rshs only supports Basic_。

### 3.6 不支持 SEARCH

rshs 未实现 `SEARCH` 方法。参考库中依赖 fake response 的 `search` 用例未做真实服务器迁移。

### 3.7 不提供 quota 属性

对根执行 `PROPFIND Depth: 0` 不返回 `quota-used-bytes` / `quota-available-bytes`，客户端 `getQuota()`
返回 `null`。对应测试：`getQuota` → _returns null when the server does not expose quota properties_。

## 4. 迁移过程中修复的客户端问题

这些是真实服务器测例暴露出来的客户端缺陷，已在 `src/operations.ts` 修复：

| 问题                                       | 修复前                                               | 修复后                                                                |
| ------------------------------------------ | ---------------------------------------------------- | --------------------------------------------------------------------- |
| 请求路径未做百分号编码                     | `file % name.txt` 生成非法 URL `file%20%%20name.txt` | 按路径段 `encodeURIComponent`，并把 `remoteUrl` 的 path 作为默认 base |
| 目录请求缺少结尾 `/`                       | 对 `/sub1` 请求，rshs 只返回 self，目录内容为空      | `getDirectoryContents` 自动补 `/`                                     |
| `filename` / `basename` 未解码、未去尾斜杠 | 返回 `/two%2520words`、`/sub1/`                      | 解码 href 并规范化（`/two%20words`、`/sub1`）                         |
| `getDAVCompliance` 读错响应头              | 读取 `Allow`（HTTP 方法）                            | 读取 `DAV`（合规级别 `1,2`），与参考语义一致                          |

相应地更新了既有单元测试：`tests/operations/basic.test.ts`、`tests/operations/query.test.ts`（compliance 头）、
`tests/operations/fixtures.test.ts`（HTML 实体 href 改为解码后形式）。

## 5. 刻意未迁移的参考测例

| 参考测例                                                                        | 未迁移原因                                                                                                                                                                     | 现状                                                                    |
| ------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------- |
| `auth.spec.ts` 中 Digest server 用例                                            | rshs 仅支持 Basic                                                                                                                                                              | 仅迁移 Basic / auto / 匿名 / 错误密码 / Token 场景                      |
| `stat` / `getDirectoryContents` / `getQuota` / `search` 的 `details: true` 用例 | 当前实现仅在 `getFileContents` 支持 `details`（返回 `body`/`headers`），其余操作会忽略 `details`                                                                               | 未迁移；需先补齐 `details` 支持                                         |
| `search.spec.ts`（真实服务器部分）                                              | rshs 未实现 `SEARCH`                                                                                                                                                           | 未迁移；fake-response 覆盖已由 `tests/operations/fixtures.test.ts` 保留 |
| `partialUpdateFileContents.spec.ts`                                             | 新 API 签名为 `(path, data, options)`，与参考 `(path, start, end, data, options)` 不同；且当前实现直接退化为整体覆盖 PUT，未按迁移计划做能力探测并抛 `UnsupportedFeatureError` | 未迁移；存在覆盖风险，见下                                              |
| `createReadStream.spec.ts` 的 callback 用例                                     | 新 API 为 `Effect<Stream>`，不暴露 Node `Readable` / `callback`                                                                                                                | 已按 Stream 语义迁移其余用例                                            |
| `createWriteStream.spec.ts` 的 callback 用例                                    | 新 API 接收 Effect `Stream`，不暴露 `callback(response)`                                                                                                                       | 已按 Stream 语义迁移写入用例                                            |
| `getFileDownloadLink` / `getFileUploadLink` 的凭据 URL 用例                     | **安全变更**：新 API 返回不含凭据的请求描述（`{ url, method, headers }`）                                                                                                      | 已按新语义迁移（断言 URL 不含 `@`）                                     |

### 5.1 `partialUpdateFileContents` 风险说明

当前实现：

```ts
const partialUpdate = (path, data, options) =>
  putFileContents(path, data, { ...options, overwrite: true });
```

即忽略 `start` / `end`，把数据当作完整文件覆盖。这与迁移计划 “先探测 DAV 能力，再选择 SabreDAV `PATCH`
或 Apache `PUT` + `Content-Range`，否则抛 `UnsupportedFeatureError`” 的目标不一致，**可能导致数据被静默覆盖**。
在实现能力探测之前，不建议在生产中使用该操作。

## 6. 建议的回归清单

修复 rshs 或调整客户端策略后，应重跑以下真实请求：

- `OPTIONS`：解析 `DAV`（`1,2`）、`Allow`、`Server`（覆盖大小写混用）。
- `PROPFIND Depth: 0`：单资源 stat。
- `PROPFIND Depth: 1`：集合自身 + 直接子资源（必须在 `seccomp=unconfined` 下验证）。
- `MKCOL`：普通与递归创建、部分已存在。
- `PUT`：新建与覆盖。
- `PUT If-None-Match: *`：目标存在时应返回 `412` 且内容不变；在 rshs 修复前应明确标记为“不支持”，不得判为通过。
- `GET`：文本 / 二进制 / 详细响应。
- `GET Range`：服务端实现后必须返回 `206` 与 `Content-Range`；当前 rshs 返回 `200`，标记为“不支持”。
- `COPY` / `MOVE`：目标存在与不存在两种情况（`Overwrite: T/F`）。
- `LOCK` / `UNLOCK`：token 生命周期；无效 token 的状态码。
- Basic Auth：正确、错误、匿名。
- `DELETE`：文件与目录。

## 7. 相关文件

- `tests/e2e/harness.ts`：容器与命名卷生命周期。
- `tests/e2e/rshs.test.ts`：迁移后的端到端用例。
- `tests/e2e/fixtures/serverContents/`：测试夹具。
- `src/operations.ts`：第 4 节的客户端修复。
- `references/rshs/src/scandir.rs`、`src/handlers/http.rs`：上述服务端行为的源码依据。
