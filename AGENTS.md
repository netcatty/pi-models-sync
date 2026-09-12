# AGENTS.md — pi-models-sync

Pi 扩展：把 `models.json` 热同步进当前会话。

## 结构

```
extensions/models-sync/index.ts        # 入口，转发 default
extensions/models-sync/models-sync.ts  # 全部实现，224 行
```

无构建步骤——pi 直接加载 TypeScript 源文件。

## 为什么需要这个扩展

三个源码级事实（**改代码前先确认它们是否仍成立**）：

| 事实 | 验证方法 |
|------|---------|
| `/reload` 不重读 `models.json` | `resource-loader.js` 的 `reload()` 只调 `settingsManager.reload()` |
| pi 不监听任何配置文件 | `dist/` 里搜 `fs.watch` / `watchFile` / `chokidar` —— 零命中 |
| 内置 provider 会叠加 `models-store.json` | 自定义 `thinkingLevelMap` / `cost` 被覆盖 |

**如果 pi 上游修了其中任何一条，这个扩展就可以删了。**

## 核心做法

```javascript
pi.unregisterProvider(id);                        // 先卸掉内置的
pi.registerProvider(id, normalizeProvider(p));    // 带 models 的注册 = 整表替换
await ctx.modelRegistry.refresh({ allowNetwork: false });
```

带 `models` 数组的 `registerProvider` 会**整表替换**该 provider 的模型列表——这是让文件成为唯一来源的关键。

## 钩子

| 事件 | 时机 |
|------|------|
| `session_start` | 启动时同步一次 |
| `fs.watch(AGENT_DIR)` | 文件变化时同步（带防抖） |
| `session_shutdown` | 关掉 watcher |

命令：`/sync-models` 手动触发。

## 路径约定

```javascript
const AGENT_DIR = process.env.PI_CODING_AGENT_DIR || join(homedir(), ".pi", "agent");
```

**如果 pi 改了目录约定，这里要跟着改。**

## 坑

**不要在 `models.json` 的 provider 上写 `headers.Authorization`。** 显式头会覆盖 pi 生成的鉴权头，空字符串也一样——等于告诉服务端「我没有凭据」。

## 命令

```bash
npm run typecheck      # tsc --noEmit
```

本地测试：

```bash
pi install /absolute/path/to/pi-models-sync
/reload
# 然后改 ~/.pi/agent/models.json，几秒内应看到同步通知
```

## 已知待办

- [ ] 无测试（扩展会读文件 + 调 modelRegistry，可 mock）
- [ ] 通知策略是「每次都弹」，可加安静模式
