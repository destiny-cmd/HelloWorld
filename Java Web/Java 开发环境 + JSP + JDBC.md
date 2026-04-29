# Java 开发环境 + JSP + JDBC

---

## Java 开发环境配置

### JDK 是什么？

JDK 全称 **Java Development Kit**（Java 开发工具包），是开发和运行 Java 程序的必备环境。没有 JDK，Eclipse 无法启动，Java 代码也无法编译运行。

JDK 里包含：

- **JVM**（Java 虚拟机）：负责运行 Java 程序
- **编译器（javac）**：把 `.java` 源代码编译成 `.class` 字节码
- **各种工具**：调试器、打包工具等

### 为什么选 JDK 21？

Eclipse 的配置文件里要求：

```
-DosgiRequiredJavaVersion=21
```

说明这个版本的 Eclipse 需要 Java 21 才能运行。用 JDK 25 会报错：

> Project facet Java version 25 is not supported.

JDK 21 是目前的 **LTS（长期支持版）**，稳定可靠，与 Eclipse 和 Tomcat 9 兼容性最好。

### Eclipse 启动失败的排查过程

**症状：**

```
Java was started but returned exit code=-1073740791
```

**第一步：检查 Java 环境**

```powershell
java -version
```

报错：`java` 不是可识别的命令，说明环境变量丢失。

**第二步：找到 Java 路径**

```powershell
Get-Command java | Select-Object -ExpandProperty Source
# 输出：C:\Program Files\Common Files\Oracle\Java\javapath\java.exe
```

这个路径是个跳板文件，不是真正的 JDK。

**第三步：寻找真正的 JDK**

```powershell
Get-ChildItem "C:\Program Files\Java"
# 发现：C:\Program Files\Java\latest\jdk-25
```

但 `jdk-25\bin\java.exe` 不存在，说明 JDK 安装损坏了。

**第四步：重新安装 JDK 21**

使用 **Microsoft Build of OpenJDK**，下载地址：

```
https://aka.ms/download-jdk/microsoft-jdk-21-windows-x64.msi
```

为什么选微软的 OpenJDK？

- `.msi` 安装包，双击一路下一步
- **自动配置环境变量**，不需要手动设置 JAVA_HOME 和 PATH
- 比 Oracle JDK 省心很多

安装完验证：

```powershell
java -version
# 输出：openjdk version "21.0.10" 2026-01-20 LTS
```

**在 Linux 上安装 JDK（对比参考）：**

```bash
# Ubuntu/Debian
sudo apt install openjdk-21-jdk

# CentOS/RHEL
sudo yum install java-21-openjdk-devel
```

Linux 上包管理器更方便，一行命令搞定，还自动处理环境变量。

---

## Eclipse 项目配置要点

### Tomcat 9 vs Tomcat 10 的重要区别

这是最容易踩的坑：

||Tomcat 9|Tomcat 10+|
|---|---|---|
|Servlet 包名|`javax.servlet.*`|`jakarta.servlet.*`|
|混用后果|编译报错，找不到类|

**错误示例（用了 jakarta 但 Tomcat 是 9.0）：**

```
The import jakarta cannot be resolved
HttpServlet cannot be resolved to a type
```

**正确写法（Tomcat 9）：**

```java
import javax.servlet.*;
import javax.servlet.http.*;
import javax.servlet.annotation.*;
```

### Servlet 文件放错位置的问题

Servlet 的 `.java` 文件必须放在：

```
src/main/java/
```

不能放在：

```
WEB-INF/classes/  ← 错误！会变成 _WEB-INF_classes_TestServlet.java
```

放错了 Tomcat 无法识别，访问会报 404。

### Project Facets 配置

右键项目 → Properties → Project Facets：

- **Java** 版本必须设为 **21**（与安装的 JDK 一致）
- **Dynamic Web Module** 版本 4.0
- **Runtimes** 标签里必须关联 Tomcat

### Java Build Path 配置

右键项目 → Properties → Java Build Path → Libraries：

- Classpath 下需要有 `mysql-connector-j-9.5.0.jar`
- 需要有 `Server Runtime [Apache Tomcat v9.0]`

---

## JDBC 详解

### JDBC 是什么？

**JDBC = Java Database Connectivity**（Java 数据库连接）

它是 Java 官方提供的一套**标准接口**，专门让 Java 程序连接和操作各种数据库。

**用比喻理解：**

- MySQL 说"MySQL 语"
- Oracle 说"Oracle 语"
- SQLite 说"SQLite 语"

JDBC 是一套**万能翻译标准**，规定了翻译的格式。各数据库厂商提供自己的**翻译官**（jar 包），按照 JDBC 标准实现。

所以代码只需按 JDBC 标准写，换数据库时只需换 jar 包，代码基本不用改。

### MySQL Connector/J 是什么？

`mysql-connector-j-9.5.0.jar` 就是 MySQL 版本的 JDBC 驱动，即"翻译官"。

没有它会报错：

```
No suitable driver found for jdbc:mysql://localhost:3306/...
```

意思是"找不到翻译官，没法和 MySQL 通信"。

**官方下载地址：**

```
https://dev.mysql.com/downloads/connector/j/
```

选 Platform Independent → 下载 ZIP → 解压取出 jar 文件。

### jar 包放在哪里？

**放在项目 WEB-INF/lib：**

- 只有该项目能用
- 有时在 JSP 中仍会报 `No suitable driver found`

**放在 Tomcat 的 lib 目录（推荐）：**

```
C:\Do\tomcat\lib\mysql-connector-j-9.5.0.jar
```

- Tomcat 启动时自动加载
- 所有项目都能用
- 不会出现驱动找不到的问题
- **放完必须重启 Tomcat 才能生效！**

### JDBC 连接模板

```java
String url = "jdbc:mysql://localhost:3306/数据库名?useSSL=false&serverTimezone=UTC";
String user = "root";
String password = "你的密码";
Connection conn = DriverManager.getConnection(url, user, password);
```

URL 参数说明：

- `localhost:3306`：MySQL 默认在本机 3306 端口
- `useSSL=false`：不使用 SSL 加密（本地开发用）
- `serverTimezone=UTC`：指定时区，避免时间相关报错

### JDBC 增删改查完整示例

```jsp
<%@ page contentType="text/html;charset=UTF-8" %>
<%@ page import="java.sql.*" %>
<html>
<body>
<%
    String url = "jdbc:mysql://localhost:3306/studentdb?useSSL=false&serverTimezone=UTC";
    String user = "root";
    String password = "1314521";
    Connection conn = DriverManager.getConnection(url, user, password);

    // 插入（Insert）
    PreparedStatement ps = conn.prepareStatement(
        "INSERT INTO student VALUES (null, '赵六', 22, 95.0)");
    ps.executeUpdate();
    out.println("插入成功！<br>");

    // 更新（Update）
    ps = conn.prepareStatement(
        "UPDATE student SET score=100 WHERE name='张三'");
    ps.executeUpdate();
    out.println("更新成功！<br>");

    // 删除（Delete）
    ps = conn.prepareStatement(
        "DELETE FROM student WHERE name='赵六'");
    ps.executeUpdate();
    out.println("删除成功！<br>");

    // 查询（Select）
    ResultSet rs = conn.prepareStatement(
        "SELECT * FROM student").executeQuery();
    out.println("<br>查询结果：<br>");
    while(rs.next()){
        out.println("ID:" + rs.getInt("id") + 
                   " 姓名:" + rs.getString("name") + 
                   " 年龄:" + rs.getInt("age") + 
                   " 成绩:" + rs.getDouble("score") + "<br>");
    }
    conn.close();
%>
</body>
</html>
```

---

## MySQL 常用操作

### 基础 SQL

```sql
-- 建库
CREATE DATABASE IF NOT EXISTS studentdb;
USE studentdb;

-- 建表
CREATE TABLE student (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50),
    age INT,
    score DOUBLE
);

-- 插入数据
INSERT INTO student VALUES (null, '张三', 20, 85.5);
INSERT INTO student VALUES (null, '李四', 21, 90.0);
INSERT INTO student VALUES (null, '王五', 19, 78.5);

-- 查询
SELECT * FROM student;

-- 更新
UPDATE student SET score=100 WHERE name='张三';

-- 删除
DELETE FROM student WHERE name='张三';
```

### MySQL 服务管理

```powershell
# 查找服务名
Get-Service | Where-Object {$_.DisplayName -like "*mysql*"}

# 启动/停止
net start MySQL80
net stop MySQL80
```

### 忘记 MySQL 密码重置步骤

```powershell
# 第一步：停止服务
net stop MySQL80

# 第二步：跳过权限验证启动（此窗口会卡住，不要关）
mysqld --skip-grant-tables --shared-memory

# 第三步：新开一个 PowerShell 窗口连接
mysql -u root

# 第四步：重置密码
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED BY '新密码';

# 第五步：退出并重启
exit
net start MySQL80
```

---

## JSP 开发详解

### JSP vs JavaScript 的本质区别

这是初学者最容易混淆的地方，虽然名字都有"Java"，但完全不同：

||JSP|JavaScript|
|---|---|---|
|运行位置|**服务器端**（Tomcat）|**浏览器端**|
|执行者|Tomcat|浏览器|
|主要用途|处理逻辑、连数据库、生成 HTML|页面交互、动画、表单验证|
|用户可见|不可见，只看到生成的 HTML|可在浏览器查看源码|
|关系|无直接关系，名字相似纯属巧合||

**用餐厅比喻：**

- JSP = 后厨（做菜，处理数据）
- JavaScript = 服务员（和顾客互动）
- HTML = 菜单/餐盘（展示内容）

### JSP 基本语法

```jsp
<%-- 导入包 --%>
<%@ page contentType="text/html;charset=UTF-8" %>
<%@ page import="java.sql.*" %>

<%-- Java 代码块 --%>
<% 
    String name = "张三";
    int age = 20;
%>

<%-- 输出变量 --%>
<p>姓名：<%= name %></p>

<%-- 获取表单数据 --%>
<%
    request.setCharacterEncoding("UTF-8"); // 防止中文乱码，必须加
    String username = request.getParameter("username");      // 单个值
    String[] hobbies = request.getParameterValues("hobby"); // 多个同名值
%>
```

### 实验3：四个 JSP 练习

**第一题：登录表单**

`login.jsp`（表单页）：

```jsp
<%@ page contentType="text/html;charset=UTF-8" %>
<html>
<body>
<form action="loginResult.jsp" method="post">
    账号：<input type="text" name="username"><br>
    密码：<input type="password" name="password"><br>
    <input type="submit" value="登录">
</form>
</body>
</html>
```

`loginResult.jsp`（结果页）：

```jsp
<%@ page contentType="text/html;charset=UTF-8" %>
<html>
<body>
<%
    request.setCharacterEncoding("UTF-8");
    String username = request.getParameter("username");
    String password = request.getParameter("password");
    if("admin".equals(username) && "123456".equals(password)){
        out.println("登录成功");
    } else {
        out.println("登录失败");
    }
%>
</body>
</html>
```

---

**第二题：打印 N 个"欢迎"**

`welcome.jsp`：

```jsp
<%@ page contentType="text/html;charset=UTF-8" %>
<html>
<body>
<form action="welcomeResult.jsp" method="post">
    请输入数字N：<input type="text" name="n"><br>
    <input type="submit" value="提交">
</form>
</body>
</html>
```

`welcomeResult.jsp`：

```jsp
<%@ page contentType="text/html;charset=UTF-8" %>
<html>
<body>
<%
    int n = Integer.parseInt(request.getParameter("n"));
    for(int i = 0; i < n; i++){
        out.println("欢迎<br>");
    }
%>
</body>
</html>
```

---

**第三题：三页面登录 + 姓名传递**

核心知识点：用 `hidden` 隐藏表单在页面间传递数据。

`page1.jsp`：

```jsp
<%@ page contentType="text/html;charset=UTF-8" %>
<html>
<body>
<form action="page2.jsp" method="post">
    账号：<input type="text" name="username"><br>
    密码：<input type="password" name="password"><br>
    <input type="submit" value="登录">
</form>
</body>
</html>
```

`page2.jsp`：

```jsp
<%@ page contentType="text/html;charset=UTF-8" %>
<html>
<body>
<%
    request.setCharacterEncoding("UTF-8");
    String username = request.getParameter("username");
    String password = request.getParameter("password");
    if("admin".equals(username) && "123456".equals(password)){
%>
    <p>登录成功！请输入姓名：</p>
    <form action="page3.jsp" method="post">
        <%-- 用隐藏表单把账号传到下一页 --%>
        <input type="hidden" name="username" value="<%=username%>">
        姓名：<input type="text" name="realname"><br>
        <input type="submit" value="提交">
    </form>
<%
    } else {
        out.println("登录失败，请返回重试");
    }
%>
</body>
</html>
```

`page3.jsp`：

```jsp
<%@ page contentType="text/html;charset=UTF-8" %>
<html>
<body>
<%
    request.setCharacterEncoding("UTF-8");
    String username = request.getParameter("username");
    String realname = request.getParameter("realname");
%>
<p>账号：<%=username%></p>
<p>姓名：<%=realname%></p>
</body>
</html>
```

---

**第四题：同名表单元素 + 隐藏表单**

核心知识点：复选框多个同名元素用 `getParameterValues()` 获取数组。

`form1.jsp`：

```jsp
<%@ page contentType="text/html;charset=UTF-8" %>
<html>
<body>
<form action="form2.jsp" method="post">
    爱好（多选）：<br>
    <input type="checkbox" name="hobby" value="篮球">篮球
    <input type="checkbox" name="hobby" value="足球">足球
    <input type="checkbox" name="hobby" value="游泳">游泳
    <input type="checkbox" name="hobby" value="跑步">跑步<br><br>
    <%-- 隐藏表单，用户不可见但会随表单提交 --%>
    <input type="hidden" name="from" value="form1页面">
    <input type="submit" value="提交">
</form>
</body>
</html>
```

`form2.jsp`：

```jsp
<%@ page contentType="text/html;charset=UTF-8" %>
<html>
<body>
<%
    request.setCharacterEncoding("UTF-8");
    String[] hobbies = request.getParameterValues("hobby");
    String from = request.getParameter("from");
%>
<p>来自：<%=from%></p>
<p>选择的爱好：</p>
<%
    if(hobbies != null){
        for(String h : hobbies){
            out.println(h + "<br>");
        }
    } else {
        out.println("未选择任何爱好");
    }
%>
</body>
</html>
```

---

### 常见报错汇总

|报错|原因|解决方法|
|---|---|---|
|`No suitable driver found`|MySQL 驱动未加载|把 jar 放到 Tomcat/lib，重启 Tomcat|
|`The import jakarta cannot be resolved`|Tomcat 9 用了 jakarta 包|改成 `javax.servlet.*`|
|`HTTP 404`|文件路径错误或 Servlet 未部署|检查文件位置，重启 Tomcat|
|`0xc0000005 Access Violation`|程序访问了无权限的内存|系统文件损坏或版本不匹配|
|`Project facet Java version 25 is not supported`|项目 Java 版本设置错误|Project Facets 改为 Java 21|
|`gpedit.msc 找不到`|Windows 家庭版没有组策略编辑器|用注册表替代|

---

### 推荐技术社区

|平台|特点|
|---|---|
|Stack Overflow|全球最大编程问答，遇到报错直接搜|
|掘金（juejin.cn）|国内质量最高，内容新|
|博客园（cnblogs.com）|老牌社区，Java 内容丰富|
|V2EX（v2ex.com）|程序员聚集地，讨论氛围好|
|GitHub Issues|用某开源项目遇到问题直接去搜|

**建议：** 遇到报错优先搜 Stack Overflow，中文问题搜掘金，避免用 CSDN（内容质量参差不齐，很多是复制粘贴）。
