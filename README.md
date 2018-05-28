# WebAPI
WebAPI 学习
本项目的博客地址http://www.cnblogs.com/gangtianci/

http://www.cnblogs.com/UliiAn/p/5402146.html



   首先，我们实现一个AuthorizatoinFilter可以用以简单的权限控制：
public class AuthFilterAttribute : AuthorizationFilterAttribute
    {
        public override void OnAuthorization(HttpActionContext actionContext)
        {
            //如果用户方位的Action带有AllowAnonymousAttribute，则不进行授权验证
            if (actionContext.ActionDescriptor.GetCustomAttributes<AllowAnonymousAttribute>().Any())
            {
                return;
            }
            var verifyResult = actionContext.Request.Headers.Authorization!=null &&  //要求请求中需要带有Authorization头
                               actionContext.Request.Headers.Authorization.Parameter == "123456"; //并且Authorization参数为123456则验证通过

            if (!verifyResult)
            {
                //如果验证不通过，则返回401错误，并且Body中写入错误原因
                actionContext.Response = actionContext.Request.CreateErrorResponse(HttpStatusCode.Unauthorized,new HttpError("Token 不正确"));
            }
        }
    }


    一个简单的用于用户验证的Filter就开发完了，这个Filter要求用户的请求中带有Authorization头并且参数为123456，如果通过则放行，不通过则返回401错误，并在Content中提示Token不正确。下面，我们需要注册这个Filter，注册Filter有三种方法：

第一种：在我们希望进行权限控制的Action上打上AuthFilterAttribute这个Attribute:

public class PersonController : ApiController
    {
        [AuthFilter]
        public CreateResult Post(CreateUser user)
        {
            return new CreateResult() {Id = "123"};
        }
    }

这种方式适合单个Action的权限控制。

第二种，找到相应的Controller，并打上这个Attribute：



[AuthFilter]
    public class PersonController : ApiController
    {
        public CreateResult Post(CreateUser user)
        {
            return new CreateResult() {Id = "123"};
        }
    }

这种方式适合于控制整个Controller，打上这个Attribute以后，整个Controller里所有Action都获得了权限控制。

第三种，找到App_Start\WebApiConfig.cs，在Register方法下加入Filter实例：


public static void Register(HttpConfiguration config)
{
     config.MapHttpAttributeRoutes();
　　  //注册全局Filter
     config.Filters.Add(new AuthFilterAttribute());

     config.Routes.MapHttpRoute(
                name: "DefaultApi",
                routeTemplate: "api/{controller}/{id}",
                defaults: new { id = RouteParameter.Optional }
            );
}

用这种方式适合于控制所有的API，任意Controller和任意Action都接受了这个权限控制。

在大多数场景中，每个API的权限验证逻辑都是一样的，在这样的前提下使用全局注册Filter的方法最为简单便捷，可这样存在一个显而易见的问题：如果某几个API是不需要控制的（例如登录）怎么办？我们可以在这样的API上做这样的处理：

[AllowAnonymous]
public CreateResult PostLogin(LoginEntity entity)
{
      //TODO:添加验证逻辑
      return new CreateResult() {Id = "123456"};
}
我为这个Action打上了AllowAnonymousAttribute，验证逻辑就放过了这个API而不进行权限校验。

    在实际的开发中，我们可以设计一套类似Session的机制，通过用户登录来获取Token，在之后的交互HTTP请求中加上Authorization头并带上这个Token，并在自定义的AuthFilterAttribute中对Token进行验证，一套标准的Token验证流程就可以实现了。

    接下来我们介绍ActionFilter:

　　ActionFilterAttrubute提供了两个方法进行拦截：OnActionExecuting和OnActionExecuted，他们都提供了同步和异步的方法。

　　OnActionExecuting方法在Action执行之前执行，OnActionExecuted方法在Action执行完成之后执行。

　　我们来看一个应用场景：使用过MVC的同学一定不陌生MVC的模型绑定和模型校验，使用起来非常方便，定义好Entity之后，在需要进行校验的地方可以打上相应的Attribute，在Action开始时检查ModelState的IsValid属性，如果校验不通过直接返回View，前端可以解析并显示未通过校验的原因。而Web API中也继承了这一方便的特性，使用起来更加方便：   


 public class CustomActionFilterAttribute : ActionFilterAttribute
{
    public override void OnActionExecuting(HttpActionContext actionContext)
    {
        if (!actionContext.ModelState.IsValid)
        {
            actionContext.Response = actionContext.Request.CreateErrorResponse(HttpStatusCode.BadRequest,  actionContext.ModelState);
        }
    }
}

    这个Filter就提供了模型校验的功能，如果未通过模型校验则返回400错误，并把相关的错误信息交给调用者。他的使用方法和AuthFilterAttribute一样，可以针对Action、Controller、全局使用。我们可以用下面一个例子来验证：

代码如下：


public class LoginEntity
{
    [Required(ErrorMessage = "缺少用户名")]
    public string UserName { get; set; }

    [Required(ErrorMessage = "缺少密码")]
    public string Password { get; set; }
}

[AllowAnonymous]
[CustomActionFilter]
public CreateResult PostLogin(LoginEntity entity)
{
     //TODO:添加验证逻辑
     return new CreateResult() {Id = "123456"};
}

**********************************************************************************
陨石坑之webapi使用filter
   首先为什么说这是一个坑，是因为我们在webapi中使用filter的时候也许会先百度一下，好吧，挖坑的来了，我看了好几篇文章写的是使用System.Web.Mvc.Filters.ActionFilterAttribute。

然后开始痛苦的调试，发现这个过滤器永远调不进来(windows azure mobile services除外)。then.... 还是Google吧 ！

   痛苦后才懂，原来不是这么一回事，ActionFilterAttribute 有2个不同的版本，一个在System.Web.Mvc空间下，另外一个则在System.Web.Http.Filters命名空间下。他们有何区别？

The System.Web.Http one is for Web API; the System.Web.Mvc one is for previous MVC versions.

You can see from the source that the Web API version has several differences.

好吧，原来System.Web.Mvc.Filters.ActionFilterAttribute是给mvc用的，我们要用System.Web.Http.Filters下的，知道这样了 就开始了改写过程....，运行调试，发现异常！！！

先看下异常代码：


public class FilterConfig
  {
      public static void RegisterGlobalFilters(GlobalFilterCollection filters)
      {
          filters.Add(new HandleErrorAttribute());
          filters.Add(new PushFilter());
      }
  }
 

“System.InvalidOperationException”类型的异常在 System.Web.Mvc.dll 中发生，但未在用户代码中进行处理

其他信息: 给定的筛选器实例必须实现以下一个或多个筛选器接口: System.Web.Mvc.IAuthorizationFilter、System.Web.Mvc.IActionFilter、System.Web.Mvc.IResultFilter、System.Web.Mvc.IExceptionFilter、System.Web.Mvc.Filters.IAuthenticationFilter。
这是为何呢。。。 明明就是这个过滤器，为什么还是会有异常？ 原来问题在FilterConfig 这个类里面，这个类只是对MVC配置起效。（汗！！！！！！）,我们加过滤器的代码要加入到webapi的配置而非mvc的配置，so 代码要这么写。

复制代码
    public static class WebApiConfig
    {
        public static void Register(HttpConfiguration config)
        {
            config.MapHttpAttributeRoutes();
            config.Routes.MapHttpRoute(
                name: "push.api.v1",
                routeTemplate: "v1/{controller}/{id}",
                defaults: new { id = RouteParameter.Optional }
            );

           config.Filters.Add(new PushFilter());
        }
    }
复制代码
运行调试，ok，success!
参考 https://www.cnblogs.com/shi-meng/p/4635571.html
