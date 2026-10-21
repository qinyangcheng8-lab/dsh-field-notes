# 05 · 给 DSH 写带面板的插件：双段装配

## 结论先行

带 UI 的 DSH 插件 = **host 段（RPC/工具）+ client 段（声明式槽位注入）**。client 段不是「改 HTML」，而是往 UI 壳声明的槽位里填组件。

## 三段式 package.json

```json
{
  "exports": {
    ".": "./dist/index.js",
    "./client": "./client/client.js"
  },
  "dsh": {
    "bundle": { "patch": "./cordis.patch.yml" },
    "client": {
      "platform": "web",
      "inject": ["@deepseek-ai/dsh-client-connection", "@deepseek-ai/dsh-client-ui-settings"]
    }
  }
}
```

## client.js 骨架

```js
window.__ModuleLoader__.load({
  id: 'my-panel',
  factory: (require) => {
    const React = require('react');
    const h = React.createElement;

    function Panel() {
      const [data, setData] = React.useState(null);
      React.useEffect(() => {
        fetch('/api/mypanel.status', { credentials: 'include' })
          .then(r => r.json()).then(j => setData(j.result?.value));
      }, []);
      return h('div', null, data ? JSON.stringify(data) : 'loading...');
    }

    return {
      name: 'my-panel',
      inject: ['slots', 'locale'],
      apply(ctx) {
        ctx.effect(() =>
          ctx.slots.inject('sidebar.footer.action', () =>
            ctx.slots.register(
              { name: 'sidebar.footer.action', id: 'my-panel', order: 100, label: 'My Panel' },
              Panel
            )
          )
        );
      },
    };
  },
});
```

## 可注入的槽位（实测）

- `sidebar.footer.action`
- `settings.plugins.tab`
- `settings.section`
- `conversation.session.header.actions`

⚠️ `root` 槽位**禁止注册**（single 语义会遮蔽整页 UI）。

## 验证渲染

用 `playwright-core` + Edge headless 打开本地 web 服务，断言：

1. DOM 里元素 count > 0
2. 点击能弹出面板
3. console 零错误
4. 截图留档

**踩坑**：别的插件的引导层（全屏 backdrop）会拦截点击，冒烟脚本里先移除再点。

## 教训

- 改 `client.js` 后必须重启 web 服务（启动时快照）
- 槽位注册的形状是 `{name, id, order, label, locale, inject}`
- 先做 host RPC 再用 `fetch` 打通，比一次性写完整 UI 好调试
