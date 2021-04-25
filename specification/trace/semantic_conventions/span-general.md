# 通用属性

本节中描述的属性是通用的，而不是特定于特定操作的。 这些属性可以在它们应用的任何Span上使用。 某些操作可能引用或需要其中一些属性。

<!-- Re-generate TOC with `markdown-toc --no-first-h1 -i` -->

<!-- toc -->

- [通用网络连接属性](#通用网络连接属性)
  * [`net.transport`](#net.transport)
  * [`net.*.name`](#net.name)
- [通用远程服务属性](#通用远程服务属性)
- [通用标识符属性](#通用标识符属性)

<!-- tocstop -->

## 通用网络连接属性

这些属性可用于与网络相关的操作。
 `net.peer.*` 属性描述了网络连接的远程端（通常是传输层对等体，例如建立TCP连接的节点）的属性，而 `net.host.*` 属性则描述了本地端。在理想情况下，不考虑代理或多个IP地址或主机名的情况下，客户端 `net.peer.*` 属性将等于服务器的 `net.host.*`属性，反之亦然

|  属性名   |                                 说明和示例                                |
| :--------------- | :-------------------------------------------------------------------------------- |
| `net.transport` | 使用的传输协议. [请参阅下面的注释](#net.transport).                         |
| `net.peer.ip`   | 远程端点的地址 (用IPv4或IPv6的[RFC5952][]，以`.`分隔)       |
| `net.peer.port` | 远程端点的端口. 例如, `80`.                                      |
| `net.peer.name` | 远程主机名或类似名称, [请参阅下面的注释](#net.name).                           |
| `net.host.ip`   | 类似 `net.peer.ip` 不过是用于指定主机的IP。 这对于具有多个IP的主机很有用.       |
| `net.host.port` | 类似 `net.peer.port` 不过是用于指定主机的端口。                                        |
| `net.host.name` | 本地主机名或类似名称, [请参阅下面的注释](#net.name).                            |

[RFC5952]: https://tools.ietf.org/html/rfc5952

<a name="net.transport"></a>

### `net.transport` 属性

该属性应该设置为传输层协议（或“应用协议”下的相关协议）的名称。 您必须使用以下字符串之一。

* `IP.TCP`
* `IP.UDP`
* `IP`: 其他基于IP的协议
* `Unix`: Unix域套接字。 见下文。
* `pipe`: 命名管道或匿名管道。 见下文。
* `inproc`: 不要使用通常需要网络属性的“真实”网络协议。 对于仅使用进程内通信的信号。 通常，在这种情况下，可以排除所有其他网络属性。
* `other`: 其他（基于IP的除外）

使用Unix和pipe时，连接通常是通过文件系统，而不是直接连接到已知对等体，因此`net.peer.name`通常是唯一的属性（请参阅下面的 `net.peer.name`说明）。

<a name="net.name"></a>

### `net.*.name` 属性

对于基于IP的通信，名称应为DNS主机名。
如果是 `net.peer.name`, 则应为用于查找要连接的IP地址的名称。（也就是说，如果设置了`net.peer.ip`，它就会匹配。例如，连接到URL`https://example.com/foo`时会出现`"example.com"`）。
如果只有一个IP地址而没有主机名，则可以选择使用反向IP查找来获取它。`net.host.name` 应该是本地主机的主机名，最好是用于当前操作的对等连接的主机名。如果您不知道主机名，则公用主机名优先于专用主机名。但是，在那种情况下，已经包含在资源中的信息可能会被复制并可能被留下。通常，使用反向查询获取`net.host.name`是没有意义的。这是静态信息，应另存为资源信息。

如果`net.transport`是`"unix"`或`"pipe"`，然后绝对到文件表示它应该是`net.peer.name`（在这种情况下`net.host.name`没有意义）。
如果没有这样的文件存在（例如匿名pipe），名称必须被明确设置为空字符串，从一个未知名字或不覆盖instrumentation的情况中区分开来。

## 通用远程服务属性

该属性被应用为所有远程访问相关的操作。用户可以基于他们自己的分布式系统服务的特定的语义来定义服务的名称。Instrumentations 需提供一种方式给用户去配置这个名称。

|  属性名 |                                 说明和示例                                |
| :-------------- | :-------------------------------------------------------------------------------- |
| `peer.service`  | 远程服务的名称[`service.name`](../../resource/semantic_conventions/README.md#service). 应该等于远程服务的实际资源的`service.name`属性(如果有的话)。 |

用户可以指定`peer.service`属性的示例:
- 一个Redis缓存授权tokens服务 `peer.service="AuthTokenCache"`。
- 一个gRPC服务 `rpc.service="io.opentelemetry.AuthService"` 可以托管在网关 `peer.service="ExternalApiService"` 和一个后端服务 `peer.service="AuthService"`中。

## 通用唯一标识属性

这些属性可以用于与经过身份验证和/或授权的终端用户进行的任何操作。

|  属性名 |                                 说明和示例                                |
| :-------------- | :-------------------------------------------------------------------------------- |
| `enduser.id`    | 来自系统外部的入站请求的访问令牌。 或从[Authorization]标头中提取的用户名或client_id。  |
| `enduser.role`  | 客户端从令牌或应用程序安全上下文中提取请求的实际/预期角色。 |
| `enduser.scope` | 从令牌或应用程序安全上下文中提取的客户端当前拥有的范围或授予的特权。 该值是从 [OAuth 2.0 Access Token] 关联的范围或者 [SAML 2.0 Assertion] 声明的属性中获取的。 |

这些属性表示运行用户代理的已认证用户，该用户代理向已检测系统发出请求。 希望使用“关联上下文”机制在系统中的节点之间不更改地传播此信息。 这些属性不应用于记录系统之间的身份验证属性。

下面是提取 `enduser.id` 属性值的示例:

| 认证协议 | 字段或描述            |
| :---------------------- | :------------------------------ |
| [HTTP Basic/Digest Authentication] | `username`               |
| [OAuth 2.0 Bearer Token] | 如果是[OAuth 2.0 Client Credentials Grant]认证流程，请从`client_id`中获取[OAuth 2.0 Client Identifier], 如果是使用其他不透明令牌的流程，请从响应中获取主题或用户名，该响应会获得令牌信息。 |
| [OpenID Connect 1.0 IDToken] | `sub` |
| [SAML 2.0 Assertion] | `urn:oasis:names:tc:SAML:2.0:assertion:Subject` |
| [Kerberos] | `PrincipalName` |

| 框架               | 字段或描述            |
| :---------------------- | :------------------------------ |
| [JavaEE/JakartaEE Servlet] | `javax.servlet.http.HttpServletRequest.getUserPrincipal()` |
| [Windows Communication Foundation] | `ServiceSecurityContext.Current.PrimaryIdentity` |

[Authorization]: https://tools.ietf.org/html/rfc7235#section-4.2
[OAuth 2.0 Access Token]: https://tools.ietf.org/html/rfc6749#section-3.3
[SAML 2.0 Assertion]: http://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0.html
[HTTP Basic/Digest Authentication]: https://tools.ietf.org/html/rfc2617
[OAuth 2.0 Bearer Token]: https://tools.ietf.org/html/rfc6750
[OAuth 2.0 Client Identifier]: https://tools.ietf.org/html/rfc6749#section-2.2
[OAuth 2.0 Client Credentials Grant]: https://tools.ietf.org/html/rfc6749#section-4.4
[OpenID Connect 1.0 IDToken]: https://openid.net/specs/openid-connect-core-1_0.html#IDToken
[Kerberos]: https://tools.ietf.org/html/rfc4120
[JavaEE/JakartaEE Servlet]: https://jakarta.ee/specifications/platform/8/apidocs/javax/servlet/http/HttpServletRequest.html
[Windows Communication Foundation]: https://docs.microsoft.com/en-us/dotnet/api/system.servicemodel.servicesecuritycontext?view=netframework-4.8

鉴于此信息的机密性，SDK和导出程序应默认删除这些属性。 对于需要信息且不违反任何策略或法规的用例，应提供配置参数以启用保留。
