---
title: "OAuth"
---

# OAuth认证

## 目录

- [OAuth认证](#oauth认证)
  - [目录](#目录)
  - [概述](#概述)
  - [创建授权 URL](#创建授权-url)
  - [验证授权码](#验证授权码)
  - [用于代码交换的证明密钥 (PKCE)](#用于代码交换的证明密钥-pkce)
  - [OpenID Connect (OIDC)](#openid-connect-oidc)
    - [OpenID Connect 发现](#openid-connect-发现)
  - [账户关联](#账户关联)
  - [其他注意事项](#其他注意事项)

## 概述

OAuth 是一种广泛使用的授权协议，支持“使用 Google 登录”和“使用 GitHub 登录”等功能。它允许用户在不共享凭据的情况下，将对外部服务（如 Google）的资源访问权限授予您的应用程序。通过 OAuth，我们可以让第三方服务处理身份验证，然后获取用户的资料来创建用户和会话。

在基本的 OAuth 流程中，用户被重定向到第三方服务，该服务对用户进行身份验证，然后用户被重定向回您的应用程序。此时，您可以获得一个访问令牌，用于代表用户请求资源。

您的应用程序需要两个服务器端点：

1. 登录端点 (GET)：将用户重定向到 OAuth 提供商。
2. 回调端点 (GET)：处理来自 OAuth 提供商的重定向。

OAuth 有多个版本，OAuth 2.0 是最新的版本。本页仅涵盖 OAuth 2.0，特别是授权码授权类型，标准化在 [RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749) 中。隐式授权类型已被弃用，不应使用。

## 创建授权 URL

以 GitHub 为例，第一步是创建一个 GET 端点（登录端点），将用户重定向到 GitHub。重定向位置是带有一些参数的授权 URL。

```
https://github.com/login/oauth/authorize?
response_type=code
&client_id=<CLIENT_ID>
&redirect_uri=<CALLBACK_ENDPOINT>
&state=<STATE>
```

`state` 用于确保发起流程的用户和被重定向回来的用户是同一个人。因此，每次请求都必须生成新的 `state`。虽然规范没有严格要求，但强烈建议这样做，且可能根据提供商的要求是必需的。它应使用加密安全的随机生成器生成，并具有至少 112 位的熵。`state` 也可以用于从登录端点传递数据到回调端点，不过也可以使用 cookie。

服务器必须跟踪与每次尝试相关联的 `state`。一种简单的方法是将其存储为具有 `HttpOnly`、`SameSite=Lax`、`Secure` 和 `Path=/` 属性的 cookie。您也可以将 `state` 分配给当前会话。

可以定义一个 `scope` 参数来请求对其他资源的访问。如果有多个范围，它们应以空格分隔。

```
&scope=email%20identity
```

您可以通过添加一个指向登录端点的链接来创建一个“登录”按钮。

```html
<a href="/login/github">使用 GitHub 登录</a>
```

## 验证授权码

用户将被重定向到回调端点（在 `redirect_uri` 中定义），并附带一个一次性授权码，该码包含在查询参数中。然后将此代码交换为访问令牌。

```
https://example.com/login/github/callback?code=<CODE>&state=<STATE>
```

如果您在授权 URL 中添加了 `state`，重定向请求将包含一个 `state` 参数。检查其是否与尝试相关联的 `state` 匹配至关重要。如果缺少 `state` 或不匹配，则返回错误。常见错误是忘记检查 URL 中是否存在 `state` 参数。

代码通过 `application/x-www-form-urlencoded` POST 请求发送到 OAuth 提供商的令牌端点。

```
POST https://github.com/login/oauth/access_token
Accept: application/json
Authorization: Basic <CREDENTIALS>
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&client_id=<CLIENT_ID>
&redirect_uri=<CALLBACK_ENDPOINT>
&code=<CODE>
```

如果您的 OAuth 提供商使用客户端密钥，则应将其与客户端 ID 一起在 Authorization 头中进行 base64 编码（HTTP 基本授权方案）。

```go
var clientId, clientSecret string
credentials := base64.StdEncoding.EncodeToString([]byte(clientId + ":" + clientSecret))
```

一些提供商也允许在请求体中包含客户端密钥。

```
POST https://github.com/login/oauth/access_token
Accept: application/json
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&client_id=<CLIENT_ID>
&client_secret=<CLIENT_SECRET>
&redirect_uri=<CALLBACK_ENDPOINT>
&code=<CODE>
```

请求将返回一个访问令牌，然后可以用来获取用户的身份。它可能还包括其他字段，如 `refresh_token` 和 `expires_in`。

```
{ "access_token": "<ACCESS_TOKEN>" }
```

例如，使用访问令牌，您可以获取其 GitHub 个人资料并存储其 GitHub 用户 ID，这将允许您在他们再次登录时获取其注册帐户。请注意，OAuth 提供商提供的电子邮件地址可能未经验证。您可能需要手动验证用户的电子邮件或阻止没有经过验证电子邮件的用户。

访问令牌本身不应作为会话的替代品。

## 用于代码交换的证明密钥 (PKCE)

PKCE 在 [RFC 7636](https://datatracker.ietf.org/doc/html/rfc7636) 中引入，为 OAuth 2.0 提供了额外的保护。我们建议在支持的情况下使用 PKCE 以及 `state` 和客户端密钥。注意，一些 OAuth 提供商在启用 PKCE 时不需要客户端密钥，在这种情况下不应使用 PKCE。

PKCE 可以完全替代 `state`，因为两者都可以防御 CSRF 攻击，但它可能是您的 OAuth 提供商所要求的。

每次请求必须生成一个新的代码验证器。它应使用加密安全的随机生成器生成，并具有至少 112 位的熵（RFC 建议 256 位）。与 `state` 类似，您的应用程序必须跟踪与每次尝试相关联的代码验证器（使用 cookies 或会话）。在授权 URL 中包含一个称为代码挑战的 base64url（无填充）编码的 SHA256 哈希。

```go
var codeVerifier string
codeChallengeBuf := sha256.Sum256([]byte(codeVerifier))
codeChallenge := base64.URLEncoding.WithPadding(base64.NoPadding).EncodeToString(codeChallengeBuf)
```

```
https://accounts.google.com/o/oauth2/v2/auth?
response_type=code
&client_id=<...>
&redirect_uri=<...>
&state=<...>
&code_challenge_method=S256
&code_challenge=<CODE_CHALLENGE>
```

在回调端点，应将当前尝试的代码验证器与授权码一起发送。

```
POST https://oauth2.googleapis.com/token
Accept: application/json
Authorization: Basic <CREDENTIALS>
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&client_id=<...>
&redirect_uri=<...>
&code=<...>
&code_verifier=<CODE_VERIFIER>
```

## OpenID Connect (OIDC)

[OpenID Connect](https://openid.net/specs/openid-connect-core-1_0.html) 是构建在 OAuth 2.0 之上的广泛使用的协议。OAuth 的一个重要补充是身份提供商返回一个 ID 令牌以及访问令牌。ID 令牌是一个包含用户数据的 [JSON Web Token](https://datatracker.ietf.org/doc/html/rfc7519)。它将始终在 `sub` 字段中包含用户的唯一标识符。

```
{
	"access_token": "<ACCESS_TOKEN>",
	"id_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwiaXNzIjoiZXhhbXBsZS5jb20ifQ.uMMQPfp7LwcLiBbfZdoHdIPjKgS2HUfOr5vlY71el8A"
}
```

虽然可以使用公钥验证令牌，但如果您使用 HTTPS 进行通信，对于服务器端应用程序来说，这不是严格必要的。

### OpenID Connect 发现

OpenID Connect 定义了一种[发现机制](https://openid.net/specs/openid-connect-discovery-1_0.html)，允许客户端动态获取 OpenID 提供商的配置，包括 OAuth 2.0 端点位置。这消除了在应用程序中硬编码端点 URL 的需要。要使用 OpenID Connect 发现，您的 OpenID 提供商必须有一个可用的发现端点。

发现端点是一个众所周知的 URL，返回一个包含 OpenID 提供商配置信息的 JSON 文档。注意，并非所有 OAuth 提供商都支持 OpenID Connect 发现。检查您的提供商文档以确定他们是否提供发现端点。如果没有，您可能仍需要在应用程序中手动配置端点 URL。

众所周知的 URL 的路径为 `/.well-known/openid-configuration`。例如，Google 的发现端点如下所示：

```
https://accounts.google.com/.well-known/openid-configuration
```

端点将返回一个 JSON 对象，包含 OpenID 提供商的配置，包括授权、令牌交换和用户信息检索的端点 URL。

```json
{
  "issuer": "https://example.com",
  "authorization_endpoint": "https://example.com/oauth2/authorize",
  "token_endpoint": "https://example.com/oauth2/token",
  "userinfo_endpoint": "https://example.com/oauth2/userinfo",
  "code_challenge_methods_supported": ["S256"],
  "grant_types_supported": ["authorization_code", "refresh_token"],
  "scopes_supported": ["openid", "email", "profile"]
}
```

通过 OpenID Connect 发现，您的应用程序可以动态适应 OpenID 提供商配置的变化而无需代码更新。这确保了您的应用程序始终使用最新的端点 URL。缺点是您将不得不进行额外的获取请求。

## 账户关联

账户关联允许用户使用任何社交账户登录，并在您的应用程序上被认证为同一用户。通常通过检查提供商注册的电子邮件地址来实现。如果您使用电子邮件来关联账户，请确保验证用户的电子邮件。大多数提供商在用户资料中提供 `is_verified` 字段或类似字段。除非提供商在其文档中明确提到，否则不要假设电子邮件已被验证。没有经过验证的电子邮件的用户应被阻止完成身份验证流程，并提示他们先验证电子邮件。

## 其他注意事项

- [开放重定向](/open-redirect)。
