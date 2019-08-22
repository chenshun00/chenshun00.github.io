### 我眼中的HttpServletRequest

#### 前言
要理解 `SpringMVC` 是怎么处理 `http` 请求的，就必须明白`http` 请求的 `uri` ，`url` 是如何生成的，`Servlet 引擎(web容器)` 是如何映射 servlet 和 怎么和http请求链接起来的。本节使用的代码你可以在 ----> [😄这里](https://github.com/chenshun00/verification)  找到

读完本文将会学习到

*	tomcat 的 context-path，和HttpServletRequest 的servlet-path，path-info，query-sering
* 	了解 `Servlet` 的映射规范

#### path分析
*	我们从一个实际案例当中分析上述几个 `path` 的区别，我们是机遇 `tomcat` 容器，首先因为 `maven` 的 `tomcat` 插件

```java
 <plugin>
    <groupId>org.apache.tomcat.maven</groupId>
    <artifactId>tomcat7-maven-plugin</artifactId>
    <version>2.2</version>
    <configuration>
        <url>http://127.0.0.1</url>
        <port>10289</port>
        <!--
        这里的path -> 其实就是context path，如果不填就是默认的项目名,后续会详细比较这些区别
        http://127.0.0.1:10289/context-path/servlet-path/pathinfo
        如果是 / 默认的servlet ，那么servlet path = uri
        如果是 /* 那么servlet path = null/""， pathinfo = 全部 这就是路径映射了
        *.do 后缀映射，uri - *.do 就是路径了
        /xxx 精确匹配
        -->
     <path>/</path> <-- context-path 是修改这里的path来实现的，如果注释掉这里，那么就是默认的项目名称 -->
     <uriEncoding>utf-8</uriEncoding>
   </configuration>
 </plugin>
```

通过引入 `tomcat` 插件，我们就可以使用 `tomcat` 了，以下的数据是我基于我的系统，使用如下配置实现的，基本上没有什么差别

*	系统 uname -a 

```java
 Darwin chenshundeMacBook-Pro.local 17.7.0 Darwin Kernel Version 17.7.0: Thu Jun 21 22:53:14 PDT 2018; root:xnu-4570.71.2~1/RELEASE_X86_64 x86_64
```
 
 *	java版本 java -version
	
``` java
java version "1.8.0_161"
Java(TM) SE Runtime Environment (build 1.8.0_161-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.161-b12, mixed mode)
```

`web.xml ` 配置的 `Servlet` 和 `Mapping` 映射信息如下

```java
 <servlet>
     <servlet-name>spring-mvc</servlet-name>
     <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
     <init-param>
         <param-name>contextConfigLocation</param-name>
         <param-value>classpath:spring-mvc.xml</param-value>
     </init-param>
     <load-on-startup>1</load-on-startup>
 </servlet>
<servlet-mapping>
    <servlet-name>spring-mvc</servlet-name>
    <-- 后面的测试结果都是通过改变这里的映射来说明 -->
    <url-pattern>*.json</url-pattern>
</servlet-mapping>
<servlet>
    <servlet-name>my-servlet</servlet-name>
    <servlet-class>top.huzhurong.cmdb.MyServlet</servlet-class>
</servlet>
<servlet-mapping>
    <servlet-name>my-servlet</servlet-name>
    <url-pattern>/servlet/*</url-pattern>
</servlet-mapping>
```
测试结果，如果 `windows` 不支持 `curl` 命令，可以从浏览器当中输入，结果一致.

```
curl 'http://localhost:10289/test/a/b/c/d.json?a=d&fg=d&name=1'
------- /* ------ 的情况
请求路径uri:/test/a/b/c/d.json 对应的http uri的结果，结果是吻合的
请求路径url:http://localhost:10289/test/a/b/c/d.json 
请求路径contextPath:/test 一致，path节点我开始填的是test，可得知结果是吻合的
请求路径servletPath:空，和httpServletRequest的处理一致，因为直接映射了 /* ，后续会分析什么是这样
请求路径pathinfo:/a/b/c/d.json 成功说明/*是路径映射
请求路径QueryString:a=d&fg=d&name=1 查询字符串


curl 'http://localhost:10289/test/a/b/c/d.json?a=d&fg=d&name=1'
------ / -------的情况
请求路径uri:/test/a/b/c/d.json 一致
请求路径url:http://localhost:10289/test/a/b/c/d.json 一致
请求路径contextPath:/test 一致
请求路径servletPath:/a/b/c/d.json 说明是默认的servlet
请求路径pathinfo:null ，默认的servlet它的pathinfo是null
请求路径QueryString:a=d&fg=d&name=1 查询字符串

curl 'http://localhost:10289/test/a/b/c/d.json?a=d&fg=d&name=1'
--------------*.json ---------的情况 后缀匹配，servlet path为映射成功的部分
请求路径uri:/test/a/b/c/d.json
请求路径url:http://localhost:10289/test/a/b/c/d.json
请求路径contextPath:/test
请求路径servletPath:/a/b/c/d.json
请求路径pathinfo:null
请求路径QueryString:a=d&fg=d&name=1

*.json的另一情况，其实这里不是被*.json处理成功了，因为它没有被找到
curl 'http://localhost:10289/test/a/b/c/d.json/z/d/e?a=d&fg=d&name=1'
请求路径uri:/test/a/b/c/d.json/z/d/e
请求路径url:http://localhost:10289/test/a/b/c/d.json/z/d/e
请求路径contextPath:/test
请求路径servletPath:/a/b/c/d.json/z/d/e
请求路径pathinfo:null
请求路径QueryString:a=d&fg=d&name=1

---------- /servlet/* ---------- 的情况，这里的path已经被去掉了
curl 'http://localhost:10289/servlet/z/d/e?a=d&fg=d&name=1'
请求路径uri: /servlet/z/d/e
请求路径url: http://localhost:10289/servlet/z/d/e
请求路径contextPath: ""
请求路径servletPath: /servlet
请求路径pathinfo: /z/d/e?a=d
请求路径QueryString:a=d&fg=d&name=1
```

> 路径出现了 test 是因为带了path，且填写的是test，其他情况是 /

出现上述结果的原因如下

*	`servlet-path` : 返回这个url调用servlert的那一部分，这个路径从` / ` 字符开始,包括了servlet 名字或者到这个servlet的路径，但是不包括其他的额外路径信息(pathinfo)，或者查询字符串(query string)，和cgi变量类同，如果servlet使用/*去匹配http路径，那么servlet path 是个空字符串

*	`request uri` : 返回http 请求的url协议和服务器域名名称之后，查询字符串之前的一部分，web容器不会对这个路径解码

*	`request url` : 重新构建url，包括了协议，服务器名称，端口，路径，但是不包括查询字符串

*	`queryString` : 返回http 请求路径之后的那一部分，通常来说是?之后的那一部分

*	`context path` : `request uri` 的一部分，`context path` 主要是为了表明这个请求的上下文，其实在其他的http实现当中是没有这个概念的，只是因为一个 `tomcat` 可以放 `多个web项目`，需要一个上下文的概念来进行区分，才有了这个概念，[例如php，go，py，node是没有这个概念的，我猜测],context path以/开始，但是不是以/结尾，对于在默认的上下文之中的servlert来说这个path是 "" ，

*	`pathinfo` : 除去匹配中的servlet路径之外，包含在uri之内的其余部分。去除 `context-path`

根据上述的情况，我们已经稍微明白了这些数据之间的区别了，但是肯定还有部分人不明白了 `匹配中的Servlet部分` 是什么意思。那么就不行去了解一下Servlet规范当中关于路径映射的这一部分了，而且也不要忘了上述 `web.xml` 中定义的 `<url-pattern>*.json</url-pattern>` 的路径映射部分。这里就明确了 `匹配中的Servlet部分`。

#### servlet 规范
```
In the Web application deployment descriptor, the following syntax is used to define mappings:

A string beginning with a / character and ending with a /* suffix is used for path mapping.
/ 开头， /* 结尾(包含了/*) 是用来作为路径映射的。 Pathinfo 的映射

A string beginning with a *. prefix is used as an extension mapping.
*.xxx 结尾是用来作为拓展映射的，这个很好理解

The empty string ("") is a special URL pattern that exactly maps to the application's context root, i.e., requests of the form http://host:port/<context-root>/. In this case the path info is / and the servlet path and context path is empty string (““).
空字符串[””]是特殊的url它会精确的映射到web应用到更根目录

A string containing only the / character indicates the "default" servlet of the application. In this case the servlet path is the request URI minus the context path and the path info is null.
只包含 [/] 字符表明这是应用到[默认]servlet, 这个类型当中，servlet path = uri - context path - path info (后边2个是null)

All other strings are used for exact matches only.
剩下的就是精准匹配了
```
****
**一定要了解context-path是用来区分容器的，因为Tomcat可以部署多个web应用，使用上下文的概念去处理即将到来的http请求**

从上边的规范，在结合实现的运行情况，我们可以如下分析:

1. `/` 是默认的servlet映射，整个路径都是servlet-path，除去context-path以外。匹配中的servlet部分就是整个path 
2. `/*` 是路径映射，servlet-path是空`""`,除去context-path以外。 其余都是path-info
3. `*.json` 是拓展映射，数据xx.json结尾的数据都会命中这里，这就是数据匹配中的Servlet部分
4. 精准映射，xml中我自定义的 `my-servlet` 就属于这一列，所有http://127.0.0.1:10289/servlet/*部分的url都会被精确处理 `通配符`了解一下，后续写到正则表达式会写到这里

#### 后续

经过如上分析 ，心里已经有了一点点的思路了，这个时候一定要结合代码，结合实际的运行情况去看看，实际是怎么样子的。代码可以在这里找到 ---> [😄这里](https://github.com/chenshun00/verification)  
