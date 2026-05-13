快速试一下
在 Geany 里新建一个 JavaBean —— UserBean.java：
￼
java
package com.example;

public class UserBean {
    private String name;
    private int age;

    public UserBean() {}   // 必须有无参构造器

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

    public int getAge() { return age; }
    public void setAge(int age) { this.age = age; }
}
编译并部署到 Tomcat：
￼
bash
javac UserBean.java
sudo mkdir
 -p /var/lib/tomcat9/webapps/myapp/WEB-INF/classes/com/example
sudo cp UserBean.class /var/lib/tomcat9/webapps/myapp/WEB-INF/classes/com/example/
然后写一个 JSP —— user.jsp 放到 /var/lib/tomcat9/webapps/myapp/：
￼
jsp
<%@ page contentType="text/html;charset=UTF-8" %>
<jsp:useBean id="user" class="com.example.UserBean" scope="page" />
<jsp:setProperty name="user" property="name" value="张三" />
<jsp:setProperty name="user" property="age" value="22" />
<html>
<body>
    <h1>JavaBean 测试</h1>
    <p>姓名：<jsp:getProperty name="user" property="name" /></p>
    <p>年龄：<jsp:getProperty name="user" property="age" /></p>
</body>
</html>
不需要重启 Tomcat（JSP 会自动编译），直接浏览器打开：
￼
http://localhost:8080/myapp/user.jsp
就能看到页面了。
小结
组件
状态
JSP
✅ Tomcat 9 原生支持
JavaBean
✅ 就是普通 Java 类，JDK 编译
<jsp:useBean>
✅ 不需要额外 jar
EL 表达式 ${...}
✅ JSP 2.3 内置
JSP + JavaBean 是学 Java Web 的经典路线，你的机器完全能跑。但有一说一，现在业界很少用 jsp:useBean 了，主流是 Servlet + JDBC + JSP 做视图（MVC 模式）。