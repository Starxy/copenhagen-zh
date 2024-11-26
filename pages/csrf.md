---
title: "Cross-site request forgery (CSRF)"
---

# 跨站请求伪造 (CSRF)

## 目录

- [跨站请求伪造 (CSRF)](#跨站请求伪造-csrf)
	- [目录](#目录)
	- [概述](#概述)
		- [跨站与跨域](#跨站与跨域)
	- [防护措施](#防护措施)
		- [反CSRF令牌](#反csrf令牌)
		- [签名双提交Cookie](#签名双提交cookie)
			- [传统双提交Cookie](#传统双提交cookie)
		- [Origin头](#origin头)
	- [SameSite Cookie属性](#samesite-cookie属性)

## 概述

CSRF攻击允许攻击者在用户的凭证存储在Cookie中时，代表用户发起已认证的请求。

当客户端发起跨域请求时，浏览器会发送预检请求以检查请求是否被允许（CORS）。但对于一些“简单”请求，例如表单提交，这一步被省略。由于即使对于跨域请求也自动包含Cookie，这使恶意攻击者能在不直接窃取任何域的令牌的情况下，作为认证用户发起请求。[同源策略](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy)默认禁止跨域客户端读取响应，但请求仍会通过。

例如，如果您登录了`bank.com`，即使表单托管在不同的域上，您的会话Cookie仍会随着此表单提交发送。

```html
<form action="https://bank.com/send-money" method="post">
  <input name="recipient" value="attacker" />
  <input name="value" value="$100" />
  <button>Send money</button>
</form>
```

这也可以通过`fetch()`请求实现，因此不需要用户输入。

```ts
const body = new URLSearchParams();
body.set("recipient", "attacker");
body.set("value", "$100");

await fetch("https://bank.com/send-money", {
  method: "POST",
  body
});
```

### 跨站与跨域

当请求在两个完全不同的域之间时，被认为是跨站和跨域请求，而在两个子域之间则仅被视为跨域请求。虽然跨站请求伪造（CSRF）暗示跨站请求，但默认应严格对待，也要防范跨域攻击。

## 防护措施

可以通过仅接受来自可信来源的浏览器发起的POST及类似POST的请求来防止CSRF。

所有处理表单的路由都必须实施保护措施。如果您的应用程序当前不使用表单，也至少应检查[Origin头](#origin头)以防止未来的问题。通常建议只使用POST及类似的请求方法（如PUT, DELETE等）来修改资源。

对于常见的基于令牌的方法，令牌不应为单次使用的（例如每次表单提交新的令牌），因为这会在按下返回按钮时导致问题。同时，页面应有严格的跨域资源共享（CORS）策略。如果`Access-Control-Allow-Credentials`不严格，恶意站点可以发送GET请求获取具有有效CSRF令牌的HTML表单。

### 反CSRF令牌

这是一个非常简单的方法，每个会话都有一个唯一的CSRF [令牌](/server-side-tokens) 关联。

```html
<form method="post">
  <input name="message" />
  <input type="hidden" name="__csrf" value="<CSRF_TOKEN>" />
  <button>Submit</button>
</form>
```

### 签名双提交Cookie

如果无法在服务器上存储令牌，可以使用签名双提交Cookie。这不同于基本的双提交Cookie，因为表单中包含的令牌使用密钥签名。

使用HMAC SHA-256和密钥生成新的[令牌](/server-side-tokens)并对其进行哈希处理。

```go
func generateCSRFToken() (string, []byte) {
	buffer := [10]byte{}
	crypto.rand.Read(buffer)
	csrfToken := base64.StdEncoding.encodeToString(buffer)
	mac := hmac.New(sha256.New, secret)
	mac.Write([]byte(csrfToken))
	csrfTokenHMAC := mac.Sum(nil)
	return csrfToken, csrfTokenHMAC
}

// 可选择将Cookie与特定会话ID关联。
func generateCSRFToken(sessionId string) (string, []byte) {
	// ...
	mac.Write([]byte(csrfToken + "." + sessionId))
	csrfTokenHMAC := mac.Sum(nil)
	return csrfToken, csrfTokenHMAC
}
```

令牌存储为Cookie，HMAC存储在表单中。Cookie应具有`Secure`、`HttpOnly`和`SameSite`标志。要验证请求，可以使用Cookie验证表单数据中发送的签名。

#### 传统双提交Cookie

常规双提交Cookie如果没有签名，在攻击者访问应用程序域的子域时仍可能导致漏洞。这将允许他们设置自己的双提交Cookie。

### Origin头

防止CSRF攻击的一个简单方法是检查非GET请求的`Origin`头。这是一个新引入的头，包含请求的[来源](https://developer.mozilla.org/en-US/docs/Glossary/Origin)。如果依赖该头，重要的是应用程序不使用GET请求来修改资源。

虽然`Origin`头可以通过自定义客户端伪造，但关键是不能通过客户端JavaScript伪造。用户仅在使用浏览器时容易受到CSRF攻击。

```go
func handleRequest(w http.ResponseWriter, request *http.Request) {
    if request.Method != "GET" {
        originHeader := request.Header.Get("Origin")
        // 还可以将其与Host或X-Forwarded-Host头进行比较。
        if originHeader != "https://example.com" {
            // 请求来源无效
            w.WriteHeader(403)
            return
        }
    }
    // ...
}
```

大约自2020年以来，所有现代浏览器都支持`Origin`头，尽管Chrome和Safari早在之前就支持它。如果未包含`Origin`头，不允许请求。

`Referer`头是`Origin`头之前引入的类似头。当`Origin`头未定义时可作为回退。

## SameSite Cookie属性

会话Cookie应具有`SameSite`标志。此标志决定浏览器在何时在请求中包含Cookie。`SameSite=Lax`的Cookie仅在使用[安全HTTP方法](https://developer.mozilla.org/en-US/docs/Glossary/Safe/HTTP)（如GET）的跨站请求中发送，而`SameSite=Strict`的Cookie不会在任何跨站请求中发送。建议默认使用`Lax`，因为`Strict`的Cookie不会在用户通过外部链接访问您网站时发送。

如果设置为`Lax`，应用程序不应使用GET请求来修改资源。`SameSite`标志的浏览器支持显示它目前对96%的网络用户可用。需要注意的是，该标志仅保护*跨站*请求伪造（不保护*跨域*请求伪造），一般不应作为唯一的防御措施。
