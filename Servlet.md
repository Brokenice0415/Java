<h1>Servlet</h1>

<span id="目录"><b>目录</b></span>

[toc]

Servlet的学习过程参考[菜鸟教程](https://www.runoob.com/servlet/servlet-tutorial.html)

这里仅做学习补充

## 环境配置

### Preferences中元素缺失

在配置Ecilpse和Tomcat时，需要在Ecilpse的`Window->Preferences`中的Server设置Tomcat路径

但有可能其中缺少Server，甚至Web，XML等选项

这时候需要在`Help->Install New Software`中选择一个最新的release版本下载，并且勾选需要的内容

下载完毕后重启即可

## 表单数据

### 多个Servlet

这里在一个项目中创建了多个Servlet，并且给出了每个Servlet对应的web.xml配置，但是并非需要多个xml文件，将这些配置代码写入一个xml即可，比如存在HelloServlet和HelloForm两个Servlet，则配置web.xml如下

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app>
  <servlet>
    <servlet-name>HelloServlet</servlet-name>
    <servlet-class>com.runoob.test.HelloServlet</servlet-class>
  </servlet>
  <servlet-mapping>
    <servlet-name>HelloServlet</servlet-name>
    <url-pattern>/TestWeb/HelloServlet</url-pattern>
  </servlet-mapping>
  
  <servlet>
    <servlet-name>HelloForm</servlet-name>
    <servlet-class>com.runoob.test.HelloForm</servlet-class>
  </servlet>
  <servlet-mapping>
    <servlet-name>HelloForm</servlet-name>
    <url-pattern>/TestWeb/HelloForm</url-pattern>
  </servlet-mapping>
</web-app>
```

### Get中文字符问题

在[菜鸟教程](https://www.runoob.com/servlet/servlet-form-data.html)中，使用如下代码处理中文字符的获取和显示

```java
// 设置响应内容类型
        response.setContentType("text/html;charset=UTF-8");
 // 处理中文
        String name =new String(request.getParameter("name").getBytes("ISO-8859-1"),"UTF-8");
```

但是在本机运行却出现中文乱码“？”

在百度上搜索得到菜鸟教程中所给方法为**Tomcat8.0之前**所使用的方法，在Tomcat8.0及其后URIEncoding默认编码方式已经由之前的ISO-8859-1变为UTF-8，即我们不需要对读取的文字再进行特殊处理了

Tomcat8.0及其后写法

```java
// 设置响应内容类型
        response.setContentType("text/html;charset=UTF-8");
//读取无需特殊处理
//name = request.getParameter("name");
```

## 过滤器Filter

- web.xml配置

  - 可以通过\<filter>中的\<init-param>设置过滤器的初始化参数

    ```xml
    <filter>
    	<filter-name>MyFilter</filter-name>
        <filter-class>FilterTest.MyFilter</filter-class>
        <init-param>
        	<param-name>p1</param-name>
            <param-value>1</param-value>
        </init-param>
        <init-param>
        	<param-name>p2</param-name>
            <param-value>2</param-value>
        </init-param>
    </filter>
    <filter-mapping>
        <filter-name>MyFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
    ```

    在Filter中这些参数通过`config.getInitParameter("_paramName_")`来获取

- java.lang.NullPointerException

  - 判断uri时，常采用`HttpServletRequest.getRequestURI()`方法，然后对所得的uri字符串进行处理

    ```java
    //父类转换子类
    HttpServletRequest req = (HttpServletRequest) request;
    HttpServletResponse resp = (HttpServletResponse) response;
    //获取uri
    String uri = req.getRequestURI();
    //uri尾部"/"后元素
    String endWith = uri.substring(uri.lastIndexOf("/") + 1);
    //获取参数
    String param = req.getParameter("name");
    
    //之所以这样写而非endWith.equals("hello.html")
    //是因为考虑到当endWith为空时，调用equals方法就会抛出"java.lang.NullPointerException"异常
    if("hello.html".equals(endWith)) {
        //进入下一层过滤
        chain.doFilter(req, resp);
    }else {
        if("admin".equals(param)) {
            chain.doFilter(req, resp);
        }else {
            //页面重定位
            resp.sendRedirect("hello.html");
        }
    }
    ```


## Cookie和Session

两者相类似，都是用于存储客户端和服务端之间的信息的

- cookie存储于客户端，也就是存储于本地，浏览器中通常有清空缓存与cookie选项，里面的cookie就是这个cookie
- session存储于服务端，其相对cookie数据较安全

### Session跳转访问问题

我在实操[菜鸟教程](https://www.runoob.com/servlet/servlet-session-tracking.html)的Session的例子中，由于我是使用了过滤器实现登陆后，经过主页跳转到服务端TestWeb/SessionTrack目录，读取到的session中的用户id为null，而不是我所指定的"admin"

<img src="img\Servlet\1.png" alt="image-20200828203814158" style="zoom:50%;" />

即便删去过滤器，只要是客户端先访问了服务端的其他页面后再访问TestWeb/SessionTrack，即便是第一次访问，用户id都将会是null，之后无论怎么刷新都不会改变其结果

```java
//部分代码
HttpSession session = request.getSession();

//设置默认值
String title = "Servlet Session实例";
Integer visitCount = 0;
String visitCountKey = new String("visitCount");
String userIDKey = new String("userID");
String userID = new String("admin");
if(session.getAttribute(visitCountKey) == null) {
    session.setAttribute(visitCountKey, visitCount);
}

//获取新Session
if (session.isNew()){
    title = "Servlet Session 实例";
    session.setAttribute(userIDKey, userID);
} else {
    visitCount = (Integer)session.getAttribute(visitCountKey);
    visitCount = visitCount + 1;
    userID = (String)session.getAttribute(userIDKey);
}
session.setAttribute(visitCountKey,  visitCount);
```

从代码逻辑上看，首先判断session是否为new，若是的话设置session的userIDKey和userID为我们指定的默认值。

**但菜鸟教程上是给出的当我们直接访问SessionTrack时候的情况。**

当客户端请求服务端的其他页面，若是第一次访问，服务端就会给出一个sessionID，名为JSESSIONID，同时保存在cookie中。然后在跳转到我们的SessionTrack时，服务端读取cookie，发现里面有一条JSESSIONID:\*\*\*\*\*\*\*\*\*的记录，判断为已有session。于是在session.isNew()中返回false，导致我们设置的userID没有置入session。

```java
boolean session.isNew(){
	if(cookie("JSESSIONID", JSESSIONID) in cookies){
        return false;
    }
    return true;
}
```

**在Java中，Session是Cookie的子集，Session不能离开Cookie而存在。**

**在.NET中，微软实现了Session和Cookie的分离，也就是说，如果浏览器不支持Cookie，Java写的服务端同样不支持Session，而.NET中支持**

因此在不使用Cookie的情况下Java写的服务端只能通过 **重写URL** 的方式获取Session，[菜鸟教程](https://www.runoob.com/servlet/servlet-session-tracking.html)中有相关的介绍，但没有相关的例子来具体说明。

这边给出我使用cookie的判断用户对**跳转后的当前网页（保证cookie不为空）**的访问次数方法

- 定义一个sessionFlag的cookie，当无cookie或cookie中没有sessionFlag时创建

```java
String sessionFlagKey = new String("sessionFlag");
String sessionFlag = new String("Y");
boolean isNew = true;

//判断cookie中是否包含sessionFlag
Cookie[] cookies = request.getCookies();
if(cookies != null) {
    for(int i = 0; i < cookies.length; i++) {
        if(sessionFlagKey.equals(cookies[i].getName())) {
            isNew = false;
            break;
        }
    }
}

//通过布尔变量isNew代替session.isNew()
if (isNew){
    session.setAttribute(userIDKey, userID);
    response.addCookie(new Cookie(sessionFlagKey, sessionFlag));
    System.out.println("NEW SESSION");
} else {
    //省略
}
```



### 重写URL实现Session

首先要说明的是，重写URL实现Session会动态生成每个 URL 来为页面分配一个 session 会话 ID，使得后端的开发十分的麻烦。

HttpServletResponse接口提供了重写 URL 的方法：`public java.lang.String encodeURL(java.lang.String url)`

- 当不存在或不能发送key为JSESSIONID的Cookie时，路径后面追加参数
- 如果Cookie存在且可以发送服务器，后面不会追加参数

下面是一个例子，能够实现无论cookie是否禁用都能实现上面的Session实例

入口jsp重写URL

```jsp
<%@ page language="java" contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8"%>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
<title>Hello</title>
</head>
<body>
	<a href="<%=response.encodeURL("/TestSession/SessionTrack") %>">Hello!</a>
</body>
</html>
```

Servlet

```java
protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
    //getSession优先读取cookie中的JSESSIONID，其次是URL中传入的jsessionid
		HttpSession session = request.getSession();
    //读取sessionID，并设置对应的cookie
		Cookie cookie = new Cookie("JSESSIONID", session.getId());
		cookie.setMaxAge(60);
		response.addCookie(cookie);
    
		Date createTime = new Date(session.getCreationTime());
		Date lastAccessTime = new Date(session.getLastAccessedTime());
		SimpleDateFormat df = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
		
		String title = "Servlet Session实例";
		Integer visitCount = 0;
		String visitCountKey = new String("visitCount");
		String userIDKey = new String("userID");
		String userID = new String("admin");

    //当session中不存在visitCounts时置入0
		if(session.getAttribute(visitCountKey) == null) {
			session.setAttribute(visitCountKey, visitCount);
		}
   		session.setAttribute(userIDKey, userID);
    
    //获得session中存储的visitCount
        visitCount = (Integer)session.getAttribute(visitCountKey);
        visitCount = visitCount + 1;
    //visitCount++
        userID = (String)session.getAttribute(userIDKey);
        session.setAttribute(visitCountKey,  visitCount);
		
		response.setContentType("text/html;charset=UTF-8");
		PrintWriter out = response.getWriter();

		String docType = "<!DOCTYPE html>\n";
		out.println(docType +
                "<html>\n" +
                "<head><title>" + title + "</title></head>\n" +
                "<body bgcolor=\"#f0f0f0\">\n" +
                "<h1 align=\"center\">" + title + "</h1>\n" +
                 "<h2 align=\"center\">Session 信息</h2>\n" +
                "<table border=\"1\" align=\"center\">\n" +
                "<tr bgcolor=\"#949494\">\n" +
                "  <th>Session 信息</th><th>值</th></tr>\n" +
                "<tr>\n" +
                "  <td>id</td>\n" +
                "  <td>" + session.getId() + "</td></tr>\n" +
                "<tr>\n" +
                "  <td>创建时间</td>\n" +
                "  <td>" +  df.format(createTime) + 
                "  </td></tr>\n" +
                "<tr>\n" +
                "  <td>最后访问时间</td>\n" +
                "  <td>" + df.format(lastAccessTime) + 
                "  </td></tr>\n" +
                "<tr>\n" +
                "  <td>用户 ID</td>\n" +
                "  <td>" + userID + 
                "  </td></tr>\n" +
                "<tr>\n" +
                "  <td>访问统计：</td>\n" +
                "  <td>" + visitCount + "</td></tr>\n" +
                "</table>\n" +
                "</body></html>"); 
    }
```

## Servlet 数据库访问

需要注意的是，在mySQL8.0及以上版本，JDBC_DRIVER需要设置为“com.mysql.cj.jdbc.Driver"，同时需要设置关闭SSL，允许公钥和设置CST

```java
static final String JDBC_DRIVER = "com.mysql.jdbc.Driver";
static final String DB_URL = "jdbc:mysql://localhost:3306/webtest";
/*
//mySQL8.0及以上
static final String JDBC_DRIVER = "com.mysql.cj.jdbc.Driver";
static final String DB_URL = "jdbc:mysql://localhost:3306/webtest?userSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC";
*/
```

### JSP 数据库访问

- 需要将两个包置入WebContent/WEB-INF/lib文件夹（Tomcat9.0）
  - jstl-1.2.jar
  - standard-1.1.2.jar

```jsp
<%@ page language="java" contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8"%>
<%@ page import="java.io.* , java.util.* , java.sql.*" %>
<%@ page import="javax.servlet.http.* , javax.servlet.*" %>
<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c" %>
<%@ taglib uri="http://java.sun.com/jsp/jstl/sql" prefix="sql"%>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
<title>SQL Test</title>
</head>
<body>
<sql:setDataSource var="snapshot" driver="com.mysql.cj.jdbc.Driver"
     url="jdbc:mysql://localhost:3306/webtest?useSSL=false&serverTimezone=UTC&useUnicode=true&characterEncoding=utf-8"
     user="root"  password="123"/>
<sql:query dataSource="${snapshot}" var="result">
	SELECT * FROM websites;
</sql:query>
<h1>数据库Test</h1>

<form>
web name:<br>
<input id="webName" type="text" name="webName">
<br>
web url:<br>
<input id="webUrl" type="text" name="webUrl">
<br><br>
<input type="submit" value="提交">
<br><br>
</form> 

<table border="1" wideth="100%">
<tr>
	<th>ID</th>
	<th>站点名</th>
	<th>站点地址</th>
</tr>
<c:forEach var="row" items="${result.rows}">
<tr>
	<td><c:out value="${row.id}"/></td>
	<td><c:out value="${row.name}"/></td>
	<td><c:out value="${row.url}"/></td>
</tr>
</c:forEach>
</table>

</body>
</html>
```

