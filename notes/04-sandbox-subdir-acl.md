# 04 · 工作区能写、子目录写不进去

> 平台：Windows 11 + DSH `workspace-write` 沙箱

## 症状

工作区根目录可以写文件，但任何子目录都报：

```
[sandbox: file access denied under workspace-write mode]
PermissionError / UnauthorizedAccessException
```

## 根因

不是代码问题，是 **NTFS ACL**：子目录的 DACL 里缺少当前用户对 `WRITE_OWNER` 的有效权限（只继承了 Modify / ReadAndExecute）。DSH 的沙箱授权（provisioning）因此无法完成，子目录写入被判为越界。根目录恰好有一条显式的 FullControl ACE，所以能写。

## 修复

用官方 skill `diagnose-windows-sandbox-acl` 的「一条命令诊断 + 修复」：

```powershell
Set-ExecutionPolicy -Scope Process Bypass
.\diagnose-windows-sandbox-acl.ps1 -Path '<失败的子目录>' -AllowRoot '<工作区根>' -Out '<工作区旁的持久目录>'
```

它会检查路径本身和每一级祖先，只对**证据成立**的 ACL 问题动手，并留下备份与回滚命令。

## 注意事项

- `-Path` 必须是**已存在**的目录（不存在会得到 `NOT_THIS_CLASS / skipped`）
- 执行策略默认禁止 `.ps1`，先 `Bypass`
- 修复 ACL 需要一次性更宽权限（受限令牌下改不了 ACL）

## 教训

看到「根目录能写、子目录不能写」这种怪异现象，**先怀疑 ACL，不要怀疑自己的代码**；也不要手工 icacls 乱补，用官方诊断脚本。
