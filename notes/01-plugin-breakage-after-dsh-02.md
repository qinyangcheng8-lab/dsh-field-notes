# 01 · 升级到 DSH 0.2 后，第三方插件为什么集体失效

> 实测环境：DSH 0.2.0-rc.2，Windows 11

## 症状

升级到 0.2 之后，好几个第三方插件「装了但页面没效果」：

- 侧聊类插件（网页版对话省钱）直接消失
- 图片理解插件设置页不见了
- 压缩类插件启动时报 `failed to import`

而 `package.json` 里包还在、目录也在。

## 根因（两层）

**第一层：导出的 API 被删了。**

```
The requested module '@deepseek-ai/dsh-settings' does not provide an export named 'installSettingsSection'
The requested module '@deepseek-ai/dsh-settings' does not provide an export named 'settingsNamespace'
The requested module '@deepseek-ai/dsh-llm' does not provide an export named 'assertNever'
```

按旧名字静态 `import` 的插件，在**模块链接期**就失败，host 段根本挂不上——所以不是「UI 没渲染」，是根本没加载。

**第二层：bundles 才是启用清单。**

DSH 的 profile 只按 `package.json` 里的 `dsh.profile.bundles` 顺序加载插件（再叠加各自 bundle 的 `cordis.patch.yml` 与用户层 patch）。**没列进 bundles 的包，即使躺在 node_modules 里也不会生效**——它们只是残留孤儿。

## 排查方法：import 冒烟

对 profile 的 `node_modules` 里所有插件包逐个动态 import，几秒就能拿到全量健康报告：

```js
import { pathToFileURL } from 'node:url';
import { readFileSync, readdirSync, existsSync } from 'node:fs';
import path from 'node:path';

const profile = process.argv[2]; // 例如 $DSH_HOME/profiles/web
const nm = path.join(profile, 'node_modules');

function entryOf(dir) {
  const pkg = JSON.parse(readFileSync(path.join(dir, 'package.json'), 'utf8'));
  const ex = pkg.exports?.['.'];
  const rel = typeof ex === 'string' ? ex : ex?.default || pkg.main || 'index.js';
  return path.join(dir, rel);
}

for (const name of readdirSync(nm).filter(n => /^dsh-/.test(n))) {
  const dir = path.join(nm, name);
  if (!existsSync(path.join(dir, 'package.json'))) continue;
  try {
    await import(pathToFileURL(entryOf(dir)).href);
    console.log('OK  ', name);
  } catch (e) {
    console.log('ERR ', name, '|', String(e.message).split('\n')[0]);
  }
}
```

输出里只要出现 `does not provide an export named`，就是 DSH 移除了 API，插件需要更新或本地 shim，不必再去页面里排查 UI。

## 处置

1. **失效且未启用** → 隔离（移动到 `_tmp/plugin-quarantine-<date>/`，别直接删），可从 `dependencies` 摘除
2. **失效但在用** → 找上游更新；没有更新就自己打兼容 shim（例如把被删的导出改成可选 shim）
3. **能加载但没列进 bundles** → 要么加进 bundles，要么清理，别让它躺在那里误导判断

## 教训

- 大版本升级后，第一件事是**跑 import 冒烟**，不是逐个开页面点
- 判断「装了没用」和「装了坏了」看两个地方：bundles（启用与否）、import（能否链接）
- 隔离式清理（rename 而非 delete）能让你在十分钟内回滚
