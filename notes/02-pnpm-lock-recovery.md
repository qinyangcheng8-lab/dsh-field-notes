# 02 · pnpm remove 被打断后，lock 被截断怎么救

## 症状

在 profile 目录执行 `pnpm remove <plugin>`，下载阶段超时被杀。事后四处状态各不相同：

- `pnpm-lock.yaml` 从 2254 行被截断成 1325 行
- `package.json` 里的依赖**没删**
- `node_modules/<plugin>` 目录**已删**
- `node_modules/.modules.yaml` 仍然登记着这个包
- `pnpm ls` 显示的依赖树与实际完全不符

## 根因

`pnpm remove` 是「先改写 lock → 再重算 node_modules」的多阶段操作，进程被超时杀掉就停在中间态。

另外 `pnpm ls` 读的是 **`node_modules/.pnpm/lock.yaml` 这份副本**，不是根目录的 `pnpm-lock.yaml`——副本过期时，pnpm 报出来的树就是错的。

## 修复步骤

1. **先恢复**：用备份覆盖被截断的 lock（做任何 pnpm 操作前都备份）
2. **精确摘除**：在 lock 里删目标包的三处
   - `importers.\.\dependencies` 下 3 行（`name` / `specifier` / `version`）
   - `packages` 里的 `name@ver:` 块
   - `packages` 里的 `name@ver(peer):` 块
   按缩进找块边界（两空格缩进的下一个键就是块尾），别用正则乱删
3. **同步副本**：把修好的 lock 复制到 `node_modules/.pnpm/lock.yaml`
4. **清 layout**：在 `.modules.yaml` 的 layout 段删掉该包的 3 行
5. **三重验证**：`pnpm ls --depth=0`、import 冒烟、`dsh --profile web --dump-config`（合成树里 0 残留、无 skip 警告）

## 教训

- 卸载插件优先「移动目录 + 手改 `package.json`」，不要让 pnpm 动全量依赖
- 非要用 pnpm：先备份 lock 和 `.modules.yaml`，并给足超时
- junction（`link:` 到本地源码）只删链接本身，源码无损——这类可以放心操作
