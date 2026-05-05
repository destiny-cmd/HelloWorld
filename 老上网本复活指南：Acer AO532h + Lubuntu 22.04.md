# 老上网本复活指南：Acer AO532h + Lubuntu 22.04

> 机型：Acer AO532h | CPU：Intel Atom N450 @ 1.66GHz | 内存：2GB DDR2 | 原系统：Windows XP

## 〇、准备工作

1. **U 盘** — 4GB 以上
2. **主力机** — 用于下载 ISO 和制作启动盘
3. **心态** — 老机器够用就好，别指望跑大型软件

---

## 一、制作 Lubuntu 22.04 启动盘

1. 下载 Lubuntu 22.04 LTS ISO（清华镜像）：  
    `https://mirrors.tuna.tsinghua.edu.cn/ubuntu-cdimage/lubuntu/releases/jammy/release/lubuntu-22.04.5-desktop-amd64.iso`
    
2. 下载 Rufus：`https://rufus.ie/` — 选 `rufus-4.14.exe`（标准版）
    
3. 打开 Rufus：
    
    - 设备 → 选你的 U 盘
    - 引导类型 → 选下载好的 ISO
    - 分区类型 → **MBR**（老机器是传统 BIOS）
    - 点 **开始** → 等写完

---

## 二、安装 Lubuntu

1. U 盘插到 AO532h，开机狂按 **F2** 进 BIOS
2. Boot 选项卡 → USB 调到第一启动 → F10 保存退出
3. 重启进入 Lubuntu 菜单 → 选 **Try Lubuntu**（先试用一下）
4. 双击桌面 **Install Lubuntu**
5. 安装类型选 **「抹除磁盘并安装 Lubuntu」**（会清空 XP 和所有数据）
6. 设置用户名密码 → 等 20-30 分钟 → 重启 → 拔掉 U 盘

---

## 三、基础开发环境一条命令装齐

先连上 Wi-Fi（右下角网络图标），打开终端（Ctrl + Alt + T），然后：

```bash
sudo apt update
sudo apt install openjdk-17-jdk tomcat9 mariadb-server gcc python3-pip geany -y
```

一行装好：

|包名|用途|
|---|---|
|`openjdk-17-jdk`|Java 开发环境|
|`tomcat9`|Java Web 服务器（Ubuntu 22.04 只有 Tomcat 9）|
|`mariadb-server`|数据库（MySQL 兼容）|
|`gcc`|C 语言编译器|
|`python3-pip`|Python 包管理器|
|`geany`|轻量级 IDE（N450 跑不动 VS Code/Eclipse）|

---

## 四、修复 Tomcat 找不到 JDK 的问题

⚠️ Lubuntu 22.04 的 Tomcat 9 启动脚本只认 JDK 8~11，不认 JDK 17，需要手动修。

### 1. 编辑 Tomcat 配置文件

```bash
sudo nano /etc/default/tomcat9
```

确认这行存在：

```
JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
```

### 2. 修复 Java 查找脚本

```bash
sudo nano /usr/libexec/tomcat9/tomcat-locate-java.sh
```

找到：

```sh
for java_version in 11 10 9 8
```

改成：

```sh
for java_version in 17 11 10 9 8
```

Ctrl+O → 回车保存 → Ctrl+X 退出

### 3. 重启 Tomcat

```bash
sudo systemctl daemon-reload
sudo systemctl restart tomcat9
sudo systemctl status tomcat9    # 看到 active (running) 就对了
```

> **按 q** 退出 status 的实时监控界面

---

## 五、常用 systemctl 命令速查

|操作|命令|
|---|---|
|启动服务|`sudo systemctl start tomcat9`|
|停止服务|`sudo systemctl stop tomcat9`|
|重启服务|`sudo systemctl restart tomcat9`|
|查看状态|`sudo systemctl status tomcat9`|
|开机自启|`sudo systemctl enable tomcat9`|
|禁止自启|`sudo systemctl disable tomcat9`|

---

## 六、数据库验证

```bash
# 检查 MariaDB 是否运行
sudo systemctl status mariadb

# 登录数据库
sudo mysql -u root

# 退出数据库命令行
exit;
```

---

## 七、MariaDB 数据库使用指南

### 7.1 启动与登录

```bash
# 检查数据库是否在跑
sudo systemctl status mariadb

# 没跑就启动
sudo systemctl start mariadb

# 登录数据库（初次无需密码）
sudo mysql -u root

# 退出数据库
exit;
```

### 7.2 创建数据库和用户

```sql
-- 登录后执行（提示符变成 MariaDB [(none)]> ）
CREATE DATABASE mydb;
CREATE USER 'myuser'@'localhost' IDENTIFIED BY '123456';
GRANT ALL PRIVILEGES ON mydb.* TO 'myuser'@'localhost';
FLUSH PRIVILEGES;
USE mydb;
```

### 7.3 建表和 CRUD 操作

```sql
-- 建表
CREATE TABLE students (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) NOT NULL,
    age INT,
    grade VARCHAR(10)
);

-- 插入数据
INSERT INTO students (name, age, grade) VALUES ('张三', 20, 'A');
INSERT INTO students (name, age, grade) VALUES ('李四', 21, 'B');
INSERT INTO students (name, age, grade) VALUES ('王五', 19, 'A');

-- 查询
SELECT * FROM students;
SELECT * FROM students WHERE grade = 'A';

-- 更新
UPDATE students SET age = 22 WHERE name = '张三';

-- 删除
DELETE FROM students WHERE name = '王五';

-- 查看表结构
DESC students;
```

### 7.4 用 Java 连接数据库（JDBC）

先在终端安装 JDBC 驱动：

```bash
sudo apt install libmariadb-java -y
```

驱动 jar 位置：`/usr/share/java/mariadb-java-client.jar`

### 7.5 在 Geany 里写 Java + 数据库的项目

#### 第一步：创建项目目录

```bash
mkdir ~/java-db-demo && cd ~/java-db-demo
```

#### 第二步：打开 Geany，写一个 Java 类

左下角开始菜单 → 搜 **Geany** → 打开

点 **新建** → 写下面的代码 → 保存为 `StudentDAO.java`：

```java
import java.sql.*;

public class StudentDAO {
    public static void main(String[] args) {
        String url = "jdbc:mariadb://localhost:3306/mydb";
        String user = "myuser";
        String password = "123456";

        try {
            // 加载驱动
            Class.forName("org.mariadb.jdbc.Driver");

            // 连接数据库
            Connection conn = DriverManager.getConnection(url, user, password);
            System.out.println("数据库连接成功！");

            // 查询
            Statement stmt = conn.createStatement();
            ResultSet rs = stmt.executeQuery("SELECT * FROM students");

            while (rs.next()) {
                int id = rs.getInt("id");
                String name = rs.getString("name");
                int age = rs.getInt("age");
                String grade = rs.getString("grade");
                System.out.println(id + " | " + name + " | " + age + " | " + grade);
            }

            rs.close();
            stmt.close();
            conn.close();

        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

#### 第三步：在 Geany 里编译运行

1. Geany 菜单栏 → **生成** → **设置生成命令**
2. 在 **Compile** 那行填：

```
javac -cp .:/usr/share/java/mariadb-java-client.jar "%f"
```

3. 在 **Execute** 那行填：

```
java -cp .:/usr/share/java/mariadb-java-client.jar "%e"
```

4. 点 **确定**
5. 按 **F8** 编译 → 按 **F5** 运行

终端会显示查询结果。

---

## 八、Python 和 C 语言环境

|语言|验证命令|
|---|---|
|Python 3|`python3 --version`|
|C (gcc)|`gcc --version`|
|Java|`java -version`|

- Python 3 系统自带，无需安装
- C 语言：`gcc hello.c -o hello && ./hello`
- Java：`javac Hello.java && java Hello`

---

## 九、装 QQ 和输入法

### QQ Linux 版

从官方下 deb 包：`https://im.qq.com/linuxqq/`  
选 x86_64 deb → 下载后：

```bash
cd ~/下载
sudo dpkg -i linuxqq_*.deb
sudo apt install -f    # 如果提示缺依赖
```

> 注意：QQ Linux 版基于 Electron，N450 上可能会卡，勉强能用。

### 中文输入法（fcitx5）

```bash
sudo apt install fcitx5 fcitx5-chinese-addons fcitx5-frontend-gtk3 fcitx5-frontend-qt5 -y
reboot
```

重启后右下角键盘图标 → 右键配置 → 添加拼音 → **Ctrl + 空格** 切换中英文。

---

## 十、今天学到的排查思路

```
日志说啥 → 顺着路径查 → 定位到具体脚本/配置 → 小改动解决
```

|现象|排查步骤|根因|
|---|---|---|
|Tomcat 起不来|`sudo systemctl status tomcat9` → 报错 "No JDK found"|JAVA_HOME 没设|
|设了 JAVA_HOME 还是报错|看 service 文件 + 启动脚本 → 发现 `locate-java.sh`|脚本只搜 JDK 8~11|
|配置文件改了没生效|追 service unit 文件|发现根本没加载 env 文件|

核心心法：**Linux 不会骗你，日志就是真相的入口。**

---

## 十一、终端小技巧

|问题|答案|
|---|---|
|输 sudo 密码看不见|正常！Linux 盲打，打完回车就行|
|`ls: command not found`|检查拼写（别打成 `1s` 了）|
|`status` 写成了 `staus`|一个字母都不能错|
|卡在 status 界面出不来|按 **q**|
|nano 编辑器怎么存盘退出|Ctrl+O → 回车 → Ctrl+X|
|卡在数据库出不来|输入 `exit;` 回车|
|`python3-pip` 装不上|是 `python3-pip` 不是 `python3-pips`|

---

## 十二、Java Web 开发第一步（明天搞）

### 12.1 用 Geany 写纯 Java + 数据库

**第一步：设置 Geany 编译命令**

打开 Geany → 菜单栏 **生成** → **设置生成命令**：

|命令|内容|
|---|---|
|Compile|`javac -cp .:/usr/share/java/mariadb-java-client.jar "%f"`|
|Execute|`java -cp .:/usr/share/java/mariadb-java-client.jar "%e"`|

点确定。

**第二步：新建文件，写 JDBC 代码**

```java
import java.sql.*;

public class TestDB {
    public static void main(String[] args) {
        String url = "jdbc:mariadb://localhost:3306/mydb";
        String user = "myuser";
        String password = "123456";

        try {
            Class.forName("org.mariadb.jdbc.Driver");
            Connection conn = DriverManager.getConnection(url, user, password);
            System.out.println("✅ 数据库连接成功！");

            Statement stmt = conn.createStatement();
            ResultSet rs = stmt.executeQuery("SELECT * FROM students");

            while (rs.next()) {
                System.out.println(rs.getInt("id") + " | " 
                    + rs.getString("name") + " | " 
                    + rs.getInt("age") + " | " 
                    + rs.getString("grade"));
            }

            rs.close();
            stmt.close();
            conn.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

按 **F8** 编译，按 **F5** 运行。

### 12.2 Hello Servlet（纯 Web）

打开 Geany → 新建 → 写下面代码 → 保存为 `HelloServlet.java`：

```java
import javax.servlet.*;
import javax.servlet.http.*;
import javax.servlet.annotation.*;
import java.io.*;

@WebServlet("/hello")
public class HelloServlet extends HttpServlet {
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {
        resp.setContentType("text/html;charset=UTF-8");
        PrintWriter out = resp.getWriter();
        out.println("<h1>Hello from AO532h!</h1>");
    }
}
```

**Geany 编译命令设置：**

|命令|内容|
|---|---|
|Compile|`javac -cp /usr/share/tomcat9/lib/servlet-api.jar "%f"`|
|Execute|（不用设，Servlet 是部署到 Tomcat 的）|

**部署：**

```bash
# 编译
javac -cp /usr/share/tomcat9/lib/servlet-api.jar HelloServlet.java

# 部署
sudo mkdir -p /var/lib/tomcat9/webapps/hello/WEB-INF/classes
sudo cp HelloServlet.class /var/lib/tomcat9/webapps/hello/WEB-INF/classes/

# web.xml
sudo nano /var/lib/tomcat9/webapps/hello/WEB-INF/web.xml
```

```xml
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee" version="4.0">
</web-app>
```

```bash
# 重启 Tomcat
sudo systemctl restart tomcat9

# 浏览器打开
# http://localhost:8080/hello/hello
```

### 12.3 Servlet 查询数据库（实战）

创建一个从数据库读数据并显示在网页上的 Servlet：

```java
import javax.servlet.*;
import javax.servlet.http.*;
import javax.servlet.annotation.*;
import java.io.*;
import java.sql.*;

@WebServlet("/students")
public class StudentServlet extends HttpServlet {
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {
        
        resp.setContentType("text/html;charset=UTF-8");
        PrintWriter out = resp.getWriter();
        
        out.println("<!DOCTYPE html>");
        out.println("<html><head><meta charset='UTF-8'><title>学生列表</title></head><body>");
        out.println("<h1>学生列表（来自数据库）</h1>");
        out.println("<table border='1'><tr><th>ID</th><th>姓名</th><th>年龄</th><th>成绩</th></tr>");

        try {
            Class.forName("org.mariadb.jdbc.Driver");
            Connection conn = DriverManager.getConnection(
                "jdbc:mariadb://localhost:3306/mydb", "myuser", "123456");
            
            Statement stmt = conn.createStatement();
            ResultSet rs = stmt.executeQuery("SELECT * FROM students");

            while (rs.next()) {
                out.println("<tr>");
                out.println("<td>" + rs.getInt("id") + "</td>");
                out.println("<td>" + rs.getString("name") + "</td>");
                out.println("<td>" + rs.getInt("age") + "</td>");
                out.println("<td>" + rs.getString("grade") + "</td>");
                out.println("</tr>");
            }

            rs.close();
            stmt.close();
            conn.close();
        } catch (Exception e) {
            out.println("<p style='color:red'>错误：" + e.getMessage() + "</p>");
        }

        out.println("</table></body></html>");
    }
}
```

**编译 + 部署 + 重启：**

```bash
# Geany 编译命令：javac -cp /usr/share/tomcat9/lib/servlet-api.jar:/usr/share/java/mariadb-java-client.jar "%f"
javac -cp /usr/share/tomcat9/lib/servlet-api.jar:/usr/share/java/mariadb-java-client.jar StudentServlet.java

sudo cp StudentServlet.class /var/lib/tomcat9/webapps/hello/WEB-INF/classes/
sudo cp /usr/share/java/mariadb-java-client.jar /var/lib/tomcat9/lib/

sudo systemctl restart tomcat9

# 浏览器打开 http://localhost:8080/hello/students
```

### 12.4 Geany 编译命令速查

|项目类型|Compile 命令|
|---|---|
|纯 Java|`javac "%f"`|
|Java + 数据库|`javac -cp .:/usr/share/java/mariadb-java-client.jar "%f"`|
|Servlet|`javac -cp /usr/share/tomcat9/lib/servlet-api.jar "%f"`|
|Servlet + 数据库|`javac -cp /usr/share/tomcat9/lib/servlet-api.jar:/usr/share/java/mariadb-java-client.jar "%f"`|

---

## 十三、这台机器的体感评价

|能干什么|不能干什么|
|---|---|
|✅ 写 Java / C / Python|❌ 跑 IntelliJ / VS Code|
|✅ 跑 Tomcat + 数据库|❌ 跑 SQL Server|
|✅ 看教学视频（480p）|❌ 开 10 个浏览器标签|
|✅ 轻度上网查文档|❌ 同时开 QQ + 微信|
|✅ 学习 Linux 命令行|❌ 现代 AAA 游戏|

**总结：够用，能学，适合折腾。**

---

> 2026-05-06 · 一台 XP 上网本的告别式