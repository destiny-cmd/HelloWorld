从 Git 到 GitHub 到 Obsidian+vscode 完整配置指南
---

> 本文是真实配置过程的记录，包含踩过的坑和解决方法，适合零基础的同学参考(其实我才是零基础的零基础）。

---
## 准备工作

- 注册 [GitHub](https://github.com/) 账号
- 安装 [Obsidian](https://obsidian.md/)
- 一个代理工具（国内访问 GitHub 需要，本文使用 FLClash）

---

## 一、安装 Git（Windows）

### 下载安装

去 [git-scm.com](https://git-scm.com/download/win) 下载安装包。

> ⚠️ **坑1**：如果之前装过旧版 Git，必须先卸载再重装，否则 PATH 可能写入失败，PowerShell 里找不到 `git` 命令。
> （这里其实我装第一遍有问题到了第二遍才好的）（我也不知道为什么见下文我其实安装第二遍只动了一个地方，第一遍其实什么都没动）
> 卸载方法：控制面板 → 程序 → 卸载程序 → 找到 Git → 卸载

### 安装关键步骤

安装过程中大部分*默认*（这是真的）（我第一遍就这么操纵的结果。。。） 
`PS C:\Users\z'y'j> git --version git : The term 'git' is not recognized as the name of a cmdlet, function, script file, or operable program. Check the s pelling of the name, or if a path was included, verify that the path is correct and try again. At line:1 char:1 + git --version + ~~~ + CategoryInfo : ObjectNotFound: (git:String) [], CommandNotFoundException + FullyQualifiedErrorId : CommandNotFoundException`
（当时就这么报错的）
就好，以下几步需要注意：

**默认编辑器**：推荐选 `Notepad++` （我之前心血来潮要弄钢丝mod就装了这么一个，结果弄拙成巧）或 `VSCode`，不建议用默认的 Vim（难用）（据说难用）

**默认分支名**：选 `Override the default branch name`，填 `main`

> 因为 GitHub 现在默认分支是 main，统一比较好

**PATH 设置**：选第二个 `Git from the command line and also from 3rd-party software`

> ⚠️ **坑2**：这一步如果选错，PowerShell 里就找不到 git 命令

其余步骤全部默认，直接 Next。

### 验证安装

重新打开 PowerShell 输入：

```bash
git --version
```

显示版本号即成功，例如 `git version 2.54.0.windows.1`

---

## 二、在 GitHub 创建仓库

1. 打开 [github.com](https://github.com/)，登录后点右上角 **+** → **New repository**
2. 填写仓库名（如 `my-diary`）
3. 选择 **Private**（私有）或 **Public**（公开）
4. 勾选 **Add a README file**
5. 点 **Create repository**

---

## 三、电脑端配置

### 1. 配置 Git 身份

```bash
git config --global user.name "你的用户名"
git config --global user.email "你的邮箱"
```

### 2. 设置代理

> ⚠️ **坑3**：国内直接 clone GitHub 会报 `Connection was reset`，必须挂代理。

FLClash 用户（端口 7890）：

```bash
git config --global http.proxy http://127.0.0.1:7890
git config --global https.proxy http://127.0.0.1:7890
```

确保 FLClash 开启了系统代理模式。
之后执行类似这样的
（git clone https://github.com/destiny-cmd/my-diary.git）

### 3. 克隆仓库到本地

```bash
cd "C:\FOR WORKING"
git clone https://github.com/用户名/仓库名.git
```

> 第一次 clone 会弹出 GitHub 登录窗口，点 **Sign in with your browser** 完成认证即可。

### 4. Obsidian 打开仓库

1. 打开 Obsidian
2. 点**打开本地仓库**（不是创建！）
3. 找到克隆好的文件夹，点**选择文件夹**

### 5. 安装 Obsidian Git 插件

1. 设置 → 第三方插件 → 关闭安全模式 → 浏览
2. 搜索 `git`，安装作者 **Vinzent** 的 Git 插件（下载量240万）
3. 启用后进入插件设置：
    - `Auto commit-and-sync interval`：填 `10`（每10分钟自动备份）
    - `Auto pull interval`：填 `10`（每10分钟自动拉取）

至此电脑端配置完成，日记会每10分钟自动同步到 GitHub。

---

## 四、手机端配置（Android）

### 1. 安装新版 Termux

> ⚠️ **坑4**：不要从 Google Play 安装 Termux，版本太旧，密钥过期无法安装软件包。

从 [GitHub Releases](https://github.com/termux/termux-app/releases/latest) 下载：

选择 `termux-app_vX.X.X+apt-android-7-github-debug_arm64-v8a.apk`（绝大多数手机选 arm64）

### 2. 安装 Git

```bash
pkg update -y && pkg install git -y
```

### 3. 配置 Git 身份

```bash
git config --global user.name "你的用户名"
git config --global user.email "你的邮箱"
```

### 4. 授权存储权限

```bash
termux-setup-storage
```

弹出权限请求，点允许。

### 5. 生成 GitHub Token

> ⚠️ **坑5**：手机端 Git 无法用浏览器登录 GitHub，需要用 Token 认证。

1. 电脑打开 GitHub → 右上角头像 → **Settings**
2. 左侧最底部 **Developer settings**
3. **Personal access tokens** → **Tokens (classic)**
4. **Generate new token (classic)**
5. 勾选 **repo** 权限
6. 点 **Generate token**，复制保存（只显示一次！）

### 6. 克隆仓库

```bash
git config --global --add safe.directory /storage/emulated/0/仓库名
cd /sdcard
git clone https://用户名:TOKEN@github.com/用户名/仓库名.git
```

把 `TOKEN` 替换成刚才生成的 token。

### 7. 配置远程同步

```bash
cd /sdcard/仓库名
git config remote.origin.fetch "+refs/heads/*:refs/remotes/origin/*"
git fetch origin
git checkout main
```

### 8. Obsidian 打开仓库

在 Obsidian 中选择 `/sdcard/仓库名` 文件夹作为库。

### 9. 创建同步脚本

```bash
echo '#!/bin/bash' > ~/sync.sh
echo 'cd /sdcard/仓库名' >> ~/sync.sh
echo 'git add .' >> ~/sync.sh
echo 'git commit -m "mobile sync"' >> ~/sync.sh
echo 'git push' >> ~/sync.sh
echo 'git pull' >> ~/sync.sh
chmod +x ~/sync.sh
```

> ⚠️ **坑6**：手机端 Obsidian Git 插件对 Android 支持不稳定，推荐直接用 Termux 脚本同步。

以后手机同步只需打开 Termux 运行：

```bash
~/sync.sh
```
### 五、用 VSCode + Git 提交代码文件

> 适用于需要同时管理 `.md` 文章、`.py` 代码、图片等多种文件类型的仓库。

#### 1. 安装 VSCode

去 [code.visualstudio.com](https://code.visualstudio.com) 下载安装，安装时勾选 **Add to PATH**。

推荐安装两个插件：

- **Chinese (Simplified)**：界面汉化
- **Markdown Preview Enhanced**：实时预览 `.md` 文件

#### 2. 打开仓库

File → Open Folder → 选择本地仓库文件夹。

左侧源代码管理图标会显示 git 已识别仓库。

#### 3. 安装 Python 依赖

在 VSCode 终端（Ctrl+`）输入：

bash

```bash
pip install matplotlib pdfplumber
```

#### 4. 日常提交流程

bash

```bash
git add .
git commit -m "简洁描述这次改了什么"
git push
```

#### 5. 压缩历史 commit

如果前几次提交 message 比较乱，可以压缩成一条：

bash

```bash
git reset --soft 最早那条commit的hash
git add .
git commit -m "init: 描述"
git push --force
```

> ⚠️ **坑7**：`git push --force` 会覆盖远程历史，个人仓库可用，多人协作仓库禁止使用。

> ⚠️ **坑8**：做 `git rebase` 操作前务必关闭 Obsidian，否则 Obsidian 后台自动修改的 `workspace.json` 会导致 rebase 冲突无法继续也无法中止。

#### 6. commit message 写作建议

|场景|示例|
|---|---|
|初次提交|`init: add article and code`|
|修改文章|`update: 修订第四节措辞`|
|修复代码|`fix: 修正年份提取正则`|

---

## 总结

| 操作       | 方式                                                                                                           |
| -------- | ------------------------------------------------------------------------------------------------------------ |
| 电脑写完自动同步 | Obsidian Git 插件，每10分钟自动执行<br>(可以在 Obsidian 里用命令面板手动触发：`Ctrl+P` → 搜索 `git` → 点 **Git: Commit and sync** 立刻同步) |
| 手机写完同步   | 打开 Termux，运行 `~/sync.sh`                                                                                     |
| 拉取最新内容   | 脚本里的 `git pull` 自动处理                                                                                         |
| vscode提交 | 在vscode终端运行：<br>`git add .`<br>`git commit -m "简洁描述这次改了什么"`<br>`git push`                                    |

---
全世界无产者，联合起来！✊
