---
uid: fundamentals/servers/yarp/http-client-config
title: YARP HTTP Client Configuration
description: YARP HTTP Client Configuration
author: wadepickett
ms.author: wpickett
ms.date: 2/6/2025
ms.topic: article
content_well_notification: AI-contribution
ai-usage: ai-assisted
---

# YARP HTTP Client Configuration

## Introduction

Each [Cluster](xref:Yarp.ReverseProxy.Configuration.ClusterConfig) has a dedicated [HttpMessageInvoker](/dotnet/api/system.net.http.httpmessageinvoker) instance used to forward requests to its [Destination](xref:Yarp.ReverseProxy.Configuration.DestinationConfig)s. The configuration is defined per cluster. On YARP startup, all clusters get new `HttpMessageInvoker` instances, however if later the cluster configuration gets changed the [IForwarderHttpClientFactory](xref:Yarp.ReverseProxy.Forwarder.IForwarderHttpClientFactory) will re-run and decide if it should create a new `HttpMessageInvoker` or keep using the existing one. The default `IForwarderHttpClientFactory` implementation creates a new `HttpMessageInvoker` when there are changes to the [HttpClientConfig](xref:Yarp.ReverseProxy.Configuration.HttpClientConfig).

Properties of outgoing requests for a given cluster can be configured as well. They are defined in [ForwarderRequestConfig](xref:Yarp.ReverseProxy.Forwarder.ForwarderRequestConfig).

The configuration is represented differently if you're using the [IConfiguration](/dotnet/api/microsoft.extensions.configuration.iconfiguration) model or the code-first model.

## IConfiguration
These types are focused on defining serializable configuration. The code based configuration model is described below in the "Code Configuration" section.

### HttpClient
HTTP client configuration is based on [HttpClientConfig](xref:Yarp.ReverseProxy.Configuration.HttpClientConfig) and represented by the following configuration schema. If you need a more granular approach, use a <!--check link--> [custom implementation](xref:fundamentals/servers/yarp/http-client-config#custom-iforwarderhttpclientfactory) of `IForwarderHttpClientFactory`.

<!-- original linke [custom implementation](https://microsoft.github.io/reverse-proxy/articles/http-client-config.html#custom-iforwarderhttpclientfactory) -->
```json
"HttpClient": {
  "SslProtocols": [ "<protocol-names>" ],
  "MaxConnectionsPerServer": "<int>",
  "DangerousAcceptAnyServerCertificate": "<bool>",
  "RequestHeaderEncoding": "<encoding-name>",
  "ResponseHeaderEncoding": "<encoding-name>",
  "EnableMultipleHttp2Connections": "<bool>",
  "WebProxy": {
    "Address": "<url>",
    "BypassOnLocal": "<bool>",
    "UseDefaultCredentials": "<bool>"
  }
}
```

Configuration settings:

* SslProtocols - [SSL protocols](/dotnet/api/system.security.authentication.sslprotocols) enabled on the given HTTP client. Protocol names are specified as array of strings. Default value is [None](/dotnet/api/system.security.authentication.sslprotocols#System_Security_Authentication_SslProtocols_None).
```json
"SslProtocols": [
  "Tls11",
  "Tls12"
]
```
* MaxConnectionsPerServer - maximal number of HTTP 1.1 connections open concurrently to the same server. Default value is [int32.MaxValue](/dotnet/api/system.int32.maxvalue).
```json
"MaxConnectionsPerServer": "10"
```
* DangerousAcceptAnyServerCertificate - indicates whether the server's SSL certificate validity is checked by the client. Setting it to `true` completely disables validation. Default value is `false`.
```json
"DangerousAcceptAnyServerCertificate": "true"
```
* RequestHeaderEncoding - enables other than ASCII encoding for outgoing request headers. Setting this value will leverage [`SocketsHttpHandler.RequestHeaderEncodingSelector`](/dotnet/api/system.net.http.socketshttphandler.requestheaderencodingselector) and use the selected encoding for all headers. The value is then parsed by [`Encoding.GetEncoding`](/dotnet/api/system.text.encoding.getencoding#System_Text_Encoding_GetEncoding_System_String_), use values like: "utf-8", "iso-8859-1", etc.
```json
"RequestHeaderEncoding": "utf-8"
```
* ResponseHeaderEncoding - enables other than ASCII encoding for incoming response headers (from requests that the proxy would forward out). Setting this value will leverage [`SocketsHttpHandler.ResponseHeaderEncodingSelector`](/dotnet/api/system.net.http.socketshttphandler.responseheaderencodingselector) and use the selected encoding for all headers. The value is then parsed by [`Encoding.GetEncoding`](/dotnet/api/system.text.encoding.getencoding#System_Text_Encoding_GetEncoding_System_String_), use values like: "utf-8", "iso-8859-1", etc.
```json
"ResponseHeaderEncoding": "utf-8"
```
Note that if you're using an encoding other than ASCII, you also need to set your server to accept requests and/or send responses with such headers. For example, when using Kestrel as the server, use [`KestrelServerOptions.RequestHeaderEncodingSelector`](/dotnet/api/Microsoft.AspNetCore.Server.Kestrel.Core.KestrelServerOptions.RequestHeaderEncodingSelector) / [`.ResponseHeaderEncodingSelector`](/dotnet/api/Microsoft.AspNetCore.Server.Kestrel.Core.KestrelServerOptions.ResponseHeaderEncodingSelector) to configure Kestrel to allow `Latin1` ("`iso-8859-1`") headers:
```csharp
var builder = WebApplication.CreateBuilder(args);
builder.WebHost.ConfigureKestrel(kestrel =>
{
    kestrel.RequestHeaderEncodingSelector = _ => Encoding.Latin1;
    // and/or
    kestrel.ResponseHeaderEncodingSelector = _ => Encoding.Latin1;
});
```
* EnableMultipleHttp2Connections - enables opening additional HTTP/2 connections to the same server when the maximum number of concurrent streams is reached on all existing connections. The default is `true`. See [SocketsHttpHandler.EnableMultipleHttp2Connections](/dotnet/api/system.net.http.socketshttphandler.enablemultiplehttp2connections)
```json
"EnableMultipleHttp2Connections": false
```
* WebProxy - Enables sending requests through an outbound HTTP proxy to reach the destinations. See [`SocketsHttpHandler.Proxy`](/dotnet/api/system.net.http.socketshttphandler.proxy) for details.
  - Address - The url address of the outbound proxy.
  - BypassOnLocal - A bool indicating if requests to local addresses should bypass the outbound proxy.
  - UseDefaultCredentials - A bool indicating if the current application credentials should be use to authenticate to the outbound proxy. ASP.NET Core does not impersonate authenticated users for outbound requests.
```json
"WebProxy": {
  "Address": "http://myproxy:8080",
  "BypassOnLocal": "true",
  "UseDefaultCredentials": "false"
}
```

### HttpRequest
HTTP request configuration is based on [ForwarderRequestConfig](xref:Yarp.ReverseProxy.Forwarder.ForwarderRequestConfig) and represented by the following configuration schema.
```json
"HttpRequest": {
  "ActivityTimeout": "<timespan>",
  "Version": "<string>",
  "VersionPolicy": ["RequestVersionOrLower", "RequestVersionOrHigher", "RequestVersionExact"],
  "AllowResponseBuffering": "<bool>"
}
```

Configuration settings:

* ActivityTimeout - how long a request is allowed to remain idle between any operation completing, after which it will be canceled. The default is 100 seconds. The timeout will reset when response headers are received or after successfully reading or writing any request, response, or streaming data like gRPC or WebSockets. TCP keep-alives and HTTP/2 protocol pings will not reset the timeout, but WebSocket pings will.
* Version - outgoing request [version](/dotnet/api/system.net.http.httprequestmessage.version). The supported values at the moment are `1.0`, `1.1`, `2` and `3`. Default value is 2.
* VersionPolicy - defines how the final version is selected for the outgoing requests. See [HttpRequestMessage.VersionPolicy](/dotnet/api/system.net.http.httprequestmessage.versionpolicy). The default value is `RequestVersionOrLower`.
* AllowResponseBuffering - allows to use write buffering when sending a response back to the client, if the server hosting YARP (e.g. IIS) supports it. **NOTE**: enabling it can break SSE (server side event) scenarios.


## Configuration example
The below example shows 2 samples of HTTP client and request configurations for `cluster1` and `cluster2`.

```json
{
  "Clusters": {
    "cluster1": {
      "LoadBalancingPolicy": "Random",
      "HttpClient": {
        "SslProtocols": [
          "Tls11",
          "Tls12"
        ],
        "MaxConnectionsPerServer": "10",
        "DangerousAcceptAnyServerCertificate": "true"
      },
      "HttpRequest": {
        "ActivityTimeout": "00:00:30"
      },
      "Destinations": {
        "cluster1/destination1": {
          "Address": "https://localhost:10000/"
        },
        "cluster1/destination2": {
          "Address": "http://localhost:10010/"
        }
      }
    },
    "cluster2": {
      "HttpClient": {
        "SslProtocols": [
          "Tls12"
        ]
      },
      "HttpRequest": {
        "Version": "1.1",
        "VersionPolicy": "RequestVersionExact"
      },
      "Destinations": {
        "cluster2/destination1": {
          "Address": "https://localhost:10001/"
        }
      }
    }
  }
}
```

## Code Configuration
HTTP client configuration uses the type [HttpClientConfig](xref:Yarp.ReverseProxy.Configuration.HttpClientConfig).

The following is an example of `HttpClientConfig` using [code based](xref:fundamentals/servers/yarp/config-providers) configuration. An instance of `HttpClientConfig` is assigned to the [ClusterConfig.HttpClient](xref:Yarp.ReverseProxy.Configuration.ClusterConfig) property before passing the cluster array to `LoadFromMemory` method.

```csharp
var routes = new[]
{
    new RouteConfig()
    {
        RouteId = "route1",
        ClusterId = "cluster1",
        Match =
        {
            Path = "{**catch-all}"
        }
    }
};
var clusters = new[]
{
    new ClusterConfig()
    {
        ClusterId = "cluster1",
        Destinations =
        {
            { "destination1", new DestinationConfig() { Address = "https://localhost:10000" } }
        },
        HttpClient = new HttpClientConfig { MaxConnectionsPerServer = 10, SslProtocols = SslProtocols.Tls11 | SslProtocols.Tls12 }
    }
};

services.AddReverseProxy()
    .LoadFromMemory(routes, clusters);
```

## Configuring the http client

`ConfigureHttpClient` provides a callback to customize the `SocketsHttpHandler` settings used for proxying requests. This will be called each time a cluster is added or changed. Cluster settings are applied to the handler before the callback. Custom data can be provided in the cluster metadata. This example shows adding a client certificate that will authenticate the proxy to the destination servers.

```csharp
var clientCert = new X509Certificate2("path");
services.AddReverseProxy()
    .ConfigureHttpClient((context, handler) =>
    {
        handler.SslOptions.ClientCertificates.Add(clientCert);
    })
```

## Custom IForwarderHttpClientFactory
If direct control of HTTP client construction is necessary, the default [IForwarderHttpClientFactory](xref:Yarp.ReverseProxy.Forwarder.IForwarderHttpClientFactory) can be replaced with a custom one. For some customizations you can derive from the default [ForwarderHttpClientFactory](xref:Yarp.ReverseProxy.Forwarder.ForwarderHttpClientFactory) and override the methods that configure the client.

It's recommended that any custom factory set the following `SocketsHttpHandler` properties to the same values as the default factory does in order to preserve a correct reverse proxy behavior and avoid unnecessary overhead.

```csharp
new SocketsHttpHandler
{
    UseProxy = false,
    AllowAutoRedirect = false,
    AutomaticDecompression = DecompressionMethods.None,
    UseCookies = false,
    EnableMultipleHttp2Connections = true,
    ActivityHeadersPropagator = new ReverseProxyPropagator(DistributedContextPropagator.Current),
    ConnectTimeout = TimeSpan.FromSeconds(15),
};
```

Always return an HttpMessageInvoker instance rather than an HttpClient instance which derives from HttpMessageInvoker. HttpClient buffers responses by default which breaks streaming scenarios and increases memory usage and latency.

Custom data can be provided in the cluster metadata.

The below is an example of a custom `IForwarderHttpClientFactory` implementation.

```csharp
public class CustomForwarderHttpClientFactory : IForwarderHttpClientFactory
{
    public HttpMessageInvoker CreateClient(ForwarderHttpClientContext context)
    {
        var handler = new SocketsHttpHandler
        {
            UseProxy = false,
            AllowAutoRedirect = false,
            AutomaticDecompression = DecompressionMethods.None,
            UseCookies = false,
            EnableMultipleHttp2Connections = true,
            ActivityHeadersPropagator = new ReverseProxyPropagator(DistributedContextPropagator.Current),
            ConnectTimeout = TimeSpan.FromSeconds(15),
        };

        return new HttpMessageInvoker(handler, disposeHandler: true);
    }
}
```
