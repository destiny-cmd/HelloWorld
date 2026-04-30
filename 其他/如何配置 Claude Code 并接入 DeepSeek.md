# 如何配置 Claude Code 并接入 DeepSeek

## 前置要求

- Windows 10/11
- 稳定的网络环境（需要代理工具，如 FLClash）

---

## 一、安装 Node.js

参考视频：[Nodejs安装零基础教程2025](https://www.bilibili.com/video/BV1sbjgzwEBX/?spm_id_from=333.1391.0.0&vd_source=a8d952fd2984acfe359946bd4ac6965c)

1. 访问 [https://nodejs.org](https://nodejs.org/)，下载 **LTS 版本**
2. 一路默认安装
3. 安装完成后打开 PowerShell，验证：

```powershell
node -v
npm -v
```

能输出版本号说明安装成功。

> 推荐使用 Microsoft Build of OpenJDK 21 作为 Java 环境（如果你同时需要 Java 开发）： 下载地址：https://aka.ms/download-jdk/microsoft-jdk-21-windows-x64.msi

---

## 二、安装 Claude Code

```powershell
npm install -g @anthropic-ai/claude-code
```

验证安装：

```powershell
claude --version
```

---

## 三、配置代理（FLClash）

Claude Code 需要访问 `api.anthropic.com`，国内需要代理。

### 3.1 确认代理端口

打开 Windows **Internet 选项** → **连接** → **局域网设置**，查看代理端口（一般是 `7890`）。

### 3.2 设置环境变量

```powershell
$env:HTTPS_PROXY = "http://127.0.0.1:7890"
$env:HTTP_PROXY = "http://127.0.0.1:7890"
```

### 3.3 代理模式

如果规则模式无法连接，在 FLClash 仪表盘将**出站模式**改为**全局**。

如需规则模式下也能连接，在配置文件 `rules:` 最前面添加：

```yaml
rules:
    - 'DOMAIN-SUFFIX,anthropic.com,🔰 节点选择'
    - 'DOMAIN-SUFFIX,claude.ai,🔰 节点选择'
    # ... 其余规则
```

> 注意：如果配置来自订阅链接，更新订阅会覆盖手动修改，建议使用 FLClash 的**覆写（Override）**功能添加规则。

---

## 四、接入 DeepSeek

### 4.1 获取 DeepSeek API Key

访问 [https://platform.deepseek.com](https://platform.deepseek.com/)，注册并创建 API Key。

### 4.2 修改 Claude Code 配置文件

配置文件路径：

```
C:\Users\你的用户名\AppData\Roaming\Claude\settings.json
```

将内容修改为：

```jsonc
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://api.deepseek.com/anthropic",
    "ANTHROPIC_AUTH_TOKEN": "你的DeepSeek API Key",
    "ANTHROPIC_MODEL": "deepseek-v4-pro[1m]",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "deepseek-v4-pro[1m]",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "deepseek-v4-pro[1m]",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "deepseek-v4-flash",
    "CLAUDE_CODE_SUBAGENT_MODEL": "deepseek-v4-flash",
    "CLAUDE_CODE_EFFORT_LEVEL": "max"
  },
  "theme": "auto"
}
```

> ⚠️ 注意 JSON 格式：`env` 对象结束的 `}` 后面必须加逗号 `,`，否则配置不生效。
> （这句话非常重要！！！！！）

### 4.3 重启 Claude Code

保存文件后重新运行：

```powershell
claude
```

右上角显示 **DeepSeek 模型名称** 而非 `Sonnet 4.6` 说明配置成功。

---

## 常见问题

|问题|原因|解决方法|
|---|---|---|
|`ECONNREFUSED`|代理端口错误|检查 LAN 设置中的实际端口|
|仍显示 Sonnet 4.6|JSON 格式错误|检查 `}` 后是否有逗号|
|规则模式连不上|规则未匹配 anthropic.com|改为全局模式或手动添加规则|
|`/doctor` 报错|settings.json 格式问题|用 JSON 校验工具检查格式|

---

## 参考资料

- [Node.js 官网](https://nodejs.org/)
- [Node.js 安装教程（B站）](https://www.bilibili.com/video/BV1sbjgzwEBX/)
- [Claude Code 文档](https://docs.anthropic.com/claude-code)
- [DeepSeek 开放平台](https://platform.deepseek.com/)