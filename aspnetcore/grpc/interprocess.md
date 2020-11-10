---
title: 使用 gRPC 进行进程间通信
author: jamesnk
description: 了解如何使用 gRPC 进行进程间通信。
monikerRange: '>= aspnetcore-5.0'
ms.author: jamesnk
ms.date: 09/16/2020
no-loc:
- 'appsettings.json'
- 'ASP.NET Core Identity'
- 'cookie'
- 'Cookie'
- 'Blazor'
- 'Blazor Server'
- 'Blazor WebAssembly'
- 'Identity'
- "Let's Encrypt"
- 'Razor'
- 'SignalR'
uid: grpc/interprocess
ms.openlocfilehash: d806a340d8540fce8af6ccc6ff68325e4b733922
ms.sourcegitcommit: ca34c1ac578e7d3daa0febf1810ba5fc74f60bbf
ms.translationtype: HT
ms.contentlocale: zh-CN
ms.lasthandoff: 10/30/2020
ms.locfileid: "93059879"
---
# <a name="inter-process-communication-with-grpc"></a><span data-ttu-id="5c0de-103">使用 gRPC 进行进程间通信</span><span class="sxs-lookup"><span data-stu-id="5c0de-103">Inter-process communication with gRPC</span></span>

<span data-ttu-id="5c0de-104">作者：[James Newton-King](https://twitter.com/jamesnk)</span><span class="sxs-lookup"><span data-stu-id="5c0de-104">By [James Newton-King](https://twitter.com/jamesnk)</span></span>

<span data-ttu-id="5c0de-105">客户端和服务之间的 gRPC 调用通常通过 TCP 套接字发送。</span><span class="sxs-lookup"><span data-stu-id="5c0de-105">gRPC calls between a client and service are usually sent over TCP sockets.</span></span> <span data-ttu-id="5c0de-106">TCP 非常适用于网络中的通信。</span><span class="sxs-lookup"><span data-stu-id="5c0de-106">TCP was designed for communicating across a network.</span></span> <span data-ttu-id="5c0de-107">但当客户端和服务在同一台计算机上时，[进程间通信 (IPC)](https://wikipedia.org/wiki/Inter-process_communication) 的效率比 TCP 更高。</span><span class="sxs-lookup"><span data-stu-id="5c0de-107">[Inter-process communication (IPC)](https://wikipedia.org/wiki/Inter-process_communication) is more efficient than TCP when the client and service are on the same machine.</span></span> <span data-ttu-id="5c0de-108">本文档介绍如何在 IPC 场景中将 gRPC 用于自定义传输。</span><span class="sxs-lookup"><span data-stu-id="5c0de-108">This document explains how to use gRPC with custom transports in IPC scenarios.</span></span>

## <a name="server-configuration"></a><span data-ttu-id="5c0de-109">服务器配置</span><span class="sxs-lookup"><span data-stu-id="5c0de-109">Server configuration</span></span>

<span data-ttu-id="5c0de-110">[Kestrel](xref:fundamentals/servers/kestrel) 支持自定义传输。</span><span class="sxs-lookup"><span data-stu-id="5c0de-110">Custom transports are supported by [Kestrel](xref:fundamentals/servers/kestrel).</span></span> <span data-ttu-id="5c0de-111">在 Program.cs 上配置 Kestrel：</span><span class="sxs-lookup"><span data-stu-id="5c0de-111">Kestrel is configured in *Program.cs* :</span></span>

```csharp
public static readonly string SocketPath = Path.Combine(Path.GetTempPath(), "socket.tmp");

public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.UseStartup<Startup>();
            webBuilder.ConfigureKestrel(options =>
            {
                if (File.Exists(SocketPath))
                {
                    File.Delete(SocketPath);
                }
                options.ListenUnixSocket(SocketPath);
            });
        });
```

<span data-ttu-id="5c0de-112">上面的示例：</span><span class="sxs-lookup"><span data-stu-id="5c0de-112">The preceding example:</span></span>

* <span data-ttu-id="5c0de-113">在 `ConfigureKestrel` 中配置 Kestrel 的终结点。</span><span class="sxs-lookup"><span data-stu-id="5c0de-113">Configures Kestrel's endpoints in `ConfigureKestrel`.</span></span>
* <span data-ttu-id="5c0de-114">调用 <xref:Microsoft.AspNetCore.Server.Kestrel.Core.KestrelServerOptions.ListenUnixSocket*> 来侦听具有指定路径的 [Unix 域套接字 (UDS)](https://wikipedia.org/wiki/Unix_domain_socket)。</span><span class="sxs-lookup"><span data-stu-id="5c0de-114">Calls <xref:Microsoft.AspNetCore.Server.Kestrel.Core.KestrelServerOptions.ListenUnixSocket*> to listen to a [Unix domain socket (UDS)](https://wikipedia.org/wiki/Unix_domain_socket) with the specified path.</span></span>

<span data-ttu-id="5c0de-115">Kestrel 提供对 UDS 终结点的内置支持。</span><span class="sxs-lookup"><span data-stu-id="5c0de-115">Kestrel has built-in support for UDS endpoints.</span></span> <span data-ttu-id="5c0de-116">UDS 在 Linux、macOS 和 [Windows 的新式版本](https://devblogs.microsoft.com/commandline/af_unix-comes-to-windows/)上受支持。</span><span class="sxs-lookup"><span data-stu-id="5c0de-116">UDS are supported on Linux, macOS and [modern versions of Windows](https://devblogs.microsoft.com/commandline/af_unix-comes-to-windows/).</span></span>

## <a name="client-configuration"></a><span data-ttu-id="5c0de-117">客户端配置</span><span class="sxs-lookup"><span data-stu-id="5c0de-117">Client configuration</span></span>

<span data-ttu-id="5c0de-118">`GrpcChannel` 支持通过自定义传输进行 gRPC 调用。</span><span class="sxs-lookup"><span data-stu-id="5c0de-118">`GrpcChannel` supports making gRPC calls over custom transports.</span></span> <span data-ttu-id="5c0de-119">创建通道后，可以使用包含自定义 `ConnectCallback` 的 `SocketsHttpHandler` 来配置它。</span><span class="sxs-lookup"><span data-stu-id="5c0de-119">When a channel is created, it can be configured with a `SocketsHttpHandler` that has a custom `ConnectCallback`.</span></span> <span data-ttu-id="5c0de-120">回调允许客户端通过自定义传输建立连接，然后通过该传输发送 HTTP 请求。</span><span class="sxs-lookup"><span data-stu-id="5c0de-120">The callback allows the client to make connections over custom transports and then send HTTP requests over that transport.</span></span>

> [!IMPORTANT]
> <span data-ttu-id="5c0de-121">`SocketsHttpHandler.ConnectCallback` 是 .NET 5 候选发布 2 中的新 API。</span><span class="sxs-lookup"><span data-stu-id="5c0de-121">`SocketsHttpHandler.ConnectCallback` is a new API in .NET 5 release candidate 2.</span></span>

<span data-ttu-id="5c0de-122">Unix 域套接字连接工厂示例：</span><span class="sxs-lookup"><span data-stu-id="5c0de-122">Unix domain sockets connection factory example:</span></span>

```csharp
public class UnixDomainSocketConnectionFactory
{
    private readonly EndPoint _endPoint;

    public UnixDomainSocketConnectionFactory(EndPoint endPoint)
    {
        _endPoint = endPoint;
    }

    public async ValueTask<Stream> ConnectAsync(SocketsHttpConnectionContext _,
        CancellationToken cancellationToken = default)
    {
        var socket = new Socket(AddressFamily.Unix, SocketType.Stream, ProtocolType.Unspecified);

        try
        {
            await socket.ConnectAsync(_endPoint, cancellationToken).ConfigureAwait(false);
            return new NetworkStream(socket, true);
        }
        catch
        {
            socket.Dispose();
            throw;
        }
    }
}
```

<span data-ttu-id="5c0de-123">使用自定义连接工厂创建通道：</span><span class="sxs-lookup"><span data-stu-id="5c0de-123">Using the custom connection factory to create a channel:</span></span>

```csharp
public static readonly string SocketPath = Path.Combine(Path.GetTempPath(), "socket.tmp");

public static GrpcChannel CreateChannel()
{
    var udsEndPoint = new UnixDomainSocketEndPoint(SocketPath);
    var connectionFactory = new UnixDomainSocketConnectionFactory(udsEndPoint);
    var socketsHttpHandler = new SocketsHttpHandler
    {
        ConnectCallback = connectionFactory.ConnectAsync
    };

    return GrpcChannel.ForAddress("http://localhost", new GrpcChannelOptions
    {
        HttpHandler = socketsHttpHandler
    });
}
```

<span data-ttu-id="5c0de-124">使用上述代码创建的通道通过 Unix 域套接字发送 gRPC 调用。</span><span class="sxs-lookup"><span data-stu-id="5c0de-124">Channels created using the preceding code send gRPC calls over Unix domain sockets.</span></span> <span data-ttu-id="5c0de-125">可以使用 Kesttrel 和 `SocketsHttpHandler` 中的扩展性实现对其他 IPC 技术的支持。</span><span class="sxs-lookup"><span data-stu-id="5c0de-125">Support for other IPC technologies can be implemented using the extensibility in Kestrel and `SocketsHttpHandler`.</span></span>