---
title: 启用 ASP.NET Core 中的跨域请求 (CORS)
author: rick-anderson
description: 了解如何作为一种标准允许或拒绝在 ASP.NET Core 应用中的跨域请求的 CORS。
ms.author: riande
ms.custom: mvc
ms.date: 02/27/2019
uid: security/cors
ms.openlocfilehash: 2cad26d0f61519f63888a2bc399bb7e8a0f1ee04
ms.sourcegitcommit: 5f299daa7c8102d56a63b214b9a34cc4bc87bc42
ms.translationtype: MT
ms.contentlocale: zh-CN
ms.lasthandoff: 03/19/2019
ms.locfileid: "58210127"
---
# <a name="enable-cross-origin-requests-cors-in-aspnet-core"></a>启用 ASP.NET Core 中的跨域请求 (CORS)

作者：[Rick Anderson](https://twitter.com/RickAndMSFT)

本文介绍如何在 ASP.NET Core 应用中启用 CORS。

浏览器安全性阻止网页发出请求到不同的域的 web 页面以提供服务。 此限制称为*同域策略*。 同域策略可阻止恶意站点读取另一个站点中的敏感数据。 有时，你可能想要允许其他站点对您的应用程序进行跨域请求。 有关详细信息，请参阅[Mozilla CORS 文章](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)。

[跨源资源共享](https://www.w3.org/TR/cors/)(CORS):

* 是一项 W3C 标准，让服务器放宽同域策略。
* **不**是一项安全功能，CORS 降低了安全性。 允许 CORS 会使得API 更不安全。 有关详细信息，请参阅[ CORS 如何工作](#how-cors)。
* 显式允许某些跨域请求的同时，拒绝其他的一些服务器。
* 比早期技术更安全、 更灵活，例如[JSONP](/dotnet/framework/wcf/samples/jsonp)。

[查看或下载示例代码](https://github.com/aspnet/Docs/tree/master/aspnetcore/tutorials/razor-pages/razor-pages-start/2.2-stage-samples)（[如何下载](xref:index#how-to-download-a-sample)）

## <a name="same-origin"></a>相同的来源

如果它们具有相同方案、 主机和端口，两个 Url 将具有相同的来源 ([RFC 6454](https://tools.ietf.org/html/rfc6454))。

以下两个 Url 具有相同的来源：

* `https://example.com/foo.html`
* `https://example.com/bar.html`

这些 Url 具有不同的源，比上述的两个 Url:

* `https://example.net` &ndash; 不同的域
* `https://www.example.com/foo.html` &ndash; 不同的子域
* `http://example.com/foo.html` &ndash; 不同的方案
* `https://example.com:9000/foo.html` &ndash; 不同的端口

比较来源时，Internet Explorer 不会考虑端口。

## <a name="cors-with-named-policy-and-middleware"></a>使用命名的策略和中间件的 CORS

CORS 中间件处理跨域请求。 下面的代码让整个应用对指定来源启用 CORS:

[!code-csharp[](cors/sample/Cors/WebAPI/Startup.cs?name=snippet&highlight=8,14-23,38)]

前面的代码：

* 策略名称设置为"\_myAllowSpecificOrigins"。 策略名称是任意的。
* 调用<xref:Microsoft.AspNetCore.Builder.CorsMiddlewareExtensions.UseCors*>扩展方法，使内核可用。
* 调用<xref:Microsoft.Extensions.DependencyInjection.CorsServiceCollectionExtensions.AddCors*>与[lambda 表达式](/dotnet/csharp/programming-guide/statements-expressions-operators/lambda-expressions)。 Lambda 采用<xref:Microsoft.AspNetCore.Cors.Infrastructure.CorsPolicyBuilder>对象。 [配置选项](#cors-policy-options)，如`WithOrigins`，本文稍后介绍。

<xref:Microsoft.Extensions.DependencyInjection.MvcCorsMvcCoreBuilderExtensions.AddCors*>方法调用将 CORS 服务添加到应用程序的服务容器：

[!code-csharp[](cors/sample/Cors/WebAPI/Startup.cs?name=snippet2)]

有关详细信息，请参阅本文档中[CORS 策略选项](#cpo)。

<xref:Microsoft.AspNetCore.Cors.Infrastructure.CorsPolicyBuilder>方法可以链接方法，如下面的代码中所示：

[!code-csharp[](cors/sample/Cors/WebAPI/Startup2.cs?name=snippet2)]

以下突出显示的代码将使得 CORS 策略应用于所有应用程序终端通过 CORS 中间件：

```csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }
    else
    {
        app.UseHsts();
    }

    app.UseCors();

    app.UseHttpsRedirection();
    app.UseMvc();
}
```

请参阅[Razor 页面、 控制器和操作方法中启用 CORS](#ecors)要应用页/控制器/操作级的 CORS 策略。

注意:

* `UseCors` 必须在`UseMvc` 之前调用。
* URL 必须**不**包含尾部反斜杠 (`/`)。 如果 URL 以`/`终止，则比较操作返回`false`并返回无标头。

请参阅[测试 CORS](#test)有关测试前面的代码中的说明。

<a name="ecors"></a>

## <a name="enable-cors-with-attributes"></a>使用属性启用 CORS

[ &lbrack;EnableCors&rbrack; ](xref:Microsoft.AspNetCore.Cors.EnableCorsAttribute)属性提供了一种全局应用 CORS 的替代方法。 `[EnableCors]`属性为所选的终端，而不是所有终端启用 CORS。

使用`[EnableCors]`可以指定默认策略，而`[EnableCors("{Policy String}")]`可以指定特定策略。

`[EnableCors]`特性可以应用于：

* Razor 页面 `PageModel`
* 控制器
* 控制器操作方法

`[EnableCors]`属性可以将不同的策略应用于Controller/page-model/action。 当`[EnableCors]`特性应用到Controller/page-model/action，而且中间件中启用了 CORS，这些策略都会得到应用。 我们建议不要合并策略。 不能同时在同一应用中使用`[EnableCors]`属性或中间件，。

下面的代码将不同的策略应用到每个方法：

[!code-csharp[](cors/sample/Cors/WebAPI/Controllers/WidgetController.cs?name=snippet&highlight=6,14)]

以下代码将创建一个 CORS 默认策略和一个名为`"AnotherPolicy"`的策略:

[!code-csharp[](cors/sample/Cors/WebAPI/StartupMultiPolicy.cs?name=snippet&highlight=12-28)]

### <a name="disable-cors"></a>禁用 CORS

[ &lbrack;DisableCors&rbrack; ](xref:Microsoft.AspNetCore.Cors.DisableCorsAttribute)属性让控制器/页的模型/操作禁用 CORS。

<a name="cpo"></a>

## <a name="cors-policy-options"></a>CORS 策略选项

本部分介绍可以在 CORS 策略设置的各种选项：

* [设置允许的来源](#set-the-allowed-origins)
* [设置允许的 HTTP 方法](#set-the-allowed-http-methods)
* [设置允许的请求标头](#set-the-allowed-request-headers)
* [设置公开的响应标头](#set-the-exposed-response-headers)
* [跨域请求中的凭据](#credentials-in-cross-origin-requests)
* [将预检过期时间设置](#set-the-preflight-expiration-time)

<xref:Microsoft.AspNetCore.Cors.Infrastructure.CorsOptions.AddPolicy*> 在`Startup.ConfigureServices`被调用。 有关一些选项，阅读[ CORS 如何工作](#how-cors)可能会有帮助。

## <a name="set-the-allowed-origins"></a>设置允许的来源

<xref:Microsoft.AspNetCore.Cors.Infrastructure.CorsPolicyBuilder.AllowAnyOrigin*> &ndash; 允许来自所有来源的使用任何方案(`http`或`https`)的 CORS 请求 。 `AllowAnyOrigin` 不安全，因为*任何网站*都可以对应用进行跨域请求。

::: moniker range=">= aspnetcore-2.2"

> [!NOTE]
> 指定`AllowAnyOrigin`和`AllowCredentials`是不安全的配置，可能会导致跨站点请求伪造。 同时使用这两种方法配置应用程序时，CORS 服务将返回无效的 CORS 响应。

::: moniker-end

::: moniker range="< aspnetcore-2.2"

> [!NOTE]
> 指定`AllowAnyOrigin`和`AllowCredentials`是一个不安全的配置，可能会导致跨站点请求伪造。 有关安全的应用程序，如果客户端必须先授权本身访问服务器资源指定确切的来源列表。

::: moniker-end

<!-- REVIEW required
I changed from
Specifying `AllowAnyOrigin` and `AllowCredentials` is an insecure configuration. **This** setting affects preflight requests and the ...
to
**`AllowAnyOrigin`** affects preflight requests and the

to remove the ambiguous **This**.
-->

`AllowAnyOrigin` 影响预检请求和`Access-Control-Allow-Origin`标头。 有关详细信息，请参阅[预检请求](#preflight-requests)部分。

::: moniker range=">= aspnetcore-2.0"

<xref:Microsoft.AspNetCore.Cors.Infrastructure.CorsPolicyBuilder.SetIsOriginAllowedToAllowWildcardSubdomains*> &ndash; 设置策略的<xref:Microsoft.AspNetCore.Cors.Infrastructure.CorsPolicy.IsOriginAllowed*>属性，以评估是否通过一个匹配配置的通配域的来源。

[!code-csharp[](cors/sample/CorsExample4/Startup.cs?range=100-104&highlight=4)]

::: moniker-end

### <a name="set-the-allowed-http-methods"></a>设置允许的 HTTP 方法

<xref:Microsoft.AspNetCore.Cors.Infrastructure.CorsPolicyBuilder.AllowAnyMethod*>：

* 允许任何 HTTP 方法：
* 影响预检请求和`Access-Control-Allow-Methods`标头。 有关详细信息，请参阅[预检请求](#preflight-requests)部分。

### <a name="set-the-allowed-request-headers"></a>设置允许的请求标头

若要允许特定标头发送在 CORS 请求中，调用*创作请求标头*，调用<xref:Microsoft.AspNetCore.Cors.Infrastructure.CorsPolicyBuilder.WithHeaders*>并指定允许的标头：

[!code-csharp[](cors/sample/CorsExample4/Startup.cs?range=55-60&highlight=5)]

若要允许所有作者的请求标头调用<xref:Microsoft.AspNetCore.Cors.Infrastructure.CorsPolicyBuilder.AllowAnyHeader*>:

[!code-csharp[](cors/sample/CorsExample4/Startup.cs?range=64-69&highlight=5)]

此设置会影响的预检请求和`Access-Control-Request-Headers`标头。 有关详细信息，请参阅[预检请求](#preflight-requests)部分。

::: moniker range=">= aspnetcore-2.2"

CORS 中间件策略匹配所指定的特定标头`WithHeaders`时，才可以发送标头`Access-Control-Request-Headers`与中所述的标头完全匹配`WithHeaders`。

例如，假设某应用配置如下：

```csharp
app.UseCors(policy => policy.WithHeaders(HeaderNames.CacheControl));
```

CORS 中间件拒绝包含以下请求标头的预检请求，因为`Content-Language`([HeaderNames.ContentLanguage](xref:Microsoft.Net.Http.Headers.HeaderNames.ContentLanguage)) 中未列出`WithHeaders`:

```
Access-Control-Request-Headers: Cache-Control, Content-Language
```

应用程序返回*200 确定*响应但不会发送回的 CORS 标头。 因此，在浏览器不会尝试执行跨域请求。

::: moniker-end

::: moniker range="< aspnetcore-2.2"

CORS 中间件始终允许在四个标头`Access-Control-Request-Headers`发送而不考虑 CorsPolicy.Headers 中配置的值。 此列表的标头包括：

* `Accept`
* `Accept-Language`
* `Content-Language`
* `Origin`

例如，假设配置，如下所示的应用：

```csharp
app.UseCors(policy => policy.WithHeaders(HeaderNames.CacheControl));
```

CORS 中间件成功响应以下请求标头的预检请求因为`Content-Language`始终是已列入允许列表：

```
Access-Control-Request-Headers: Cache-Control, Content-Language
```

::: moniker-end

### <a name="set-the-exposed-response-headers"></a>设置公开的响应标头

默认情况下，浏览器不会向应用程序公开所有的响应标头。 有关详细信息，请参阅[W3C 跨域资源共享 （术语）：简单的响应标头](https://www.w3.org/TR/cors/#simple-response-header)。

默认情况下可用的响应标头是：

* `Cache-Control`
* `Content-Language`
* `Content-Type`
* `Expires`
* `Last-Modified`
* `Pragma`

CORS 规范称呼这些标头为*简单响应标头*。 若要使其他标头可用于应用程序，请调用<xref:Microsoft.AspNetCore.Cors.Infrastructure.CorsPolicyBuilder.WithExposedHeaders*>:

[!code-csharp[](cors/sample/CorsExample4/Startup.cs?range=73-78&highlight=5)]

### <a name="credentials-in-cross-origin-requests"></a>跨域请求中的凭据

在一个 CORS 请求中，凭据要求特殊处理。 默认情况下，在浏览器不会发送跨域请求的凭据。 凭据包括 cookie 和 HTTP 身份验证方案。 若要将发送具有跨源请求的凭据，客户端必须设置`XMLHttpRequest.withCredentials`为`true`。

使用`XMLHttpRequest`直接：

```javascript
var xhr = new XMLHttpRequest();
xhr.open('get', 'https://www.example.com/api/test');
xhr.withCredentials = true;
```

使用 jQuery:

```javascript
$.ajax({
  type: 'get',
  url: 'https://www.example.com/api/test',
  xhrFields: {
    withCredentials: true
  }
});
```

使用[Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API):

```javascript
fetch('https://www.example.com/api/test', {
    credentials: 'include'
});
```

服务器必须允许凭据。 若要允许跨域凭据，调用<xref:Microsoft.AspNetCore.Cors.Infrastructure.CorsPolicyBuilder.AllowCredentials*>:

[!code-csharp[](cors/sample/CorsExample4/Startup.cs?range=82-87&highlight=5)]

HTTP 响应包括`Access-Control-Allow-Credentials`标头，这会告诉浏览器，服务器允许跨域请求凭据。

如果浏览器发送凭据，但响应不包含有效`Access-Control-Allow-Credentials`标头，则浏览器不会对应用程序公开响应，跨域请求失败。

允许跨域凭据是一个安全风险。 在另一个域的网站可以在用户不知情的情况下以的用户的名义发送已登录用户的凭据。 <!-- TODO Review: When using `AllowCredentials`, all CORS enabled domains must be trusted.
I don't like "all CORS enabled domains must be trusted", because it implies that if you're not using  `AllowCredentials`, domains don't need to be trusted. -->

CORS 规范同时声明设置来源为`"*"`（允许所有来源） 是无效的，如果设置了`Access-Control-Allow-Credentials`标头的话。

### <a name="preflight-requests"></a>预检请求

对于某些 CORS 请求，在浏览器发出实际请求之前发送额外请求。 此请求称为*预检请求*。 如果满足以下条件，则浏览器可以跳过预检请求：

* 请求方法是 GET、 HEAD 或 POST。
* 应用程序没有设置`Accept`， `Accept-Language`， `Content-Language`， `Content-Type`，或`Last-Event-ID`以外的其他请求标头。
* `Content-Type`标头，如果设置了，具有以下值之一：
  * `application/x-www-form-urlencoded`
  * `multipart/form-data`
  * `text/plain`

在请求标头的规则集的客户端请求适用于通过调用来设置应用程序的标头`setRequestHeader`上`XMLHttpRequest`对象。 CORS 规范称呼这些标头为*创作请求标头*。 该规则不能应用于标头设置可以在浏览器，如`User-Agent`， `Host`，或`Content-Length`。

下面是预检请求的示例：

```
OPTIONS https://myservice.azurewebsites.net/api/test HTTP/1.1
Accept: */*
Origin: https://myclient.azurewebsites.net
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: accept, x-my-custom-header
Accept-Encoding: gzip, deflate
User-Agent: Mozilla/5.0 (compatible; MSIE 10.0; Windows NT 6.2; WOW64; Trident/6.0)
Host: myservice.azurewebsites.net
Content-Length: 0
```

预检请求使用 HTTP OPTIONS 方法。 它包括两个特殊的标头：

* `Access-Control-Request-Method`：将用于实际请求的 HTTP 方法。
* `Access-Control-Request-Headers`：应用程序设置实际请求的请求标头的列表。 如前面所述，这不包括标头的浏览器设置，如`User-Agent`。

CORS 预检请求可能包括`Access-Control-Request-Headers`标头，指示服务器与实际请求一起发送的标头。

若要允许的特定标头，请调用<xref:Microsoft.AspNetCore.Cors.Infrastructure.CorsPolicyBuilder.WithHeaders*>:

[!code-csharp[](cors/sample/CorsExample4/Startup.cs?range=55-60&highlight=5)]

若要允许所有作者的请求标头调用<xref:Microsoft.AspNetCore.Cors.Infrastructure.CorsPolicyBuilder.AllowAnyHeader*>:

[!code-csharp[](cors/sample/CorsExample4/Startup.cs?range=64-69&highlight=5)]

浏览器不是在设置方式完全一致`Access-Control-Request-Headers`。 如果将标头设置到的任何内容以外`"*"`(或使用<xref:Microsoft.AspNetCore.Cors.Infrastructure.CorsPolicy.AllowAnyHeader*>)，至少应包含`Accept`， `Content-Type`，和`Origin`，以及你想要支持任何自定义标头。

下面是预检请求 （假设服务器允许请求） 的示例响应：

```
HTTP/1.1 200 OK
Cache-Control: no-cache
Pragma: no-cache
Content-Length: 0
Access-Control-Allow-Origin: https://myclient.azurewebsites.net
Access-Control-Allow-Headers: x-my-custom-header
Access-Control-Allow-Methods: PUT
Date: Wed, 20 May 2015 06:33:22 GMT
```

响应包括`Access-Control-Allow-Methods`标头，其中列出了允许的方法和 （可选）`Access-Control-Allow-Headers`标头，其中列出了允许的标头。 如果预检请求成功，则浏览器发送实际请求。

如果预检请求被拒绝，该应用程序将返回*200 确定*响应但不会发送回的 CORS 标头。 因此，在浏览器不会尝试执行跨域请求。

### <a name="set-the-preflight-expiration-time"></a>将预检过期时间设置

`Access-Control-Max-Age`标头指定可以缓存预检请求的响应的时间。 若要设置此标头，请调用<xref:Microsoft.AspNetCore.Cors.Infrastructure.CorsPolicyBuilder.SetPreflightMaxAge*>:

[!code-csharp[](cors/sample/CorsExample4/Startup.cs?range=91-96&highlight=5)]

<a name="how-cors"></a>

## <a name="how-cors-works"></a>CORS 的工作原理

本部分介绍了中会发生什么情况[CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)请求在 HTTP 消息的级别。

* CORS 是**不**一项安全功能。 CORS 是一项 W3C 标准，可让服务器放宽同域策略。
  * 例如，可以使用恶意行动者[防止跨站点脚本 (XSS)](xref:security/cross-site-scripting)针对你的站点并执行对其启用 CORS 的站点的跨站点请求窃取信息。
* 你的 API 不是更安全，从而 CORS。
  * 它由客户端 （浏览器） 强制实施 CORS。 服务器执行该请求，并返回响应，而是返回错误和块的响应的客户端。 例如，以下任何工具将显示服务器响应：
    * [Fiddler](https://www.telerik.com/fiddler)
    * [Postman](https://www.getpostman.com/)
    * [.NET HttpClient](/dotnet/csharp/tutorials/console-webapiclient)
    * 通过在地址栏中输入 URL web 浏览器。
* 它是服务器以允许浏览器执行跨域的方法[XHR](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest)或[提取 API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)否则会被禁止的请求。
  * （不 CORS) 的浏览器不能执行跨域请求。 CORS 前, [JSONP](https://www.w3schools.com/js/js_json_jsonp.asp)用于绕过此限制。 JSONP 不会使用 XHR，它使用`<script>`标记接收的响应。 允许脚本要加载的跨域。

[CORS 规范](https://www.w3.org/TR/cors/)引入了几个新的 HTTP 标头启用跨域请求。 如果浏览器支持 CORS，则将设置自动跨域请求这些标头。 自定义 JavaScript 代码不需要启用 CORS。

下面是跨域请求的示例。 `Origin`标头提供发出请求的站点的域：

```
GET https://myservice.azurewebsites.net/api/test HTTP/1.1
Referer: https://myclient.azurewebsites.net/
Accept: */*
Accept-Language: en-US
Origin: https://myclient.azurewebsites.net
Accept-Encoding: gzip, deflate
User-Agent: Mozilla/5.0 (compatible; MSIE 10.0; Windows NT 6.2; WOW64; Trident/6.0)
Host: myservice.azurewebsites.net
```

如果服务器允许该请求，它会设置`Access-Control-Allow-Origin`在响应中的标头。 此标头的值或者与匹配`Origin`请求标头或为通配符值`"*"`，允许任何源的含义：

```
HTTP/1.1 200 OK
Cache-Control: no-cache
Pragma: no-cache
Content-Type: text/plain; charset=utf-8
Access-Control-Allow-Origin: https://myclient.azurewebsites.net
Date: Wed, 20 May 2015 06:27:30 GMT
Content-Length: 12

Test message
```

如果响应不包括`Access-Control-Allow-Origin`标头，跨域请求会失败。 具体而言，在浏览器不允许该请求。 即使服务器返回成功的响应，请在浏览器不使响应可供客户端应用程序。

<a name="test"></a>

## <a name="test-cors"></a>测试 CORS

若要测试 CORS:

1. [创建 API 项目](xref:tutorials/first-web-api)。 此外，也可以[下载示例](https://github.com/aspnet/Docs/tree/master/aspnetcore/security/cors/sample/Cors)。
1. 启用 CORS 的方法之一使用本文档中。 例如：

  [!code-csharp[](cors/sample/Cors/WebAPI/StartupTest.cs?name=snippet2&highlight=13-18)]

  > [!WARNING]
  > `WithOrigins("https://localhost:<port>");` 应仅用于测试示例应用类似于[下载示例代码](https://github.com/aspnet/Docs/tree/live/aspnetcore/security/cors/sample/Cors)。

1. 创建 web 应用程序项目 （Razor 页面或 MVC）。 此示例使用 Razor 页面。 可以为 API 项目在同一解决方案中创建 web 应用。
1. 将以下突出显示的代码添加到*Index.cshtml*文件：

  [!code-csharp[](cors/sample/Cors/ClientApp/Pages/Index2.cshtml?highlight=7-99)]

1. 在上述代码中，将为`url: 'https://<web app>.azurewebsites.net/api/values/1',`已部署的应用的 url。
1. 部署 API 项目。 例如，[部署到 Azure](xref:host-and-deploy/azure-apps/index)。
1. 从桌面上运行的 Razor 页面或 MVC 应用并单击**测试**按钮。 使用 F12 工具来查看错误消息。
1. 删除从 localhost 原点`WithOrigins`并将其部署应用程序。 此外，运行客户端应用程序，使用不同的端口。 例如，从 Visual Studio 中运行。
1. 使用客户端应用进行测试。 CORS 故障返回错误，但错误消息不可用到 JavaScript。 在 F12 工具中使用控制台选项卡查看该错误。 具体取决于浏览器中，你收到的错误 （中的 F12 工具控制台） 类似于下面：

   * 使用 Microsoft Edge:

     **SEC7120: CORS 原点`https://localhost:44375`找不到`https://localhost:44375`处的跨域资源的访问控制的允许的源响应标头中 `https://webapi.azurewebsites.net/api/values/1`**

   * 使用 Chrome:

     **访问在 XMLHttpRequest`https://webapi.azurewebsites.net/api/values/1`来源`https://localhost:44375`CORS 策略阻止：请求的资源上存在没有访问控制的允许的域标头。**

## <a name="additional-resources"></a>其他资源

* [跨域资源共享 (CORS)](https://developer.mozilla.org/docs/Web/HTTP/CORS)
