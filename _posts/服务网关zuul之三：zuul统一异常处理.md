# [                   服务网关zuul之三：zuul统一异常处理](https://www.cnblogs.com/duanxz/p/7543040.html)

我们详细介绍了Spring Cloud Zuul中自己实现的一些核心过滤器，以及这些过滤器在请求生命周期中的不同作用。我们会发现在这些核心过滤器中并没有实现error阶段的过滤器。那么这些过滤器可以用来做什么呢？接下来，本文将介绍如何利用error过滤器来实现统一的异常处理。

## 过滤器中抛出异常的问题

首先，我们可以来看看默认情况下，过滤器中抛出异常Spring Cloud Zuul会发生什么现象。我们创建一个pre类型的过滤器，并在该过滤器的run方法实现中抛出一个异常。比如下面的实现，在run方法中调用的`doSomething`方法将抛出`RuntimeException`异常。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
package com.dxz.zuul;

import org.apache.log4j.Logger;
import com.netflix.zuul.ZuulFilter;

public class ThrowExceptionFilter extends ZuulFilter {

    private static Logger log = Logger.getLogger(ThrowExceptionFilter.class);

    @Override
    public String filterType() {
        return "pre";
    }

    @Override
    public int filterOrder() {
        return 0;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() {
        log.info("This is a pre filter, it will throw a RuntimeException");
        doSomething();
        return null;
    }

    private void doSomething() {
        throw new RuntimeException("Exist some errors...");
    }

}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

在启动类为自定义过滤器创建具体的Bean才能启动该过滤器，如下：

```
    @Bean
    public ThrowExceptionFilter throwExceptionFilter() {
        return new ThrowExceptionFilter();
    }
```

运行网关程序并访问某个路由请求，此时我们会发现：在API网关服务的控制台中输出了ThrowExceptionFilter的过滤逻辑中的日志信息，但是并没有输出任何异常信息，同时发起的请求也没有获得任何响应结果。为什么会出现这样的情况呢？我们又该如何在过滤器中处理异常呢？

 

### **解决方案一：严格的try-catch处理**

　　运行网关程序并访问某个路由请求，此时我们会发现：在API网关服务的控制台中输出了`ThrowExceptionFilter`的过滤逻辑中的日志信息，但是并没有输出任何异常信息，同时发起的请求也没有获得任何响应结果。为什么会出现这样的情况呢？我们又该如何在过滤器中处理异常呢？

回想一下，我们在上一节中介绍的所有核心过滤器，是否还记得有一个`post`过滤器`SendErrorFilter`是用来处理异常信息的？根据正常的处理流程，该过滤器会处理异常信息，那么这里没有出现任何异常信息说明很有可能就是这个过滤器没有被执行。所以，我们不妨来详细看看`SendErrorFilter`的`shouldFilter`函数：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
    @Override
    public boolean shouldFilter() {
        RequestContext ctx = RequestContext.getCurrentContext();
        // only forward to errorPath if it hasn't been forwarded to already
        return ctx.containsKey("error.status_code")
                && !ctx.getBoolean(SEND_ERROR_FILTER_RAN, false);
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

![img](https://images2017.cnblogs.com/blog/285763/201709/285763-20170918150433150-2013481451.png)

可以看到该方法的返回值中有一个重要的判断依据`ctx.containsKey("error.status_code")`，也就是说请求上下文中必须有`error.status_code`参数，我们实现的`ThrowExceptionFilter`中并没有设置这个参数，所以自然不会进入`SendErrorFilter`过滤器的处理逻辑。那么我们要如何用这个参数呢？我们可以看一下`route`类型的几个过滤器，由于这些过滤器会对外发起请求，所以肯定会有异常需要处理，比如spring-cloud-netflix-core-1.1.4.RELEASE.jar中的org.springframework.cloud.netflix.zuul.filters.route.`RibbonRoutingFilter`的`run`方法实现如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
    @Override
    public Object run() {
        RequestContext context = RequestContext.getCurrentContext();
        this.helper.addIgnoredHeaders();
        try {
            RibbonCommandContext commandContext = buildCommandContext(context);
            ClientHttpResponse response = forward(commandContext);
            setResponse(response);
            return response;
        }
        catch (ZuulException ex) {
            context.set(ERROR_STATUS_CODE, ex.nStatusCode);
            context.set("error.message", ex.errorCause);
            context.set("error.exception", ex);
        }
        catch (Exception ex) {
            context.set("error.status_code",
                    HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
            context.set("error.exception", ex);
        }
        return null;
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

可以看到，整个发起请求的逻辑都采用了`try-catch`块处理。在`catch`异常的处理逻辑中并没有做任何输出操作，而是往请求上下文中添加一些`error`相关的参数，主要有下面三个参数：

- `error.status_code`：错误编码
- `error.exception`：`Exception`异常对象
- `error.message`：错误信息

其中，`error.status_code`参数就是`SendErrorFilter`过滤器用来判断是否需要执行的重要参数。分析到这里，实现异常处理的大致思路就开始明朗了，我们可以参考`RibbonRoutingFilter`的实现对`ThrowExceptionFilter`的`run`方法做一些异常处理的改造，具体如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
    @Override
    public Object run() {
        log.info("This is a pre filter, it will throw a RuntimeException");
        RequestContext ctx = RequestContext.getCurrentContext();
        try {
            doSomething();
        } catch (Exception e) {
            ctx.set("error.status_code", HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
            ctx.set("error.exception", e);
        }
        return null;
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

通过上面的改造之后，我们再尝试访问之前的接口，这个时候我们可以得到如下响应内容：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
{
"timestamp": 1481674980376,
"status": 500,
"error": "Internal Server Error",
"exception": "java.lang.RuntimeException",
"message": "Exist some errors..."
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

此时，我们的异常信息已经被`SendErrorFilter`过滤器正常处理并返回给客户端了，同时在网关的控制台中也输出了异常信息。从返回的响应信息中，我们可以看到几个我们之前设置在请求上下文中的内容，它们的对应关系如下：

- `status`：对应`error.status_code`参数的值
- `exception`：对应`error.exception`参数中`Exception`的类型
- `message`：对应`error.exception`参数中`Exception`的`message`信息。对于`message`的信息，我们在过滤器中还可以通过`ctx.set("error.message", "自定义异常消息");`来定义更友好的错误信息。`SendErrorFilter`会优先取`error.message`来作为返回的`message`内容，如果没有的话才会使用`Exception`中的`message`信息

### 解决方案二：ErrorFilter处理

通过上面的分析与实验，我们已经知道如何在过滤器中正确的处理异常，让错误信息能够顺利地流转到后续的`SendErrorFilter`过滤器来组织和输出。但是，即使我们不断强调要在过滤器中使用`try-catch`来处理业务逻辑并往请求上下文添加异常信息，但是不可控的人为因素、意料之外的程序因素等，依然会使得一些异常从过滤器中抛出，对于意外抛出的异常又会导致没有控制台输出也没有任何响应信息的情况出现，那么是否有什么好的方法来为这些异常做一个统一的处理呢？

这个时候，我们就可以用到`error`类型的过滤器了。由于在请求生命周期的`pre`、`route`、`post`三个阶段中有异常抛出的时候都会进入`error`阶段的处理，所以我们可以通过创建一个`error`类型的过滤器来捕获这些异常信息，并根据这些异常信息在请求上下文中注入需要返回给客户端的错误描述，这里我们可以直接沿用在`try-catch`处理异常信息时用的那些error参数，这样就可以让这些信息被`SendErrorFilter`捕获并组织成消息响应返回给客户端。比如，下面的代码就实现了这里所描述的一个过滤器：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
package com.dxz.zuul;

import javax.servlet.http.HttpServletResponse;

import org.apache.log4j.Logger;

import com.netflix.zuul.ZuulFilter;
import com.netflix.zuul.context.RequestContext;

public class ErrorFilter extends ZuulFilter {

    Logger log = Logger.getLogger(ErrorFilter.class);

    @Override
    public String filterType() {
        return "error";
    }

    @Override
    public int filterOrder() {
        return 10;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        Throwable throwable = ctx.getThrowable();
        log.error("this is a ErrorFilter :" + throwable.getCause().getMessage(), throwable);
        ctx.set("error.status_code", HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
        ctx.set("error.exception", throwable.getCause());
        return null;
    }

}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

在启动类为自定义过滤器创建具体的Bean才能启动该过滤器，如下：

```
    @Bean
    public ErrorFilter errorFilter() {
        return new ErrorFilter();
    }
```

修改并保留ThrowExceptionFilter，

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
    @Override
    public Object run() {
        log.info("This is a pre filter, it will throw a RuntimeException");
        doSomething();
        /*RequestContext ctx = RequestContext.getCurrentContext();
        try {
            doSomething();
        } catch (Exception e) {
            ctx.set("error.status_code", HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
            ctx.set("error.exception", e);
        }*/
        return null;
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

在将该过滤器加入到我们的API网关服务之后，我们可以尝试使用之前介绍`try-catch`处理时实现的`ThrowExceptionFilter`（不包含异常处理机制的代码），让该过滤器能够抛出异常。这个时候我们再通过API网关来访问服务接口。此时，我们就可以在控制台中看到`ThrowExceptionFilter`过滤器抛出的异常信息，并且请求响应中也能获得如下的错误信息内容，而不是什么信息都没有的情况了。

![img](https://images2017.cnblogs.com/blog/285763/201709/285763-20170918151619368-1555310138.png)

 

两种解决方案：一种是通过在各个阶段的过滤器中增加`try-catch`块，实现过滤器内部的异常处理；另一种是利用`error`类型过滤器的生命周期特性，集中地处理`pre`、`route`、`post`阶段抛出的异常信息。通常情况下，我们可以将这两种手段同时使用，其中第一种是对开发人员的基本要求；而第二种是对第一种处理方式的补充，以防止一些意外情况的发生。这样的异常处理机制看似已经完美，但是如果在多一些应用实践或源码分析之后，我们会发现依然存在一些不足。

### 不足之处

下面，我们不妨跟着源码来看看，到底上面的方案还有哪些不足之处需要我们注意和进一步优化的。先来看看外部请求到达API网关服务之后，各个阶段的过滤器是如何进行调度的：

![img](https://images2017.cnblogs.com/blog/285763/201709/285763-20170918154940556-1741762725.png)

 

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
    @Override
    public void service(javax.servlet.ServletRequest servletRequest, javax.servlet.ServletResponse servletResponse) throws ServletException, IOException {
        try {
            init((HttpServletRequest) servletRequest, (HttpServletResponse) servletResponse);

            // Marks this request as having passed through the "Zuul engine", as opposed to servlets
            // explicitly bound in web.xml, for which requests will not have the same data attached
            RequestContext context = RequestContext.getCurrentContext();
            context.setZuulEngineRan();

            try {
                preRoute();
            } catch (ZuulException e) {
                error(e);
                postRoute();
                return;
            }
            try {
                route();
            } catch (ZuulException e) {
                error(e);
                postRoute();
                return;
            }
            try {
                postRoute();
            } catch (ZuulException e) {
                error(e);
                return;
            }

        } catch (Throwable e) {
            error(new ZuulException(e, 500, "UNHANDLED_EXCEPTION_" + e.getClass().getName()));
        } finally {
            RequestContext.getCurrentContext().unset();
        }
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

问题分析与进一步优化上面代码源自`com.netflix.zuul.http.ZuulServlet`的`service`方法实现，它定义了Zuul处理外部请求过程时，各个类型过滤器的执行逻辑。从代码中我们可以看到三个`try-catch`块，它们依次分别代表了`pre`、`route`、`post`三个阶段的过滤器调用，在`catch`的异常处理中我们可以看到它们都会被`error`类型的过滤器进行处理（之前使用`error`过滤器来定义统一的异常处理也正是利用了这个特性）；`error`类型的过滤器处理完毕之后，除了来自`post`阶段的异常之外，都会再被`post`过滤器进行处理。而对于从`post`过滤器中抛出异常的情况，在经过了`error`过滤器处理之后，就没有其他类型的过滤器来接手了，这就是使用之前所述方案存在不足之处的根源。

回想一下之前实现的两种异常处理方法，其中非常核心的一点，这两种处理方法都在异常处理时候往请求上下文中添加了一系列的`error.*`参数，而这些参数真正起作用的地方是在`post`阶段的`SendErrorFilter`，在该过滤器中会使用这些参数来组织内容返回给客户端。而对于`post`阶段抛出异常的情况下，由`error`过滤器处理之后并不会在调用`post`阶段的请求，自然这些`error.*`参数也就不会被`SendErrorFilter`消费输出。所以，如果我们在自定义`post`过滤器的时候，没有正确的处理异常，就依然有可能出现日志中没有异常并且请求响应内容为空的问题。我们可以通过修改之前`ThrowExceptionFilter`的`filterType`修改为`post`来验证这个问题的存在，注意去掉`try-catch`块的处理，让它能够抛出异常。

解决上述问题的方法有很多种，比如最直接的我们可以在实现`error`过滤器的时候，直接来组织结果返回就能实现效果，但是这样的缺点也很明显，对于错误信息组织和返回的代码实现就会存在多份，这样非常不易于我们日后的代码维护工作。所以为了保持对异常返回处理逻辑的一致，我们还是希望将`post`过滤器抛出的异常能够交给`SendErrorFilter`来处理。

在前文中，我们已经实现了一个`ErrorFilter`来捕获`pre`、`route`、`post`过滤器抛出的异常，并组织`error.*`参数保存到请求的上下文中。由于我们的目标是沿用`SendErrorFilter`，这些`error.*`参数依然对我们有用，所以我们可以继续沿用该过滤器，让它在`post`过滤器抛出异常的时候，继续组织`error.*`参数，只是这里我们已经无法将这些`error.*`参数再传递给`SendErrorFitler`过滤器来处理了。所以，我们需要在`ErrorFilter`过滤器之后再定义一个`error`类型的过滤器，让它来实现`SendErrorFilter`的功能，但是这个`error`过滤器并不需要处理所有出现异常的情况，它仅对`post`过滤器抛出的异常才有效。根据上面的思路，我们完全可以创建一个继承自`SendErrorFilter`的过滤器，就能复用它的`run`方法，然后重写它的类型、顺序以及执行条件，实现对原有逻辑的复用，具体实现如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
package com.dxz.zuul;

import org.springframework.cloud.netflix.zuul.filters.post.SendErrorFilter;
import org.springframework.stereotype.Component;

@Component
public class ErrorExtFilter extends SendErrorFilter {

    @Override
    public String filterType() {
        return "error";
    }

    @Override
    public int filterOrder() {
        return 30; // 大于ErrorFilter的值
    }

    @Override
    public boolean shouldFilter() {
        // TODO 判断：仅处理来自post过滤器引起的异常
        return true;
    }

}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

到这里，我们在过滤器调度上的实现思路已经很清晰了，但是又有一个问题出现在我们面前：怎么判断引起异常的过滤器是来自什么阶段呢？（`shouldFilter`方法该如何实现）对于这个问题，我们第一反应会寄希望于请求上下文`RequestContext`对象，可是在查阅文档和源码后发现其中并没有存储异常来源的内容，所以我们不得不扩展原来的过滤器处理逻辑，当有异常抛出的时候，记录下抛出异常的过滤器，这样我们就可以在`ErrorExtFilter`过滤器的`shouldFilter`方法中获取并以此判断异常是否来自`post`阶段的过滤器了。

为了扩展过滤器的处理逻辑，为请求上下文增加一些自定义属性，我们需要深入了解一下Zuul过滤器的核心处理器：`com.netflix.zuul.FilterProcessor`。该类中定义了下面过滤器调用和处理相关的核心方法：到这里，我们在过滤器调度上的实现思路已经很清晰了，但是又有一个问题出现在我们面前：怎么判断引起异常的过滤器是来自什么阶段呢？（`shouldFilter`方法该如何实现）对于这个问题，我们第一反应会寄希望于请求上下文`RequestContext`对象，可是在查阅文档和源码后发现其中并没有存储异常来源的内容，所以我们不得不扩展原来的过滤器处理逻辑，当有异常抛出的时候，记录下抛出异常的过滤器，这样我们就可以在`ErrorExtFilter`过滤器的`shouldFilter`方法中获取并以此判断异常是否来自`post`阶段的过滤器了。

- `getInstance()`：该方法用来获取当前处理器的实例
- `setProcessor(FilterProcessor processor)`：该方法用来设置处理器实例，可以使用此方法来设置自定义的处理器
- `processZuulFilter(ZuulFilter filter)`：该方法定义了用来执行`filter`的具体逻辑，包括对请求上下文的设置，判断是否应该执行，执行时一些异常处理等
- `getFiltersByType(String filterType)`：该方法用来根据传入的`filterType`获取API网关中对应类型的过滤器，并根据这些过滤器的`filterOrder`从小到大排序，组织成一个列表返回
- `runFilters(String sType)`：该方法会根据传入的`filterType`来调用`getFiltersByType(String filterType)`获取排序后的过滤器列表，然后轮询这些过滤器，并调用`processZuulFilter(ZuulFilter filter)`来依次执行它们
- `preRoute()`：调用`runFilters("pre")`来执行所有`pre`类型的过滤器
- `route()`：调用`runFilters("route")`来执行所有`route`类型的过滤器
- `postRoute()`：调用`runFilters("post")`来执行所有`post`类型的过滤器
- `error()`：调用`runFilters("error")`来执行所有`error`类型的过滤器

根据我们之前的设计，我们可以直接通过扩展`processZuulFilter(ZuulFilter filter)`方法，当过滤器执行抛出异常的时候，我们捕获它，并往请求上下中记录一些信息。比如下面的具体实现：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
package com.dxz.zuul;

import com.netflix.zuul.FilterProcessor;
import com.netflix.zuul.ZuulFilter;
import com.netflix.zuul.context.RequestContext;
import com.netflix.zuul.exception.ZuulException;

public class DidiFilterProcessor extends FilterProcessor {

    @Override
    public Object processZuulFilter(ZuulFilter filter) throws ZuulException {
        try {
            return super.processZuulFilter(filter);
        } catch (ZuulException e) {
            RequestContext ctx = RequestContext.getCurrentContext();
            ctx.set("failed.filter", filter);
            throw e;
        }
    }

}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

在上面代码的实现中，我们创建了一个`FilterProcessor`的子类，并重写了`processZuulFilter(ZuulFilter filter)`，虽然主逻辑依然使用了父类的实现，但是在最外层，我们为其增加了异常捕获，并在异常处理中为请求上下文添加了`failed.filter`属性，以存储抛出异常的过滤器实例。在实现了这个扩展之后，我们也就可以完善之前`ErrorExtFilter`中的`shouldFilter()`方法，通过从请求上下文中获取该信息作出正确的判断，具体实现如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
package com.dxz.zuul;

import org.springframework.cloud.netflix.zuul.filters.post.SendErrorFilter;
import org.springframework.stereotype.Component;

import com.netflix.zuul.ZuulFilter;
import com.netflix.zuul.context.RequestContext;

@Component
public class ErrorExtFilter extends SendErrorFilter {

    @Override
    public String filterType() {
        return "error";
    }

    @Override
    public int filterOrder() {
        return 30; // 大于ErrorFilter的值
    }

    @Override
    public boolean shouldFilter() {
        // 判断：仅处理来自post过滤器引起的异常
        RequestContext ctx = RequestContext.getCurrentContext();
        ZuulFilter failedFilter = (ZuulFilter) ctx.get("failed.filter");
        if (failedFilter != null && failedFilter.filterType().equals("post")) {
            return true;
        }
        return false;
    }

}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 

到这里，我们的优化任务还没有完成，因为扩展的过滤器处理类并还没有生效。最后，我们需要在应用主类中，通过调用`FilterProcessor.setProcessor(new DidiFilterProcessor());`方法来启用自定义的核心处理器以完成我们的优化目标。