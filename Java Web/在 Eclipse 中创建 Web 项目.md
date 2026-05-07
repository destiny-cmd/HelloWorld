### 在 Eclipse 中创建 Web 项目

#### 前提条件

确保 Eclipse 安装了 **Eclipse IDE for Enterprise Java and Web Developers** 版本，普通 Java 版没有 Web 支持。

---

#### 步骤一：新建动态 Web 项目

1. 菜单栏 **File → New → Dynamic Web Project**
    - 若没看到这个选项，点 **Other…** 搜索 "Dynamic Web"
2. 填写项目信息：
    - **Project name**：填写项目名称
    - **Target runtime**：选择 Tomcat（见步骤二）
    - **Dynamic web module version**：推荐选 **4.0**（对应 Jakarta EE）
3. 点 **Next → Next**，勾选 **Generate web.xml deployment descriptor**
4. 点 **Finish**

---

#### 步骤二：配置 Tomcat（如果没有）

1. **Window → Preferences → Server → Runtime Environments → Add**
2. 选择对应的 **Apache Tomcat** 版本（如 Tomcat 10）
3. 点 **Browse** 指向本地 Tomcat 安装目录
4. **Finish**

> 没有 Tomcat 的话先去 [https://tomcat.apache.org](https://tomcat.apache.org) 下载解压即可。

---

#### 步骤三：项目结构说明

```
项目名/
├── src/main/java/          # Java 源代码（Servlet等）
├── src/main/webapp/        # Web资源
│   ├── WEB-INF/
│   │   └── web.xml         # 部署描述文件
│   ├── index.jsp           # 首页
│   └── css / js / img/     # 静态资源
```

---

#### 步骤四：创建第一个 JSP 页面

1. 右键 **webapp** 文件夹 → **New → JSP File**
2. 文件名填 `index.jsp` → Finish
3. 写入简单内容：

jsp

```jsp
<%@ page contentType="text/html; charset=UTF-8" %>
<html>
<body>
    <h2>Hello, Web!</h2>
</body>
</html>
```

---

#### 步骤五：运行项目

1. 右键项目 → **Run As → Run on Server**
2. 选择之前配置的 Tomcat → **Finish**
3. Eclipse 会自动启动 Tomcat，浏览器访问：

```
   http://localhost:8080/项目名/index.jsp
```

---

>You said: Runtime "Apache Tomcat v9.
   Runtime "Apache Tomcat v9.0" is invalid. Unknown version of Tomcat was specified. 
   这是什么意思

答：
Claude responded: 这个错误说明 Eclipse 找到了 Tomcat 目录，但无法识别其版本，通常手动点击tomcat再配置一下就好了

记住，运行最好直接右键点击项目Run As