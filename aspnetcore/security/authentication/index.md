---
title: ASP.NET Core 身份验证概述
author: mjrousos
description: 了解 ASP.NET Core 中的身份验证。
ms.author: riande
ms.custom: mvc
ms.date: 03/03/2020
no-loc:
- appsettings.json
- ASP.NET Core Identity
- cookie
- Cookie
- Blazor
- Blazor Server
- Blazor WebAssembly
- Identity
- Let's Encrypt
- Razor
- SignalR
uid: security/authentication/index
ms.openlocfilehash: e9e4ca11d20557666c75b84e56af825d002df0f1
ms.sourcegitcommit: fbd5427293d9ecccc388bd5fd305c2eb8ada7281
ms.translationtype: HT
ms.contentlocale: zh-CN
ms.lasthandoff: 11/10/2020
ms.locfileid: "94463998"
---
# <a name="overview-of-aspnet-core-authentication"></a>ASP.NET Core 身份验证概述

作者：[Mike Rousos](https://github.com/mjrousos)

身份验证是确定用户身份的过程。 [授权](xref:security/authorization/introduction)是确定用户是否有权访问资源的过程。 在 ASP.NET Core 中，身份验证由 `IAuthenticationService` 负责，而它供身份验证[中间件](xref:fundamentals/middleware/index)使用。 身份验证服务会使用已注册的身份验证处理程序来完成与身份验证相关的操作。 与身份验证相关的操作示例包括：

* 对用户进行身份验证。
* 在未经身份验证的用户试图访问受限资源时作出响应。

已注册的身份验证处理程序及其配置选项被称为“方案”。

身份验证方案由 `Startup.ConfigureServices` 中的注册身份验证服务指定：

* 方式是在调用 `services.AddAuthentication` 后调用方案特定的扩展方法（例如 `AddJwtBearer` 或 `AddCookie`）。 这些扩展方法使用 [AuthenticationBuilder.AddScheme](xref:Microsoft.AspNetCore.Authentication.AuthenticationBuilder.AddScheme*) 向适当的设置注册方案。
* 比较不常用的方式是直接调用 [AuthenticationBuilder.AddScheme](xref:Microsoft.AspNetCore.Authentication.AuthenticationBuilder.AddScheme*)。

例如，下列代码会为 cookie 和 JWT 持有者身份验证方案注册身份验证服务和处理程序：

```csharp
services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(JwtBearerDefaults.AuthenticationScheme, options => Configuration.Bind("JwtSettings", options))
    .AddCookie(CookieAuthenticationDefaults.AuthenticationScheme, options => Configuration.Bind("CookieSettings", options));
```

`AddAuthentication` 参数 `JwtBearerDefaults.AuthenticationScheme` 是方案的名称，未请求特定方案时会默认使用此名称。

如果使用了多个方案，授权策略（或授权属性）可[指定](xref:security/authorization/limitingidentitybyscheme)对用户进行身份验证时要依据的一个或多个身份验证方案。 在上例中，可通过指定 cookie 身份验证方案的名称来使用该方案（默认为 `CookieAuthenticationDefaults.AuthenticationScheme`，但也可在调用 `AddCookie` 时提供其他名称）。

在某些情况下，其他扩展方法会自动调用 `AddAuthentication`。 例如，在使用 [ASP.NET Core Identity](xref:security/authentication/identity) 时，会在内部调用 `AddAuthentication`。

通过在应用的 `IApplicationBuilder` 上调用 <xref:Microsoft.AspNetCore.Builder.AuthAppBuilderExtensions.UseAuthentication*> 扩展方法，在 `Startup.Configure` 中添加身份验证中间件。 如果调用 `UseAuthentication`，会注册使用之前注册的身份验证方案的中间节。 请在依赖于要进行身份验证的用户的所有中间件之前调用 `UseAuthentication`。 如果使用终结点路由，则必须按以下顺序调用 `UseAuthentication`：

* 在 `UseRouting`之后调用，以便路由信息可用于身份验证决策。
* 在 `UseEndpoints` 之前调用，以便用户在经过身份验证后才能访问终结点。

## <a name="authentication-concepts"></a>身份验证概念

### <a name="authentication-scheme"></a>身份验证方案

身份验证方案是与下列项相对应的名称：

* 身份验证处理程序。
* 用于配置处理程序的该特定实例的选项。

方案可用作一种机制，供用户参考相关处理程序的身份验证、质询和禁止行为。 例如，授权策略可使用方案名称来指定应使用哪种（或哪些）身份验证方案来对用户进行身份验证。 配置身份验证时，通常是指定默认身份验证方案。 除非资源请求了特定方案，否则使用默认方案。 还可：

* 指定其他默认方案供授权、质询和禁止操作使用。
* 可通过[策略方案](xref:security/authentication/policyschemes)将多个方案合成一个。

### <a name="authentication-handler"></a>身份验证处理程序

身份验证处理程序：

* 是一种实现方案行为的类型。
* 派生自 <xref:Microsoft.AspNetCore.Authentication.IAuthenticationHandler> 或 <xref:Microsoft.AspNetCore.Authentication.AuthenticationHandler`1>。
* 具有对用户进行身份验证的主要责任。

根据身份验证方案的配置和传入的请求上下文，身份验证处理程序：

* 构造表示用户身份的 <xref:Microsoft.AspNetCore.Authentication.AuthenticationTicket> 对象（若身份验证成功）。
* 返回“无结果”或“失败”（若身份验证失败）。
* 具有用于质询和禁止操作的方法，供用户在下述情况下访问资源时使用：
  * 他们未获得访问授权（禁止）。
  * 他们未经过身份验证（质询）。

### <a name="authenticate"></a>Authenticate

身份验证方案的身份验证操作负责根据请求上下文构造用户的身份。 它会返回一个 <xref:Microsoft.AspNetCore.Authentication.AuthenticateResult>指示身份验证是否成功；若成功，则还在身份验证票证中指示用户的身份。 请参阅 <xref:Microsoft.AspNetCore.Authentication.AuthenticationHttpContextExtensions.AuthenticateAsync%2A>。 身份验证示例包括：

* 根据 cookie 构造用户身份的 cookie 身份验证方案。
* 对 JWT 持有者令牌进行反序列化和验证以构造用户身份的 JWT 持有者方案。

### <a name="challenge"></a>质询

当未经身份验证的用户请求要求身份验证的终结点时，授权会发起身份验证质询。 例如，当匿名用户请求受限资源或单击登录链接时，会引发身份验证质询。 授权会使用指定的身份验证方案发起质询；如果未指定任何方案，则使用默认方案。 请参阅 <xref:Microsoft.AspNetCore.Authentication.AuthenticationHttpContextExtensions.ChallengeAsync%2A>。 身份验证质询示例包括：

* 将用户重定向到登录页面的 cookie 身份验证方案。
* 返回具有 `www-authenticate: bearer` 标头的 401 结果的 JWT 持有者方案。

质询操作应告知用户要使用哪种身份验证机制来访问所请求的资源。

### <a name="forbid"></a>禁止

当已经过身份验证的用户尝试访问其无权访问的资源时，授权会调用身份验证方案的禁止操作。 请参阅 <xref:Microsoft.AspNetCore.Authentication.AuthenticationHttpContextExtensions.ForbidAsync%2A>。 身份验证禁止示例包括：
* 将用户重定向到表示访问遭禁的页面的 cookie 身份验证方案。
* 返回 403 结果的 JWT 持有者方案。
* 重定向到用户可请求资源访问权限的页面的自定义身份验证方案。

用户可通过禁止操作知道：

* 他们已经过身份验证。
* 他们无权访问所请求的资源。

请查看以下链接，了解质询与禁止之间区别：

* [使用操作资源处理程序的质询和禁止](xref:security/authorization/resourcebased#challenge-and-forbid-with-an-operational-resource-handler)。
* [质询与禁止之间的区别](xref:security/authorization/secure-data#challenge)。

## <a name="authentication-providers-per-tenant"></a>每个租户的身份验证提供程序

ASP.NET Core 框架没有用于多租户身份验证的内置解决方案。
虽然客户当然可以使用内置功能编写一个解决方案，但建议客户查看 [Orchard Core](https://www.orchardcore.net/) 以实现此目的。

Orchard Core 是：

* 使用 ASP.NET Core 生成的开放源代码模块和多租户应用框架。
* 基于该应用框架生成的内容管理系统 (CMS)。

请参阅 [Orchard Core](https://github.com/OrchardCMS/OrchardCore) 源，了解每个租户的身份验证提供程序的示例。

## <a name="additional-resources"></a>其他资源

* <xref:security/authorization/limitingidentitybyscheme>
* <xref:security/authentication/policyschemes>
* <xref:security/authorization/secure-data>
* [全球要求经过身份验证的用户](xref:security/authorization/secure-data#rau)
* [GitHub 上关于使用多个身份验证方案的问题](https://github.com/dotnet/aspnetcore/issues/26002)
