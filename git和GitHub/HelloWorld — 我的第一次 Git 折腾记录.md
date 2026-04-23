# HelloWorld — 我的第一次 Git 折腾记录
> 2026-04-23，和 Claude 折腾了一整天，终于搞定了 Git + GitHub + Obsidian 全平台同步。

---

## 起因

在网上问有什么免费开源的日记软件，Claude 推荐了 Obsidian + Git + GitHub 的组合，说作为计算机专业学生早晚都要学 Git，不如顺手学了。

我的态度是：**"用 git 吧，早死晚死都得死。"**

---

## 第一关：安装 Git

去 git-scm.com 下载安装包，安装过程中有几个关键步骤：

- **默认编辑器**：改成 Notepad++（我恰好装了）
- **默认分支名**：选 Override，填 `main`，和 GitHub 保持一致
- **PATH 设置**：选第二个 `Git from the command line and also from 3rd-party software`

装完打开 PowerShell 输入 `git --version`，报错：

```
git : The term 'git' is not recognized...
```

**原因**：之前装过旧版 Git，PATH 写入失败有残留。

**解决**：控制面板卸载旧版，重新安装新版，PATH 那步其实也没动，估计是旧版残留导致的问题。

重新开 PowerShell，`git --version` 显示 `git version 2.54.0.windows.1`，成功。

---

## 第二关：在 GitHub 创建仓库

注册 GitHub 账号，右上角 + → New repository：

- 名字：`my-diary`
- 选择：**Private**（私有日记）
- 勾选：Add a README file
- 点：Create repository

仓库创建成功。

---

## 第三关：克隆仓库到本地

先配置 Git 身份：

```bash
git config --global user.name "destiny-cmd"
git config --global user.email "3023153648@qq.com"
```

然后 clone：

```bash
git clone https://github.com/destiny-cmd/my-diary.git
```

报错：

```
fatal: unable to access 'https://github.com/destiny-cmd/my-diary.git/': 
Recv failure: Connection was reset
```

**原因**：国内访问 GitHub 被墙。

**解决**：用 FLClash 代理（端口 7890），设置：

```bash
git config --global http.proxy http://127.0.0.1:7890
git config --global https.proxy http://127.0.0.1:7890
```

再次 clone，弹出 GitHub 登录窗口，点 **Sign in with your browser**，浏览器显示 **Authentication Succeeded**，回到 PowerShell，clone 自动完成：

```
Receiving objects: 100% (3/3), done.
```

把仓库克隆到了 `C:\FOR WORKING\my-diary`。

---

## 第四关：配置 Obsidian 电脑端

### 打开仓库

Obsidian → 打开本地仓库（注意不是"创建"！）→ 找到 `C:\FOR WORKING\my-diary` → 选择文件夹。

左下角显示 `my-diary`，成功。

### 安装 Obsidian Git 插件

设置 → 第三方插件 → 关闭安全模式 → 浏览 → 搜索 `git`。

第一次搜到了一个叫 **GitHobs** 的插件，不是我们要的。重新搜 `git` 等加载完全，找到作者 **Vinzent**、下载量 **240万** 的 Git 插件，安装启用。

插件设置：

- `Auto commit-and-sync interval`：`10`（每10分钟自动备份推送）
- `Auto pull interval`：`10`（每10分钟自动拉取）

电脑端配置完成。

---

## 第五关：手机端配置（Android）— 最难的部分

### 安装 Termux

从应用商店装的 Termux 版本太旧，`pkg install git` 直接报错：

```
Error: Unable to locate package git
```

换清华源也不行，密钥过期：

```
NO_PUBKEY 5A897D96E57CF20C
```

**解决**：从 GitHub Releases 下载新版 Termux，选 `arm64-v8a.apk` 安装，卸载旧版，安装新版。

新版 Termux 安装 Git：

```bash
pkg update -y && pkg install git -y
```

成功，`git version 2.54.0` 安装完成。

### 授权存储权限

```bash
termux-setup-storage
```

弹出权限请求，点允许。

### 克隆仓库到手机

把仓库复制到手机存储：

```bash
cp -r ~/my-diary /sdcard/my-diary
cp -r ~/marxism-works /sdcard/marxism-works
```

在 Obsidian 手机端打开 `/sdcard/my-diary` 作为库。

### 生成 GitHub Token

手机端 Git 无法用浏览器登录 GitHub，需要 Token 认证：

GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token (classic) → 勾选 **repo** → Generate token。

Token 只显示一次，保存好。

### 配置远程同步

```bash
git config --global --add safe.directory /storage/emulated/0/my-diary
cd /sdcard/my-diary
git remote set-url origin https://destiny-cmd:TOKEN@github.com/destiny-cmd/my-diary.git
git config remote.origin.fetch "+refs/heads/*:refs/remotes/origin/*"
git fetch origin
git checkout main
```

中间遇到了一堆报错：

```
fatal: detected dubious ownership in repository
fatal: failed to resolve HEAD as a valid ref
fatal: the requested upstream branch 'origin/main' does not exist
error: Pulling is not possible because you have unmerged files
fatal: It seems that there is already a rebase-merge directory
fatal: You are not currently on a branch
```

逐一解决后，`branch 'main' set up to track 'origin/main'` 终于出现，配置成功。

### 放弃 Obsidian Git 插件，改用脚本

手机端 Obsidian Git 插件一直报错：

```
Could not find a fetch refspec for remote "origin"
```

折腾了很久，最终放弃插件，改用 Termux 脚本手动同步。

创建同步脚本：

```bash
echo '#!/bin/bash' > ~/sync-diary.sh
echo 'cd /sdcard/my-diary' >> ~/sync-diary.sh
echo 'git add .' >> ~/sync-diary.sh
echo 'git commit -m "mobile sync"' >> ~/sync-diary.sh
echo 'git push' >> ~/sync-diary.sh
echo 'git pull' >> ~/sync-diary.sh
chmod +x ~/sync-diary.sh
```

以后手机同步只需打开 Termux 运行：

```bash
~/sync-diary.sh
```

同样方式创建了 `~/sync-marxism.sh` 用于同步马克思笔记仓库。

---

## 第六关：电脑端和手机端冲突

手机配置完推送后，电脑端 `git pull` 报错：

```
fatal: refusing to merge unrelated histories
```

**解决**：

```bash
git pull --allow-unrelated-histories
```

出现冲突，用远程版本覆盖：

```bash
git checkout --theirs .
git add .
git commit -m "merge: accept remote changes"
git push
```

冲突解决，双端同步正常。

---

## 最终成果

|仓库|类型|用途|
|---|---|---|
|`my-diary`|私有|个人日记|
|`marxism-works`|公开|马克思著作阅读笔记和文章，中英双语|
|`HelloWorld`|公开|本文|

**同步方式总结：**

|场景|操作|
|---|---|
|电脑写完|Obsidian Git 插件每10分钟自动同步|
|手机写完日记|Termux 运行 `~/sync-diary.sh`|
|手机写完笔记|Termux 运行 `~/sync-marxism.sh`|

---

## 感想

没有 AI 的话这些东西可能要折腾两三天。即便有 AI 陪着也折腾了整整一个下午加晚上。

但折腾本身就是学习。今天学到了：Git 基本操作、Termux 使用、代理配置、仓库同步原理、Markdown 写作（其实没学到这个）。

第一篇日记写的是：

> 我们正在向未来逼近，我的每一步都是无产阶级坚实的一步，我们一定能取得胜利。

---

✊ 全世界无产者，联合起来！
