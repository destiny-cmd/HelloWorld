# Windows 系统问题排查

---

## 任务管理器无法打开

### 起因

电脑任务管理器完全无法打开，点击后没有任何反应。

### 排查过程

**第一步：尝试多种打开方式**

- `Ctrl + Shift + Esc`
- `Ctrl + Alt + Delete` → 任务管理器
- 右键任务栏 → 任务管理器
- `Win + R` 输入 `taskmgr`

全部没有反应，说明不是快捷键问题。

---

**第二步：检查注册表策略（DisableTaskMgr）**

打开注册表，检查以下两个路径有没有 `DisableTaskMgr` 键值：

```
HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Policies\System
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System
```

- `HKEY_CURRENT_USER` 下没有 System 文件夹
- `HKEY_LOCAL_MACHINE` 下有 System 但没有 `DisableTaskMgr`

**结论：不是注册表策略禁用导致的。**

---

**第三步：尝试用 gpedit.msc 检查组策略**

`Win + R` 输入 `gpedit.msc`，报错：

> Windows 找不到文件 'gpedit.msc'

**原因：** 使用的是 Windows 家庭版，家庭版没有本地组策略编辑器，只有专业版、企业版、教育版才有。

---

**第四步：确认 taskmgr.exe 文件是否存在**

在 `C:\Windows\System32\` 搜索 `taskmgr.exe`，文件存在，大小 5.18MB，正常。

**结论：不是文件缺失或损坏导致的。**

---

**第五步：用 PowerShell 强制启动**

```powershell
Start-Process "C:\Windows\System32\taskmgr.exe" -Verb RunAs
```

命令执行后没有任何输出，任务管理器也没有打开。这是典型的**被静默拦截**症状。

---

**第六步：检查映像劫持（IFEO）**

检查注册表路径：

```
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\taskmgr.exe
```

该路径不存在，说明没有被映像劫持。

但在 `Image File Execution Options` 下发现了可疑条目：

- `LSASS.exe` 出现在列表中（正常系统不应该有）
- 点进去发现只有 `AuditLevel = 8`，后确认这是 Windows 正常的安全审计配置，不是病毒

**结论：不是映像劫持导致的。**

---

**第七步：运行 SFC 和 DISM 修复**

```powershell
sfc /scannow
DISM /Online /Cleanup-Image /RestoreHealth
```

- DISM 修复成功
- SFC 未发现完整性冲突

修复后任务管理器从"没有反应"变成了"闪退"，有进展！

---

**第八步：查看崩溃日志**

```powershell
& "C:\Windows\System32\taskmgr.exe"; Start-Sleep -Seconds 2; Get-EventLog -LogName Application -Newest 5 | Select-Object TimeGenerated, Source, Message | Format-List
```

日志显示：

```
错误应用程序名称: taskmgr.exe
异常代码: 0xc0000005
错误偏移量: 0x00000000000bd842
```

关键发现：**每次崩溃的错误偏移量完全一样**（`0x00000000000bd842`），说明程序每次都在同一个位置崩溃，是固定的功能触发了问题。

**错误码 `0xc0000005` = 访问冲突（Access Violation）**，程序启动时试图访问没有权限的内存区域。

---

**第九步：发现版本不匹配**

- taskmgr.exe 版本：`10.0.22621.3085`
- 系统镜像版本：`10.0.22631.3155`

两个版本号不一致！说明 taskmgr.exe 没有随系统正确更新，这是导致崩溃的根本原因。

**为什么会版本不匹配？**

- Windows 更新被暂停到了 2032 年（用工具强制暂停的）
- 但安全补丁绕过暂停强制安装，导致部分组件更新了，整体版本没跟上
- taskmgr.exe 就是这样被单独更新又没更新完整导致的

---

### 中途发现宏病毒

火绒全盘扫描时发现：

```
风险类型：宏病毒（OMacro/Downloader.axn）
风险路径：C:\$Recycle.Bin\...\$R86GU9G.xlsb
```

**宏病毒特点：**

- 藏在回收站里的 Excel 二进制文件（.xlsb）中
- 会感染 Office 系列文档，通过 Office 模板传播
- 检查 Word 模板文件（Normal.dotm）修改时间正常，未被感染
- 火绒处理后清除

---

### 最终解决方案

由于不想更新 Windows，采用替代方案：

- 使用 **System Informer**（Process Hacker 官方继任者）替代任务管理器
- 使用 **火绒剑**（火绒自带，中文界面）替代任务管理器
- 等 Windows 自然更新后 taskmgr.exe 会自动修复

---

## 映像劫持（IFEO）原理详解

### 什么是 IFEO？

IFEO 全称 **Image File Execution Options**（映像文件执行选项），位于注册表：

```
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options
```

### 正常用途

微软设计它是给开发者调试程序用的：

> "每次启动 xxx.exe 时，自动附加调试器"

这样程序一启动就能被调试器捕获，方便排查 bug。

### 被病毒滥用的原理

1. 病毒在 IFEO 下创建 `taskmgr.exe` 键
2. 添加 `Debugger` 值，指向病毒自身的路径
3. 之后每次启动任务管理器，Windows 先启动病毒
4. 病毒选择不启动真正的任务管理器
5. 用户看到的就是"没有反应"

**为什么杀毒软件有时检测不到？** 因为注册表里的操作本身是合法的，IFEO 是系统正常功能，病毒只是把它用在了不该用的地方。

### 本次排查结果

检查发现 `taskmgr.exe` 的 IFEO 条目不存在，说明任务管理器的问题不是映像劫持导致的，而是系统文件版本不匹配。

---

## 内存占用过高问题

### 发现问题

用 System Informer 查看进程，发现总内存占用高达 **15.54GB**（16GB 内存）。

### 主要内存杀手（按占用排序）

|进程|占用|说明|
|---|---|---|
|sqlservr.exe|734MB|SQL Server 数据库服务|
|mysqld.exe|586MB|MySQL 数据库服务|
|claude.exe × 3|约800MB|Claude 桌面客户端|
|AdsPower × 4|约600MB|指纹浏览器|
|msedge.exe × 多个|约1GB+|Microsoft Edge|
|PhoneExperienceHost|378MB|Microsoft Phone Link|

### 解决方法

停止不用的数据库服务（释放约1.3GB）：

```powershell
# 先查找真实服务名
Get-Service | Where-Object {$_.DisplayName -like "*mysql*" -or $_.DisplayName -like "*sql*"}

# 停止服务
net stop MSSQLSERVER
net stop MySQL80
net stop SQLBrowser
net stop SQLTELEMETRY
net stop SQLWriter
```

**注意：** 不能直接在进程管理器里强制终止数据库进程，会导致数据损坏，必须用 `net stop` 命令安全停止。

### 优化结果

从 15.54GB 降至 **10.37GB**，释放超过 5GB。

---

## 代理软件与任务管理器冲突

### 发现过程

新建测试账户后发现任务管理器可以正常打开，切换回主账户就不行，最终发现关闭 Clash Verge 后任务管理器恢复正常。

### 原因

Clash Verge 的 **TUN 模式**会在系统层面注入驱动，接管网络流量，这个驱动与 taskmgr.exe 启动时的内存访问产生冲突，加上 taskmgr.exe 本身版本不匹配，共同导致了崩溃。

### 解决方案

换用 **FLClash**（同样基于 Mihomo 内核，兼容性更好），或在 Clash Verge 中关闭 TUN 模式只用系统代理模式。

---
其实都是一堆屁话