# 03 · DSH 0.2 的 RPC 契约四件套

> 实测：0.2.0-rc.2。照旧版写法直连 `/api` 的脚本，在 0.2 上会 401 或 404。

## 四处同时变了

| # | 变化 | 症状 |
|---|---|---|
| ① | 认证闸在路由之前 | 无 cookie 一律 **401**（掩盖了后面的问题） |
| ② | 核心 API 路径由点号改斜杠 | `/api/session/prompt`；点形式 404 |
| ③ | payload 必须装箱 | 裸 body 报 `Remote payload must contain exactly one plain-object args field` |
| ④ | 内层字段名分两套 | `session/prompt` 用 `args.request`；`session/list` 用 `args._request` |

## 正确形状

```http
POST /api/session/list
Cookie: <ds-auth cookie>
Content-Type: application/json

{"args":{"_request":{"requestId":"...","offset":0,"limit":100}}}
```

```http
POST /api/session/prompt
Cookie: <ds-auth cookie>

{"args":{"request":{"requestId":"...","sessionId":"session-...","mode":"...","content":"..."}}}
```

## 排查建议

动手前先用一个 20 行探针脚本实测「路径形态 × 装箱形态 × 有无 cookie」矩阵，不要照抄别的脚本——同一台机器上可能同时存在斜杠端点（核心 API）和点号端点（插件 `webServer.register` 的 RPC）。

## 名单源

`sessions.overview` 在部分环境下返回不全（不含活跃会话），改用 `session/list` 作为主名单源，overview 只作去重补充，且失败不阻断。

## 教训

- 401 会掩盖 404：先补 cookie，再看端点
- 只读工具也要加 `--dry` 契约自检，否则「跑不出数据」和「没有数据」分不开
