# pi-models-sync

把 `models.json` **热同步**进当前 pi 会话——改完存盘即生效，不用重启。

## 解决什么

pi 的模型配置有几个反直觉的行为：

| 行为 | 后果 |
|------|------|
| `/reload` **不**重读 `models.json` | 改了没反应，只能重启 pi |
| pi **不监听**任何配置文件 | 开着 pi 改配置，它不知道 |
| 内置 provider 会把 `models-store.json` 的远程模型**叠加**到自定义列表 | 你写的 `thinkingLevelMap`、`cost` 被悄悄覆盖 |

第二条是源码级事实：`resource-loader.reload()` 只重载 **settings / skills / extensions**，整个 `dist/` 里没有任何针对配置文件的 `fs.watch`。

## 装之前 vs 装之后

| 场景 | 没装 | 装了 |
|------|------|------|
| 改 `models.json` 的 cost | 重启 pi | ✅ 存盘即生效 |
| 加一个 provider | 重启 pi | ✅ 存盘即生效 |
| 改 `thinkingLevelMap` | 可能被 `models-store.json` 覆盖 | ✅ 以文件为唯一来源 |
| 切换模型参数 | 重启 pi | ✅ 立即 |

## 安装

```bash
# 从 GitHub
pi install git:github.com/netcatty/pi-models-sync

# 固定版本
pi install git:github.com/netcatty/pi-models-sync@v1.0.0
```

重启 pi 或 `/reload`。扩展在 `session_start` 时自动同步一次。

## 用法

装好后**不需要做任何事**——它会：

1. `session_start` 时读取 `~/.pi/agent/models.json` 并同步
2. 监听配置目录，文件变化时（带防抖）自动重新同步
3. 同步成功/失败会弹通知

**手动触发**：

```
/sync-models
```

## 工作原理

```javascript
// 对 models.json 里的每个 provider：
pi.unregisterProvider(id);                        // 先卸掉内置的
pi.registerProvider(id, normalizeProvider(p));    // 带 models 的注册 = 整表替换
await ctx.modelRegistry.refresh({ allowNetwork: false });
```

**关键点**：带 `models` 数组的 `registerProvider` 会**整表替换**该 provider 的模型列表——所以当前会话的模型列表以 `models.json` 为唯一来源，不会被内置目录的远程模型叠上去。

监听用 `fs.watch(AGENT_DIR)` + 防抖（避免编辑器多次写入触发重复同步）。

## 配置

无。它读的就是 pi 自己的 `~/.pi/agent/models.json`。

路径解析遵循 pi 约定：

```javascript
const AGENT_DIR = process.env.PI_CODING_AGENT_DIR || join(homedir(), ".pi", "agent");
```

所以设了 `PI_CODING_AGENT_DIR` 也能正确工作。

## 支持的字段

除了标准的 `baseUrl` / `api` / `apiKey` / `models`，还处理：

| 字段 | 说明 |
|------|------|
| `headers` | 自定义请求头 |
| `compat` | 兼容性开关（`sendSessionAffinityHeaders` 等） |
| `authHeader` | 自动加 `Authorization: Bearer` |
| `modelOverrides` | provider 级模型字段覆盖 |

`models.json` 支持 **JSON with comments**——扩展会先剥离注释再解析。

## ⚠️ 注意：不要写 `headers.Authorization`

```json
// ❌ 会让 key 失效
{ "headers": { "Authorization": "" } }
```

显式的 `Authorization` 头会**覆盖** pi 生成的鉴权头。空字符串也一样——等于告诉服务端「我没有凭据」。

正确做法是只写 `apiKey` + `authHeader: true`，让 pi 生成。

## 已知限制

- **只读不写**——扩展只读 `models.json`，不会帮你改文件
- **依赖 `PI_CODING_AGENT_DIR` 语义**——如果 pi 改了目录约定，这里要跟着改
- **通知会打断**——同步成功也会弹一次 `info` 通知

## 兼容性

| 依赖 | 版本 |
|------|------|
| `@earendil-works/pi-coding-agent` | `*` (peer) |

开发时用 pi `0.85.1` 验证。

## 开发

```bash
git clone git@github.com:netcatty/pi-models-sync.git
cd pi-models-sync

# 本地挂载
pi install /absolute/path/to/pi-models-sync

/reload
```

改 `models.json` 测试：应该在几秒内看到同步通知。

## License

[MIT](LICENSE)
